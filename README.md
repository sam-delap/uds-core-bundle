# uds-core-bundle

A [UDS bundle](https://uds.defenseunicorns.com/structure/bundles/) for home lab (and
other bare-metal) clusters. It combines the prerequisite services package with a selected
set of [UDS Core functional layers](https://uds.defenseunicorns.com/reference/uds-core/functional-layers/),
deployed in the correct dependency order from a single artifact.

## What's in this bundle

| Order | Package                        | Ref              | Source                                                             |
| ----- | ------------------------------ | ---------------- | ------------------------------------------------------------------ |
| 1     | `uds-prereq-services`          | `1.0.1`          | `ghcr.io/sam-delap/uds-prereq-services`                            |
| 2     | `core-base`                    | `1.8.0-upstream` | `ghcr.io/defenseunicorns/packages/uds/core-base`                   |
| 3     | `uds-gateway-certs`            | `1.0.0`          | `ghcr.io/sam-delap/uds-gateway-certs`                              |
| 4     | `core-identity-authorization`  | `1.8.0-upstream` | `ghcr.io/defenseunicorns/packages/uds/core-identity-authorization` |
| 5     | `core-metrics-server`          | `1.8.0-upstream` | `ghcr.io/defenseunicorns/packages/uds/core-metrics-server`         |
| 6     | `core-logging`                 | `1.8.0-upstream` | `ghcr.io/defenseunicorns/packages/uds/core-logging`                |
| 7     | `core-monitoring`              | `1.8.0-upstream` | `ghcr.io/defenseunicorns/packages/uds/core-monitoring`             |

All UDS Core layers are pinned to the latest published **`upstream`** flavor, `1.8.0`.
Per UDS guidance, every core layer uses the **same version** for compatibility.

## Layering & ordering model

```
zarf init (stock)
  → uds-prereq-services   ← MetalLB + cert-manager (services + default config)
  → UDS Core layers       ← this bundle deploys base + selected functional layers
  → apps
```

- **MetalLB is a UDS Core prerequisite** — it must exist before `core-base` so Istio's
  ingress gateways (type `LoadBalancer`) can be assigned an address.
- **`core-base` must be first** among the core layers. It provides Istio, the UDS
  Operator, and the UDS Policy Engine that every other layer assumes.
- **`uds-gateway-certs` runs right after `core-base`** — it needs the Istio gateway
  namespaces and the prereq ClusterIssuers to exist.
- **`core-monitoring` is last.** It provides user login and therefore depends on
  `core-identity-authorization`.

See the UDS Core
[prerequisites](https://uds.defenseunicorns.com/reference/uds-core/prerequisites/) and
[functional layers](https://uds.defenseunicorns.com/reference/uds-core/functional-layers/) docs.

## Prerequisite services

The `uds-prereq-services` package (`1.0.1`) provides MetalLB + cert-manager, and ships its
on-by-default config components (`metallb-config` IPAddressPool/L2Advertisement,
`cert-manager-config` Let's Encrypt ClusterIssuers via Cloudflare DNS-01), kept via
`optionalComponents`. In a UDS bundle, optional components are only deployed when listed
under `optionalComponents`.

> **MetalLB policy exemption:** MetalLB's `speaker` pods require privileged Pod Security.
> The UDS Core Policy Engine can block them on reconciliation/upgrade unless an
> `Exemption` exists. This exemption is managed in the `uds-prereq-services` package (as a
> separate component), not in this bundle.

## Gateway TLS (`uds-gateway-certs`)

UDS Core's Istio tenant/admin gateways ship placeholder `uds.dev` certs. This bundle
serves real certs for `uds.sams-club-it.com` via the `uds-gateway-certs` package
(`ghcr.io/sam-delap/uds-gateway-certs:1.0.0`), which requests wildcard cert-manager
`Certificate`s from the prereq ClusterIssuers:

| Gateway | Namespace              | Secret              | DNS name                       |
| ------- | ---------------------- | ------------------- | ------------------------------ |
| tenant  | `istio-tenant-gateway` | `gateway-tls`       | `*.uds.sams-club-it.com`       |
| admin   | `istio-admin-gateway`  | `admin-gateway-tls` | `*.admin.uds.sams-club-it.com` |

The `core-base` package sets each gateway's `tls.credentialName` (via bundle `overrides`)
to the matching secret. `uds-gateway-certs` deploys **after `core-base`** so the gateway
namespaces and ClusterIssuers exist; Istio hot-reloads the gateway once the secret is
issued.

The base domain and issuer are Zarf variables on the `uds-gateway-certs` package: `DOMAIN`
(default `uds.sams-club-it.com`) and `CERT_ISSUER` (default `letsencrypt-staging`).
Validate against staging first, then flip to production and redeploy:

```bash
uds run deploy --set CERT_ISSUER=letsencrypt-prod
```

The gateway domain is set cluster-wide via `uds-config.yaml` (`shared.domain`).

### Required secret (out-of-band)

The prereq's cert-manager ClusterIssuers reference a Cloudflare API token Secret you must
create yourself — it is never committed to git:

```bash
kubectl create secret generic cloudflare-api-token \
  --namespace cert-manager \
  --from-literal=api-token='<cloudflare-dns-edit-token>'
```

The token needs `Zone:DNS:Edit` permission for `sams-club-it.com`. Create it after the
first deploy (the `cert-manager` namespace is created by the prereq); cert-manager then
issues the gateway certs and Istio hot-reloads them.

## Building, deploying & publishing

Requires `uds` (uds-cli) and `zarf` on PATH.

```bash
uds run build   --set VERSION=0.1.0   # create the bundle artifact
uds run deploy  --set VERSION=0.1.0   # deploy to the current cluster
uds run publish --set VERSION=0.1.0   # push to the OCI registry
uds run inspect --set VERSION=0.1.0   # inspect SBOM / images / metadata
uds run lint    --set VERSION=0.1.0   # validate the bundle definition
```

Task vars (`tasks.yaml`): `VERSION` (default `0.1.0`), `REGISTRY`
(`ghcr.io/sam-delap`), `ARCH` (`amd64`).

## Versioning

`metadata.version` in `uds-bundle.yaml` is a literal version string. The `uds run` tasks
pass it through with `uds create --version ${VERSION}`; keep the two in sync (or override
at build time with `--set VERSION=`). CI publishing via semantic-release can be wired up
later.
