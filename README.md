# flux-loop

A Helm chart for declaratively managing multiple Flux CD objects — HelmRepositories, HelmReleases, Namespaces, and Secrets — from a single values file.

## Overview

`flux-loop` acts as a meta-chart that renders Flux CD CRDs and supporting Kubernetes resources. Instead of writing individual manifests for each Flux object, you define them all in one values file and let this chart render and manage them together.

**Renders the following resource types:**
- `source.toolkit.fluxcd.io/v1` — `HelmRepository`
- `helm.toolkit.fluxcd.io/v2` — `HelmRelease`
- `v1` — `Namespace`
- `v1` — `Secret`

## Prerequisites

- Kubernetes cluster
- [Flux CD](https://fluxcd.io/flux/installation/) installed in the cluster
- Helm 3.x

## Installation

### From GHCR (OCI)

```sh
helm install flux-loop oci://ghcr.io/55octet/flux-loop --version 1.0.0 -f my-values.yaml
```

### From source

```sh
git clone https://github.com/55octet/flux-loop.git
helm install flux-loop ./flux-loop -f my-values.yaml
```

## Values

### Top-level

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `nameOverride` | string | `""` | Override the chart name |
| `fullnameOverride` | string | `""` | Override the full release name |
| `helmRepositories` | list | `[]` | List of HelmRepository objects to create |
| `namespaces` | list | `[]` | List of Namespace objects to create |
| `helmReleases` | map | `{}` | Map of HelmRelease objects to create (key = release name) |
| `secrets` | list | `[]` | List of Secret objects to create |

### `helmRepositories[]`

| Key | Type | Required | Description |
|-----|------|----------|-------------|
| `name` | string | yes | Name of the HelmRepository resource |
| `namespace` | string | yes | Namespace to create the resource in |
| `url` | string | yes | Repository URL (HTTP/S or OCI) |
| `type` | string | no | Repository type (e.g. `oci`). Omit for standard HTTP/S repos |
| `interval` | string | no | Reconciliation interval. Defaults to `10m0s` |
| `annotations` | map | no | Additional annotations |
| `labels` | map | no | Additional labels |

### `namespaces[]`

| Key | Type | Required | Description |
|-----|------|----------|-------------|
| `name` | string | yes | Name of the Namespace |
| `annotations` | map | no | Additional annotations |
| `labels` | map | no | Additional labels |

### `helmReleases`

`helmReleases` is a **map** where each key is the release name. Each entry supports:

| Key | Type | Required | Description |
|-----|------|----------|-------------|
| `namespace` | string | no | Namespace for the HelmRelease |
| `apiVersion` | string | no | Override the apiVersion. Defaults to `helm.toolkit.fluxcd.io/v2` |
| `spec` | map | yes | Full HelmRelease spec (passed through directly). `install` and `upgrade` are extracted and defaulted |
| `spec.install` | map | no | Install remediation config. Defaults to `remediation.retries: 3` |
| `spec.upgrade` | map | no | Upgrade remediation config. Defaults to `remediation.retries: 3` |

### `secrets[]`

| Key | Type | Required | Description |
|-----|------|----------|-------------|
| `name` | string | yes | Name of the Secret |
| `namespace` | string | yes | Namespace to create the Secret in |
| `values` | map | no | Key/value pairs. Values are base64-encoded automatically |
| `type` | string | no | Secret type. Defaults to `Opaque` |
| `annotations` | map | no | Additional annotations |
| `labels` | map | no | Additional labels |

## Example

```yaml
helmRepositories:
  - name: metallb
    namespace: bootstrap
    type: oci
    interval: 5m
    url: oci://registry-1.docker.io/bitnamicharts

namespaces:
  - name: my-app
    annotations:
      owner: platform-team
    labels:
      env: production

helmReleases:
  metallb:
    namespace: bootstrap
    spec:
      interval: 1m
      chart:
        spec:
          chart: metallb
          version: ">=0.14.0"
          sourceRef:
            kind: HelmRepository
            name: metallb
            namespace: bootstrap
      install:
        remediation:
          retries: 5
      upgrade:
        remediation:
          retries: 5

secrets:
  - name: registry-credentials
    namespace: my-app
    type: kubernetes.io/dockerconfigjson
    values:
      .dockerconfigjson: '{"auths":{"ghcr.io":{"auth":"<base64-token>"}}}'
```

## Publishing

This chart is published to GHCR via GitHub Actions on each `v*.*.*` tag push. To release a new version:

1. Update `version` in `Chart.yaml`
2. Push a tag matching the version: `git tag v1.1.0 && git push origin v1.1.0`
