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
| 8     | `postgres-operator`            | `1.15.1-uds.5-upstream` | `ghcr.io/uds-packages/postgres-operator`                    |

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
  Operator, and the UDS Policy Engine that every other layer assumes. This bundle also
  injects a narrow `uds-exemptions` override into `core-base` so k3s
  `local-path-provisioner` helper pods can create writable hostPath-backed PVs for
  `local-path` PVCs.
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

> **Policy exemptions:** MetalLB's `speaker` pods require privileged Pod Security; that
> exemption is managed in the `uds-prereq-services` package. k3s
> `local-path-provisioner` creates writable hostPath-backed PVs via short-lived
> `helper-pod` pods in `local-path-storage`; this bundle adds a narrow `core-base`
> `uds-exemptions` override for only those helper pods and only the policies needed to
> preserve the helper pod's required host write behavior and root/capability defaults.

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

## Postgres datastores (two-phase: embedded → Postgres)

Keycloak and Grafana are the only services in these layers that use a SQL database.
Both default to embedded storage (Keycloak `devMode`/H2, Grafana SQLite) and both can be
switched to external Postgres provisioned in-cluster by the bundled `postgres-operator`.

The switch is driven entirely by `uds-config.yaml` — the **same bundle artifact** produces
either state depending on the variables at deploy time. No rebuild is needed between phases.

Ordering: `postgres-operator` deploys **last**. It requires `core-base` (Istio, UDS
Operator, Policy Engine) to exist first.

### Phase 1 — embedded (default)

Deploy with the stock `uds-config.yaml`. All Postgres variables default to empty/false:
the operator installs but provisions no cluster, Keycloak runs in `devMode`, Grafana on
SQLite. Validate Core, then move to Phase 2.

### Phase 2 — cut over to Postgres

> **State reset:** switching backends does not migrate data. Keycloak re-seeds the `uds`
> realm from the identity-config image and Grafana re-provisions dashboards from
> ConfigMaps, so declarative state is restored automatically; ad-hoc runtime data is lost.

1. Add the block below to `uds-config.yaml`.
2. Read the operator-generated Grafana password and export it (never commit it):

   ```bash
   export UDS_GF_PG_PASSWORD="$(kubectl get secret grafana.pg-cluster.credentials.postgresql.acid.zalan.do \
     -n grafana -o jsonpath='{.data.password}' | base64 -d)"
   ```

3. Redeploy the same tarball: `uds run deploy`.

```yaml
shared:
  domain: uds.sams-club-it.com
variables:
  postgres-operator:
    pg_cluster_enabled: true
    pg_users:
      keycloak.keycloak: []
      grafana.grafana: []
    pg_databases:
      keycloakdb: keycloak.keycloak
      grafanadb: grafana.grafana
  core-identity-authorization:
    # Full map — replaces the chart subtree, so include every field.
    kc_postgresql:
      host: pg-cluster.postgres-operator.svc.cluster.local
      port: 5432
      database: keycloakdb
      internal:
        enabled: true
        remoteNamespace: postgres-operator
        remoteSelector:
          application: spilo
      secretRef:
        username:
          name: keycloak.pg-cluster.credentials.postgresql.acid.zalan.do
          key: username
        password:
          name: keycloak.pg-cluster.credentials.postgresql.acid.zalan.do
          key: password
  core-monitoring:
    gf_postgresql:
      host: pg-cluster.postgres-operator.svc.cluster.local
      port: 5432
      database: grafanadb
      user: grafana
      ssl_mode: require
      internal:
        enabled: true
        remoteNamespace: postgres-operator
        remoteSelector:
          application: spilo
    # gf_pg_password supplied via UDS_GF_PG_PASSWORD (do not commit)
```

Notes:
- Confirm the operator secret name and `remoteSelector` against your cluster; Zalando uses
  `{user}.{cluster}.credentials.postgresql.acid.zalan.do` and pod label `application: spilo`.
- Keycloak's `postgresql` map override replaces the whole subtree — keep the map complete.
- Grafana's chart has no `secretRef`; its password is a plaintext value sourced from the
  `GF_PG_PASSWORD` variable (env `UDS_GF_PG_PASSWORD` recommended).

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
