# ITL.Forgejo

Wrapper Helm chart for self-hosted [Forgejo](https://forgejo.org/), deployed
via the official [`forgejo-helm`](https://code.forgejo.org/forgejo-helm/forgejo-helm)
chart as a dependency (distributed via OCI, not a classic chart repo) — this
repo just holds our values overrides and cluster-specific wiring (Traefik
ingress, cert-manager, external Postgres, in-cluster Redis), same pattern
as `ITL.Gitlab` and `rustdesk-server`.

- Chart: `charts/itl-forgejo` (depends on `forgejo/forgejo` v17.3.1, app v15.0.10)
- Upstream: `oci://code.forgejo.org/forgejo-helm/forgejo` ·
  https://code.forgejo.org/forgejo-helm/forgejo-helm
- Originally published on Codeberg
  (https://codeberg.org/forgejo-contrib/forgejo-helm), now archived there —
  active development and the OCI registry moved to code.forgejo.org.

## Not quite deploy-ready

Unlike `gitlab/gitlab`, this chart does **not** hard-require external
services — it falls back to SQLite + in-memory cache/session/queue if left
unconfigured, which is fine for a quick try-out. `values.yaml` here wires
Postgres to an external instance (upstream explicitly says the
in-memory/SQLite defaults are "not recommended for production use"), but
uses **Redis deployed in-cluster by this chart itself**
(`templates/redis.yaml`, gated by `redis.enabled`, default `true`) — a
single-replica Deployment+PVC+Service, no external Redis needed. Forgejo's
Redis load (cache/session/queue for one instance) doesn't need HA or a
managed service.

| Need | Source | Values.yaml key |
|---|---|---|
| PostgreSQL | External — CNPG cluster, see `ITL.CNPG.Cluster` | `forgejo.gitea.config.database.HOST` + `forgejo-postgresql-password` secret (still `CHANGEME-...`, needs real infra) |
| Redis (cache/session/queue) | In-cluster, bundled with this chart | `redis.*` (works out of the box — nothing to provision) |
| Admin user | — | `forgejo-admin-secret` (username/password/email keys) |

The DB password is injected via `gitea.additionalConfigFromEnvs` (rendered
as `GITEA__DATABASE__PASSWD`, sourced from a Secret) — nothing sensitive is
in `values.yaml` itself. The in-cluster Redis has no auth configured (fine
for an in-cluster-only, unexposed service).

## Prerequisites

- The Secrets referenced above, created ahead of install:
  ```bash
  kubectl create secret generic forgejo-admin-secret -n forgejo \
    --from-literal=username=admin \
    --from-literal=password='...' \
    --from-literal=email=admin@itlusions.com

  kubectl create secret generic forgejo-postgresql-password -n forgejo \
    --from-literal=password='...'
  ```
- Traefik ingress controller + cert-manager with a working
  `letsencrypt-issuer` `ClusterIssuer` (already in place cluster-wide,
  same as `itl.website` / `rustdesk-server` / `ITL.Gitlab`)

## Install

```bash
cd charts/itl-forgejo
helm dependency update .
helm upgrade --install forgejo . \
  --namespace forgejo --create-namespace \
  --timeout 600s
```

## What's overridden vs. upstream defaults

See `charts/itl-forgejo/values.yaml` for the full annotated list — in short:

- `ingress.*` → enabled, `className: traefik`,
  `cert-manager.io/cluster-issuer: letsencrypt-issuer` annotation, host
  `forgejo.dev.itlusions.com` (TODO: confirm), own TLS secret (this chart
  uses one plain `Ingress` resource, not the multi-Ingress-per-component
  shape `gitlab/gitlab` has — no shared-secret collision to worry about
  here, unlike `ITL.Gitlab`)
- `gitea.config.server.{DOMAIN,ROOT_URL,SSH_DOMAIN}` → matches the ingress host
- `gitea.config.database` → external Postgres (placeholder host; password
  via `additionalConfigFromEnvs` + Secret, not plaintext)
- `redis.enabled: true` + `gitea.config.{cache,session,queue}` → points at
  the in-cluster Redis this chart deploys
  (`<release>-redis.<namespace>.svc.cluster.local`, hardcoded assuming
  `helm install forgejo -n forgejo` per the command below — adjust if you
  install under a different release name/namespace)
- `gitea.admin.existingSecret` → `forgejo-admin-secret`
- `persistence.size` → 10Gi (chart default), `storageClass` left to cluster
  default — confirm this is what you want

## Verify after install

```bash
kubectl get pods -n forgejo
kubectl get ingress -n forgejo
kubectl get certificate -n forgejo
```
