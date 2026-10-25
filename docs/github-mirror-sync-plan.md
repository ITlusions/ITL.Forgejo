# Plan: sync GitHub repos into Forgejo as pull mirrors

Not implemented yet - this describes the changes, for review before any file is
written or anything is applied to the cluster.

## Goal

Every repo under `github.com/ITlusions` gets a matching pull mirror in this
Forgejo instance, living in a real `itlusions` organization. GitHub stays the
source of truth; Forgejo periodically pulls updates (default interval: 8h).
No push-back to GitHub, no cutover - this is additive, safe to run alongside
existing GitHub usage.

## Open questions (need an answer before building)

1. **Scope**: all 116 repos (98 public, 18 private), or just the
   `ITL.*`-prefixed infra repos? The full list includes things like
   `prefect`, `webull-openapi-cli`, `brk-mutaties`, `maatwerk-poc`/`-legacy`,
   `HNL-HVM`, `ITL.IB`, `ITL.CV` that may not belong here.
2. **Mirror interval**: default 8h okay, or faster/slower?

## Why a Helm hook Job, not a CRD or CLI command

Forgejo has no CLI subcommand for repo migration (confirmed earlier via
`forgejo admin --help` for orgs - same absence applies to repos) and no CRD
for it either. The only way in is the REST API (`POST /repos/migrate`), same
reasoning as `templates/seed-organizations-job.yaml` for organizations - this
reuses that exact pattern: a post-install/post-upgrade hook Job, idempotent
(GET first, only POST on 404), using the admin credentials already generated.

## Changes

### 1. `values.yaml` - populate `organizations`

Currently `organizations: []` (inert placeholder). Change to actually create
the org these mirrors live in:

```yaml
organizations:
  - name: itlusions
    fullName: "ITlusions"
    visibility: private
```

### 2. `values.yaml` - new `githubMirrors` block

A static list (Helm can't call the GitHub API at template time, so this is
generated once from `gh repo list ITlusions` and checked in, not computed).
Already generated from the real repo list - 116 entries, each with a
`private` flag so the Job knows whether to attach the GitHub token:

```yaml
githubMirrors:
  enabled: true
  interval: "8h0m0s"
  # GitHub PAT, repo-read scope, only used for the private entries below.
  # Not needed if scope is narrowed to public-only repos.
  tokenSecretName: github-mirror-token
  tokenSecretKey: token
  repos:
    - name: "ITL.CNPG"
      private: false
    - name: "ITL.Forgejo"
      private: true
    # ... full list, 116 entries total
```

### 3. New file: `templates/sync-github-mirrors-job.yaml`

Same shape as `templates/seed-organizations-job.yaml`:

- `helm.sh/hook: post-install,post-upgrade`
- `curlimages/curl` image, admin credentials from `forgejo-admin-secret`
  (`ADMIN_USER`/`ADMIN_PASS` env vars, same as the org-seeding job)
- GitHub token from the `github-mirror-token` Secret (only mounted/used if
  `githubMirrors.enabled` and at least one entry is `private: true`)
- Per repo: `GET /api/v1/repos/itlusions/<name>` - if 404, `POST
  /api/v1/repos/migrate` with:
  ```json
  {
    "clone_addr": "https://github.com/ITlusions/<name>.git",
    "repo_name": "<name>",
    "repo_owner": "itlusions",
    "mirror": true,
    "mirror_interval": "8h0m0s",
    "service": "git",
    "auth_token": "<only for private repos>"
  }
  ```
- Idempotent: existing repos are left untouched, safe to re-run on every
  `helm upgrade` as the repo list grows

### 4. New Secret (created by hand, not by Helm)

A GitHub PAT with repo-read scope, for the private repos only. Not generated
by me - you create it, then run:

```bash
kubectl create secret generic github-mirror-token -n forgejo \
  --from-literal=token='<the PAT>'
```

Referenced by `githubMirrors.tokenSecretName`/`tokenSecretKey` in values.yaml,
same `existingSecret`-only pattern as everything else credential-related in
this chart - never generated or stored by Helm itself.

## What this does NOT do

- No push mirroring back to GitHub (GitHub stays authoritative)
- No issues/PRs/wiki/releases migration - git refs only (Forgejo's migrate
  API can pull those too, but that's a separate, heavier decision)
- No automatic pruning if a GitHub repo is later deleted or renamed - the
  mirror just stops updating, nothing removes it

## Rollout

1. Confirm scope (full org vs. `ITL.*` only) and mirror interval
2. Create the GitHub PAT and the `github-mirror-token` Secret by hand
3. Add the two `values.yaml` blocks and `templates/sync-github-mirrors-job.yaml`
4. `helm upgrade forgejo .` - the Job runs once, creates every missing mirror
5. Verify a handful of mirrors in the Forgejo UI (`itlusions` org), confirm
   at least one private repo actually cloned (proves the token works)
