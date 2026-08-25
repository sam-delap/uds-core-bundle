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
  ordering, the `core-base` gateway `credentialName` overrides, and the narrow
  `local-path-provisioner` UDS policy exemption).
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
  Operator, and the Policy Engine that all other layers assume. The `uds-exemptions`
  override for `local-path-provisioner` helper pods lives on this package.
- **`uds-gateway-certs` must come right after `core-base`** — it needs the Istio gateway
  namespaces and the prereq ClusterIssuers to exist.
- **`core-monitoring` must come after `core-identity-authorization`** — monitoring
  provides user login and depends on identity/auth. Keep it last.
- **`postgres-operator` must come last** — after `core-monitoring`. It needs `core-base`
  (Istio, UDS Operator, Policy Engine) to exist. It is the only app package in the bundle.
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
- `postgres-operator` is pinned to the latest published **`upstream`** tag
  (`ghcr.io/uds-packages/postgres-operator:1.15.1-uds.5-upstream`). Discover with:
  `zarf tools registry ls ghcr.io/uds-packages/postgres-operator | grep -- -upstream$`.

## Postgres datastores (two-phase)

- Only **Keycloak** and **Grafana** use SQL. Both default to embedded storage and can be
  switched to external Postgres (provisioned in-cluster by `postgres-operator`) purely via
  `uds-config.yaml` variables — the same bundle artifact serves both phases, no rebuild.
- Bundle variables (empty/false defaults): `postgres-operator` exposes `pg_cluster_enabled`,
  `pg_users`, `pg_databases`, etc.; `core-identity-authorization` exposes `kc_postgresql`
  (whole `postgresql` map); `core-monitoring` exposes `gf_postgresql` map + sensitive
  `gf_pg_password`.
- The `kc_postgresql` / `gf_postgresql` map overrides **replace the whole chart subtree** —
  Phase 2 maps must be complete. Grafana password is env-sourced (`UDS_GF_PG_PASSWORD`),
  never committed. See README for the Phase 2 block.

## Out of scope for this repo

- **MetalLB policy Exemption** for the Policy Engine lives in the `uds-prereq-services`
  package (as a separate component), not here.
- **local-path-provisioner policy Exemption** lives here as a `core-base` override because
  the `Exemption` CRD and UDS policy engine are installed by `core-base`. Keep it limited
  to `local-path-storage` `helper-pod.*` pods and only policies needed for local-path
  helper pod host-path directory creation.
- **Cloudflare API token Secret** is created out-of-band (see README); never commit it.
