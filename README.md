# nudgebee-compatibility-registry

Static compatibility data published as raw JSON over HTTPS. The files in this
repo are intended to be fetched directly from
`https://raw.githubusercontent.com/nudgebee/nudgebee-compatibility-registry/master/<file>.json`.

The `master` branch is the live release — a merge is the deploy. There are no
tags; if you need to roll back, revert the offending commit.

## Files

### `addon_compatibility.json`

Maps every EKS-managed add-on (including marketplace add-ons) to the list of
add-on versions AWS marks as compatible with each cluster Kubernetes version.

Shape:

```jsonc
{
  "<addon-name>": {
    "<k8s-version>": [
      "<addon-version>",
      "..."
    ]
  }
}
```

Example:

```jsonc
{
  "vpc-cni": {
    "1.34": ["v1.21.1-eksbuild.8", "v1.21.1-eksbuild.7", "..."],
    "1.33": ["v1.21.1-eksbuild.8", "..."]
  },
  "coredns": {
    "1.34": ["v1.13.2-eksbuild.7", "..."]
  }
}
```

Add-on versions within each cluster version are sorted newest-first.

**Source of truth:** `aws eks describe-addon-versions --kubernetes-version <V>`.
**Coverage:** AWS-managed core add-ons (`vpc-cni`, `coredns`, `kube-proxy`,
`aws-ebs-csi-driver`, `metrics-server`, `cert-manager`, etc.) plus every EKS
Marketplace add-on AWS exposes through the same API.

### `api_deprecations.json`

Kubernetes API deprecation / removal registry, keyed by the cluster version in
which the deprecation or removal applies.

Shape:

```jsonc
{
  "<k8s-version>": {
    "deprecated": [
      {
        "kind": "Ingress",
        "group": "extensions",
        "version": "v1beta1",
        "deprecated_version": "1.14",
        "deleted_version": "1.22",
        "replacement_group": "networking.k8s.io",
        "replacement_version": "v1",
        "replacement_kind": "Ingress",
        "description": "Optional human-readable migration note."
      }
    ],
    "deleted": [ /* same shape */ ]
  }
}
```

A version with no removals or deprecations is recorded as
`{"deleted": [], "deprecated": []}` so consumers can distinguish "no data" from
"explicitly nothing to report."

## Refreshing `addon_compatibility.json`

The file is regenerated from AWS EKS itself.

**Prerequisites**

- AWS CLI v2 configured with any IAM principal that can call
  `eks:DescribeAddonVersions` (read-only, account-agnostic — any active
  EKS-enabled account works).
- Python 3.9+.

**Script** (run from the repo root):

```python
import json, re, subprocess
from collections import defaultdict

VERSIONS = [f"1.{m}" for m in range(20, 35)]  # extend as AWS GA's new minors

data = defaultdict(lambda: defaultdict(set))
for v in VERSIONS:
    next_token = None
    while True:
        cmd = ["aws", "eks", "describe-addon-versions",
               "--kubernetes-version", v,
               "--max-results", "100",
               "--region", "us-east-1",
               "--output", "json"]
        if next_token:
            cmd += ["--next-token", next_token]
        resp = json.loads(subprocess.check_output(cmd))
        for a in resp.get("addons", []):
            for av in a.get("addonVersions", []):
                for c in av.get("compatibilities", []):
                    if c.get("clusterVersion") == v:
                        data[a["addonName"]][v].add(av["addonVersion"])
        next_token = resp.get("nextToken")
        if not next_token:
            break

def ver_key(s):
    return tuple(-int(p) for p in re.findall(r"\d+", s))

out = {
    name: {v: sorted(list(versions), key=ver_key)
           for v, versions in sorted(data[name].items(), key=lambda kv: -float(kv[0]))}
    for name in sorted(data)
}

with open("addon_compatibility.json", "w") as f:
    json.dump(out, f, indent=4)
```

The refresh is idempotent — re-running on the same day against the same AWS
data should produce a byte-identical file.

**When to refresh**

- New EKS Kubernetes minor reaches GA (add it to `VERSIONS`).
- A core add-on (CNI, CoreDNS, kube-proxy, EBS CSI, etc.) ships a new version
  that should appear as a recommended upgrade target.
- Periodically (suggested: monthly) to keep data current.

## Refreshing `api_deprecations.json`

This file is curated by hand against the upstream
[Kubernetes deprecation guide](https://kubernetes.io/docs/reference/using-api/deprecation-guide/)
and version-specific deprecation pages (e.g.
`https://v1-34.docs.kubernetes.io/docs/reference/using-api/deprecation-guide/`).

When a new Kubernetes minor is released:

1. Open the version-specific deprecation guide for that minor.
2. Add an entry under the new key in `api_deprecations.json`. Use empty
   `deleted` / `deprecated` arrays if the release removes / deprecates nothing.
3. Keep `deprecated` and `deleted` separate — they are read as distinct lists.

## Contributing

1. Refresh or edit the relevant JSON file.
2. Validate it parses: `python3 -m json.tool <file>`.
3. Open a PR with a one-line note on what changed and why (e.g.,
   "add EKS 1.34 add-ons", "register Ingress v1beta1 removal in 1.22").
