# AGENTS.md

## What this repo is

A **UDS bundle** (not application code, not a single Zarf package). It composes several
already-published OCI artifacts into one deployable bundle:

- `uds-prereq-services` (MetalLB + cert-manager prerequisite services),
- `uds-gateway-certs` (wildcard cert-manager `Certificate`s for the Istio tenant/admin
  gateways), and
- selected **UDS Core functional layers** (`core-base`, `core-identity-authorization`,
  `core-metrics-server`, `core-logging`, `core-monitoring`).

The bundle only references and orders published packages; it contains no Helm charts or
manifests of its own.

## Layout

- `uds-bundle.yaml` — the bundle definition (source of truth for the package list, refs,
  ordering, and the `core-base` gateway `credentialName` overrides).
- `uds-config.yaml` — cluster-wide config; sets `shared.domain`. Auto-loaded by `uds
  deploy` from CWD. No secrets.
- `tasks.yaml` — `uds run` task runner (build/deploy/publish/inspect/lint).
- `README.md` — layering model, consumption, and commands.

## Commands

Requires `uds` (uds-cli) and `zarf` on PATH.

```bash
uds run lint   --set VERSION=0.1.0   # validate before committing
uds run build  --set VERSION=0.1.0
uds run deploy --set VERSION=0.1.0
```

Task vars (`tasks.yaml`): `VERSION` (default `0.1.0`), `REGISTRY`
(`ghcr.io/sam-delap`), `ARCH` (`amd64`).

## Versioning / release

- `metadata.version` in `uds-bundle.yaml` is a **literal** version string. The `uds run`
  tasks pass it via `uds create --version ${VERSION}` (override with `--set VERSION=`).
  Keep the literal and the task var in sync.
- CI publishing (semantic-release) is not yet wired up; add it later if desired.

## Ordering gotchas (keep intact when editing)

- **`uds-prereq-services` must be first** — MetalLB must exist before `core-base` so
  Istio LoadBalancer ingress gateways get an address.
- **`core-base` must be first among the core layers** — it provides Istio, the UDS
  Operator, and the Policy Engine that all other layers assume.
- **`uds-gateway-certs` must come right after `core-base`** — it needs the Istio gateway
  namespaces and the prereq ClusterIssuers to exist.
- **`core-monitoring` must come after `core-identity-authorization`** — monitoring
  provides user login and depends on identity/auth. Keep it last.
- All UDS Core layers must share the **same version/flavor** (`1.8.0-upstream`). If you
  bump one, bump them all together.

## Version pinning

- UDS Core layers use the latest published **`upstream`** flavor. Discover versions with:
  `zarf tools registry ls ghcr.io/defenseunicorns/packages/uds/core-base`.
- `uds-prereq-services` is pinned to a published tag
  (`ghcr.io/sam-delap/uds-prereq-services:1.0.1`), consumed with its config components
  (`metallb-config`, `cert-manager-config`) kept via `optionalComponents`.
- `uds-gateway-certs` is pinned to a published tag
  (`ghcr.io/sam-delap/uds-gateway-certs:1.0.0`). Its `DOMAIN` / `CERT_ISSUER` Zarf vars
  default to `uds.sams-club-it.com` / `letsencrypt-staging`.

## Out of scope for this repo

- **MetalLB policy Exemption** for the Policy Engine lives in the `uds-prereq-services`
  package (as a separate component), not here.
- **Cloudflare API token Secret** is created out-of-band (see README); never commit it.
