# AGENTS.md — Cospace Deployment Project

This repo manages a Kubernetes deployment of **Cospace** via its Helm chart. This file gives AI coding agents (Claude Code, etc.) the context needed to work safely in this project.

## What this project is

Cospace is deployed with a Helm chart published as an OCI artifact:

```
oci://ghcr.io/twigex/helm-charts/cospace
```

No `helm repo add` is needed — it's pulled directly by OCI reference and version tag (e.g. `--version 0.0.34`).

Stack components (all optional except Cospace + one DB mode + Redis):

- **Cospace** — the app itself (required)
- **MariaDB** — one of 4 modes: embedded, operator-single, operator-Galera/MaxScale, or externally managed
- **Redis** — required, deployed as a subchart
- **LiveKit** — optional, *deployed separately*; this chart only configures the cospace client side (`cospace.channel.*`) for voice/video
- **EuroOffice** or **Collabora** — optional, *deployed separately*; this chart only configures the cospace client side (`cospace.office.*`) for office document editing (WOPI)
- **Ingress** — opt-in (`ingress.enabled=true`), off by default

## Repo layout conventions

> ⚠️ **Proposed layout — not yet adopted in this repo.** Current state: single root `values.yaml` + `INSTALLATION.md`. Agents may propose this layout but must confirm with the user before creating any of these directories.

Agents should expect (or create) a layout like:

```
.
├── AGENTS.md
├── values/
│   ├── values-base.yaml        # shared, non-secret config
│   ├── values-prod.yaml        # env-specific overrides
│   └── values-dev.yaml
├── secrets/                    # NEVER commit real secrets here — see below
│   └── .gitkeep
└── scripts/
    ├── install.sh
    ├── upgrade.sh
    └── verify.sh
```

If this layout doesn't exist yet, agents may propose it but should confirm with the user before restructuring existing files.

## Critical rules — read before editing anything

1. **Never commit real passwords, encrypt keys, or API secrets** to values files that get checked into git. Use `--set` at install time, an external secret manager (Vault / External Secrets Operator), or a git-ignored local values file.
2. **`cospace.server.encryptKey` is immutable in practice.** It's exactly 32 hex chars (`openssl rand -hex 16`). Rotating it after go-live invalidates all encrypted data in the DB. Never regenerate it in an upgrade script.
3. **`mariadb.password` cannot be changed after first boot** with embedded MariaDB without reinitializing the DB (data loss). Treat it as a one-time-set value.
4. **Ingress is opt-in.** Don't assume `ingress.enabled=true`; check the values file before assuming the app is externally reachable.
5. **Service addresses are auto-derived from the Helm release name** (`<release>-redis-master:6379`, `<release>-mariadb`). Don't hardcode different service names unless the release name actually changes — and if it does, update every reference.
6. **`cospace.db.host`, `cospace.redis.address`, `cospace.redis.password`** in `values.yaml` are deprecated/ignored by the chart. Don't "fix" these — they're intentionally inert placeholders kept for backward compatibility. The real values are auto-derived or come from `redis.auth.password` / release name.
7. **No `lookup()` calls and no random generation (`randAlphaNum`, `randAlpha`, etc.) in chart templates.** Both break ArgoCD's diff/ServerSideApply and cause perpetual "diff detected" syncs. All values must be explicit — user-provided via `--set` or values file, or static defaults in `values.yaml`. No cluster lookups, no in-template password/key generation.

## Common agent tasks

### Install
Use the mode-specific command from the Cospace manual (Installation page). Always require: `domain`, `cospace.admin.password`, `cospace.server.encryptKey`, `redis.auth.password`, `mariadb.password` (+ `mariadb.rootPassword` unless mode 4 with `mariadb.externalSecret=true`).

### Upgrade
```sh
helm upgrade cospace oci://ghcr.io/twigex/helm-charts/cospace --version <new-version> \
  --namespace cospace -f your-values.yaml
```
Always check the chart's Changelog for breaking changes before bumping versions, especially around the database backend.

### Verify health
```sh
kubectl get pods -n cospace
kubectl get svc -n cospace
kubectl logs -n cospace -l app=<sanitized-domain> -f
```
`<sanitized-domain>` = the `domain` value with dots replaced by dashes.

### Uninstall
```sh
helm uninstall cospace --namespace cospace
kubectl delete namespace cospace
```
Note: PVCs, Secrets, Role, and RoleBinding have `helm.sh/resource-policy: keep` and survive `helm uninstall` by design — don't "clean these up" automatically unless the user explicitly asks for a full wipe.

## Testing / validation expectations

Before proposing a values-file change, an agent should:
1. Run `helm template` (or `helm install --dry-run`) against the new values to catch template-render failures early (e.g. the chart will hard-fail if `encryptKey` isn't exactly 32 hex chars).
2. Diff against the currently-deployed values where possible.
3. Flag any change to `mariadb.password`, `cospace.server.encryptKey`, or `domain` as high-risk and ask for explicit confirmation.

## Where to look for more detail

Full chart configuration reference (all `values.yaml` keys, defaults, types): see the Cospace manual's Installation page → Configuration section, or `templates/cospace-secret.yaml` in the chart for exact secret key names.

Also see `SKILL.md` in this repo for a condensed, agent-oriented operating procedure.
