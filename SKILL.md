---
name: cospace-deployment
description: Operate, install, upgrade, configure, or troubleshoot the Cospace Helm chart deployment on Kubernetes. Use this skill whenever the user mentions Cospace, cospace.mytwigex.com, the Cospace Helm chart, or asks about installing/upgrading/configuring Cospace, its MariaDB backend, Redis, LiveKit (Channel/WebRTC), EuroOffice, or Collabora integration for this project — even if they don't say "Cospace" explicitly and just describe symptoms (e.g. "office editing isn't connecting", "chat isn't working", "helm install failed with encryptKey error"). Always consult this skill before running helm commands against this project.
---

# Cospace Deployment Skill

Operating procedure for installing, upgrading, configuring, and troubleshooting the Cospace Helm chart (`oci://ghcr.io/twigex/helm-charts/cospace`).

## Quick facts

- OCI chart — no `helm repo add` needed.
- Required for **every** install: `domain`, `cospace.admin.password`, `cospace.server.encryptKey` (32 hex chars, `openssl rand -hex 16`), `redis.auth.password`, `mariadb.password`.
- Required additionally for modes 1–3: `mariadb.rootPassword` (skip only in mode 4 with `mariadb.externalSecret=true`).
- Ingress is **off by default** — must opt in with `ingress.enabled=true`.
- No CPU/memory requests/limits are set by the chart — add `resources:` blocks yourself for production.

## Step 1 — Identify the DB mode needed

Ask (or infer from context) which of these fits:

| Mode | When to use | Extra prerequisite |
|---|---|---|
| 1. Embedded MariaDB | dev/test, small/medium prod | none |
| 2. MariaDB Operator (single) | prod, want backups/PITR/metrics | mariadb-operator + CRDs pre-installed |
| 3. MariaDB Operator (Galera+MaxScale) | HA prod | 3+ worker nodes, operator pre-installed |
| 4. External MariaDB | bring-your-own DB (on/off-cluster, RDS) | pre-created secrets/service, see below |

If operator mode (2 or 3) is chosen and the operator isn't installed yet, install it first:
```sh
helm repo add mariadb-operator https://helm.mariadb.com/mariadb-operator
helm repo update
helm install mariadb-operator-crds mariadb-operator/mariadb-operator-crds \
  --namespace mariadb-operator --create-namespace
helm install mariadb-operator mariadb-operator/mariadb-operator \
  --namespace mariadb-operator --version 25.10.2
```

For mode 4, the user must pre-create:
- Secret `<release>-cospace` (see `templates/cospace-secret.yaml` in the chart for keys)
- Secret `<release>-mariadb` with keys `mariadb-password`, `mariadb-root-password`
- Service `<release>-mariadb` pointing at the external DB on port 3306

## Step 2 — Run the install

**Mode 1 (default, embedded MariaDB):**
```sh
helm install cospace oci://ghcr.io/twigex/helm-charts/cospace --version 0.0.34 \
  --namespace cospace --create-namespace \
  --set domain=example.com \
  --set cospace.admin.password=your_password \
  --set cospace.server.encryptKey=$(openssl rand -hex 16) \
  --set mariadb.password=your_mariadb_password \
  --set mariadb.rootPassword=your_mariadb_root_password \
  --set redis.auth.password=your_redis_password
```

**Mode 2 (operator, single instance):** use a values file with `mariadb.enabled: false` and `mariadbOperatorSingle.enabled: true`. `mariadb.password`/`mariadb.rootPassword` are still required — the operator CRs and `cospace-secret` both read them.

**Mode 3 (operator, Galera+MaxScale):** same as mode 2 but `mariadbOperatorMaxscaleGalera.enabled=true`. Needs 3+ worker nodes with persistent storage.

**Mode 4 (external):** `mariadb.enabled=false`, `mariadb.externalSecret=true`, still set `mariadb.password` (must match what's in the pre-created secret) and skip `mariadb.rootPassword`.

Add ingress if the app needs to be externally reachable:
```sh
--set ingress.enabled=true \
--set ingress.ingressClassName=traefik   # or nginx
```
Add TLS via cert-manager (requires an existing `ClusterIssuer`):
```sh
--set ingress.tls.enabled=true \
--set ingress.certManagerIssuer=letsencrypt-prod
```

## Step 3 — Verify

```sh
kubectl get pods -n cospace
kubectl get svc -n cospace
kubectl get pods -n cospace --show-labels   # find exact app= label
kubectl logs -n cospace -l app=<sanitized-domain> -f
kubectl get mariadb -n cospace               # operator modes only
```
Healthy state: cospace pod `Running` and listening on port 3000 (or `cospace.server.port`); `<release>-mariadb-0` and `<release>-redis-master-0` `Running`.
Healthy state — pod names differ by MariaDB mode:
- **Mode 1 (embedded):** `mariadb-deployment-<hash>` and `<release>-redis-master-0` `Running`
- **Modes 2/3 (operator):** `<release>-mariadb-0` and `<release>-redis-master-0` `Running`
The cospace pod label is `app=<sanitized-domain>`; the cospace Service is always `<release>`. The MariaDB Service is always `<release>-mariadb` regardless of mode — checking via Service is mode-independent.

## Optional dependencies — when the user asks about them

- **LiveKit (Channel/voice-video):** chart only configures Cospace's client side (`cospace.channel.*`). LiveKit server itself must be installed separately via `helm.livekit.io`. Needs an API key/secret from `lk api create-key` and a Redis address (reuse `<release>-redis-master:6379`).
- **EuroOffice / Collabora (office editing):** set `cospace.office.enabled=true`, `cospace.office.type` to `"eurooffice"` or `"collabora"` (same WOPI API for both), `cospace.office.host`. The office server's WOPI allow-list (`aliasgroup1` or equivalent) must include the Cospace domain, and that domain must be reachable from end-user browsers — this is the #1 cause of "office editing won't load" issues.

## Troubleshooting playbook

| Symptom | Likely cause | Fix |
|---|---|---|
| `helm install` fails at template render, encryptKey error | `cospace.server.encryptKey` missing, wrong length, or non-hex | Must be exactly 32 hex chars: `openssl rand -hex 16` |
| App can't reach DB/Redis after changing release name | Service addresses are auto-derived from release name | Use `--name cospace` (default) or update all downstream refs to match new `<release>-*` names |
| Office editing fails to load / WOPI errors | Office server's alias/allow-list doesn't include Cospace domain, or domain unreachable from browsers | Check `aliasgroup1` (or equivalent) on EuroOffice/Collabora side |
| Voice/video (Channel) not connecting | LiveKit not installed, or `cospace.channel.host`/key/secret misconfigured | Verify LiveKit server is up and reachable at the configured `ws://` URL |
| Upgrade breaks DB connectivity | Breaking change in DB backend defaults between chart versions | Read the chart Changelog before bumping `--version` |
| Want to rotate `mariadb.password` or `encryptKey` | Not supported live with embedded MariaDB (password) or safely at all (encryptKey — invalidates encrypted data) | Treat both as fixed at install time; changing requires reinit/data migration |
| PVCs/Secrets still exist after `helm uninstall` | Intentional — `helm.sh/resource-policy: keep` on PVCs and Secrets | Delete manually if a full wipe is wanted: `kubectl delete pvc -n cospace --all` etc. |

## Full configuration reference

For any `values.yaml` key not covered above (SMTP, LDAP, Gitea OAuth, S3 storage, external secret management via `<release>-cospace`/`<release>-mariadb` secrets, persistence sizing, etc.), consult the Cospace manual's Installation page Configuration section directly — don't guess at key names or defaults.
