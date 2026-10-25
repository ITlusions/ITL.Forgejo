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

Further docs (plans, deeper write-ups) live in [`docs/`](docs/).

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
| PostgreSQL | In-cluster CNPG `Cluster`, bundled with this chart (`templates/cnpg-cluster.yaml`) — needs the CNPG operator already installed cluster-wide, see `itl.cnpg/operator` | `postgresql.*` (works out of the box — CNPG auto-creates the `forgejo` db/user and a `forgejo-db-app` Secret, nothing to provision by hand) |
| Redis (cache/session/queue) | In-cluster, bundled with this chart | `redis.*` (works out of the box — nothing to provision) |
| Admin user | Auto-generated on first install (`templates/admin-secret.yaml`) | `adminSecret.*` — password is random, generated once and kept stable across upgrades |

The DB password is injected via `gitea.additionalConfigFromEnvs` (rendered
as `GITEA__DATABASE__PASSWD`, sourced from CNPG's own `forgejo-db-app`
Secret) — nothing sensitive is in `values.yaml` itself. Postgres is
in-cluster only by design (no ingress/`IngressRouteTCP` for it, reachable
only via the `forgejo-db-rw` ClusterIP service CNPG generates) — the
in-cluster Redis likewise has no auth configured (fine for an
in-cluster-only, unexposed service).

## Prerequisites

- The CNPG operator installed cluster-wide first (see `itl.cnpg/operator`)
  — `templates/cnpg-cluster.yaml` here needs the `Cluster` CRD to exist.
- Traefik ingress controller + cert-manager with a working
  `letsencrypt-issuer` `ClusterIssuer` (already in place cluster-wide,
  same as `itl.website` / `rustdesk-server` / `ITL.Gitlab`)

Nothing else to create by hand - both the admin password and the database
credentials are generated automatically on first install.

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
  `cert-manager.io/cluster-issuer: letsencrypt-issuer` annotation, two
  hosts (`git.itlusions.com` canonical, `git.dev.itlusions.com` as a
  second host routing to the same instance), each with its own TLS
  secret (this chart uses one plain `Ingress` resource, not the
  multi-Ingress-per-component shape `gitlab/gitlab` has — no
  shared-secret collision to worry about here, unlike `ITL.Gitlab`)
- `gitea.config.server.{DOMAIN,ROOT_URL,SSH_DOMAIN}` → matches the ingress host
- `gitea.config.database` → in-cluster CNPG Postgres (see `postgresql.*`
  above); password via `additionalConfigFromEnvs` + CNPG's own Secret, not
  plaintext
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

Retrieve the auto-generated admin password:

```bash
kubectl get secret forgejo-admin-secret -n forgejo -o jsonpath="{.data.password}" | base64 --decode
```
