# Cospace Helm Chart — Installation Guide

This guide covers installing the Cospace Helm chart and its optional dependencies (MariaDB operator, LiveKit, EuroOffice/Collabora).

## Table of Contents

- [Prerequisites](#prerequisites)
- [Repository](#repository)
- [System Requirements](#system-requirements)
- [Install Cospace](#install-cospace)
  - [1. Default — Embedded MariaDB](#1-default--embedded-mariadb)
  - [2. With MariaDB Operator (single instance)](#2-with-mariadb-operator-single-instance)
  - [3. With MariaDB Operator (Galera + MaxScale)](#3-with-mariadb-operator-galera--maxscale)
  - [4. Externally Managed MariaDB](#4-externally-managed-mariadb)
  - [Ingress & TLS](#ingress--tls)
- [Install Optional Dependencies](#install-optional-dependencies)
  - [MariaDB Operator](#mariadb-operator)
  - [LiveKit (Channel / WebRTC)](#livekit-channel--webrtc)
  - [EuroOffice (Office Document Editing)](#eurooffice-office-document-editing)
  - [Collabora (Office Document Editing — alternative)](#collabora-office-document-editing--alternative)
- [Post-Install Verification](#post-install-verification)
- [Upgrading](#upgrading)
- [Uninstalling](#uninstalling)
- [Configuration](#configuration)

---

## Prerequisites

- **Helm v3.8.0+**
- **Kubernetes 1.24+** cluster with a default StorageClass (or create PVs manually)
- **kubectl** configured to access your cluster
- **MariaDB operator** installed in the cluster (only if using `mariadbOperatorSingle.enabled=true` or `mariadbOperatorMaxscaleGalera.enabled=true`) — see [MariaDB Operator](#mariadb-operator)
- **cert-manager** with a ClusterIssuer (recommended for TLS)

> **Note:** Service addresses (Redis, MariaDB) are auto-derived from the Helm release name — `<release>-redis-master:6379` for Redis and `<release>-mariadb` for MariaDB. If you install with `--name cospace` (the default), these are `cospace-redis-master:6379` and `cospace-mariadb`. If you use a different release name, the app's env vars are auto-derived and should resolve correctly without any extra configuration.

---

## Repository

The chart is published as an OCI artifact to GitHub Container Registry:

```
oci://ghcr.io/twigex/helm-charts/cospace
```

No `helm repo add` is required for OCI charts.

---

## System Requirements

Rough sizing estimates for the full stack (cospace + MariaDB + Redis + EuroOffice + LiveKit). Actual usage depends on document size, concurrent office edits, and concurrent video streams — validate with your own load tests.

| Deployment size       | Users  | vCPU      | RAM          | Storage   | Notes                                                                                        |
|-----------------------|--------|-----------|--------------|-----------|----------------------------------------------------------------------------------------------|
| **Minimum / testing** | 1–3    | 2         | 4 GiB        | 20 GiB    | Single-node k3s. Tests, dev, demo. Office editing usable but slow under load.         |
| **Small (light prod)**| ~10    | 4–6       | 8–12 GiB     | 100 GiB   | 1–2 node cluster. Single replica of each component. EuroOffice/LiveKit single node.         |
| **Medium (prod)**     | ~100   | 8–16      | 16–32 GiB    | 500 GiB   | 3+ worker nodes. Cospace + EuroOffice + LiveKit multi-replica. MariaDB operator (single or Galera). |
| **Large (prod)**      | 500+   | 16–32+    | 32–64+ GiB   | 1+ TiB    | Galera + MaxScale. EuroOffice horizontally scaled. LiveKit multi-node with dedicated RTC.    |


> ⚠️ **The chart does NOT set CPU/memory `requests` or `limits` on any component.** The numbers above are **node sizing guides**, not what Kubernetes will reserve or throttle. For production, add explicit `resources:` blocks to `values.yaml` so the scheduler can binpack and prevent noisy-neighbor OOM kills.

---

## Install Cospace

Cospace supports four MariaDB deployment modes. Pick the one that matches your environment.

**Required values for ALL modes:**

These are required regardless of which MariaDB backend you pick:

- `domain` — FQDN for the Cospace application
- `cospace.admin.password` — initial admin password (change for production)
- `cospace.server.encryptKey` — exactly 32 hex characters. Generate with `openssl rand -hex 16`
- `redis.auth.password` — Redis password (change for production)
- `mariadb.password` — MariaDB user password (change for production). In mode 4, set `mariadb.externalSecret: true` and provide it via the `<release>-mariadb` secret instead.

**Required only for modes 1, 2, and 3** (skip in mode 4 when `mariadb.externalSecret: true`):

- `mariadb.rootPassword` — MariaDB root password (change for production)

> Generate the encrypt key once and keep it stable across upgrades — rotating it invalidates all encrypted data in the database.

> ⚠️ **The Ingress is opt-in.** By default the chart creates only the cospace Pod and Service. To expose it via your ingress controller, add `--set ingress.enabled=true` to your install command. See [Ingress & TLS](#ingress--tls) for HTTP-only, TLS, and cert-manager examples.

### 1. Default — Embedded MariaDB

A bundled MariaDB instance is deployed as a single Pod inside the chart. Simplest option, suitable for small/medium deployments.

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

### 2. With MariaDB Operator (single instance)

Uses the [mariadb-operator](https://github.com/mariadb-operator/mariadb-operator) to manage a single-replica MariaDB. Production-friendly with automated backups, point-in-time recovery, and metrics.

**Prerequisites:** [Install the MariaDB operator and CRDs first](#mariadb-operator).

```sh
helm install cospace oci://ghcr.io/twigex/helm-charts/cospace --version 0.0.34 \
  --namespace cospace --create-namespace \
  -f - <<EOF
domain: "cospace.example.com"

cospace:
  image:
    repository: ghcr.io/twigex/cospace
    tag: "latest"
  server:
    encryptKey: "a1b2c3d4e5f6a1b2c3d4e5f6a1b2c3d4"
  admin:
    password: "admin123"

mariadb:
  enabled: false
  password: "mariadb123"
  rootPassword: "root123"

mariadbOperatorSingle:
  enabled: true

redis:
  auth:
    password: "redispassword"
EOF
```

> The operator manages the MariaDB user/root credentials via the `<release>-mariadb` secret. The `mariadb.password` / `mariadb.rootPassword` values are still required because the chart's `cospace-secret` reads `mariadb.password` for `COSPACE_DB_PASSWORD`, and the operator CRs reference the same secret.

### 3. With MariaDB Operator (Galera + MaxScale)

Same as option 2 but with a 3-replica Galera cluster behind MaxScale for high availability.

**Prerequisites:** [Install the MariaDB operator and CRDs first](#mariadb-operator).

```sh
helm install cospace oci://ghcr.io/twigex/helm-charts/cospace --version 0.0.34 \
  --namespace cospace --create-namespace \
  --set domain=example.com \
  --set cospace.admin.password=your_password \
  --set cospace.server.encryptKey=$(openssl rand -hex 16) \
  --set mariadb.enabled=false \
  --set mariadb.password=your_mariadb_password \
  --set mariadb.rootPassword=your_mariadb_root_password \
  --set redis.auth.password=your_redis_password \
  --set mariadbOperatorMaxscaleGalera.enabled=true
```

Requires at least 3 worker nodes (one per Galera replica) and persistent storage on each.

### 4. Externally Managed MariaDB

Connect to any external MariaDB (on-cluster, off-cluster, or cloud-managed RDS).

Requirements:
- A Secret named `<release>-cospace` with the expected keys (see `templates/cospace-secret.yaml`)
- A Secret named `<release>-mariadb` with keys: `mariadb-password`, `mariadb-root-password`
- A Service named `<release>-mariadb` pointing to your external MariaDB on port 3306

```sh
helm install cospace oci://ghcr.io/twigex/helm-charts/cospace --version 0.0.34 \
  --namespace cospace --create-namespace \
  --set domain=example.com \
  --set cospace.admin.password=your_password \
  --set cospace.server.encryptKey=$(openssl rand -hex 16) \
  --set mariadb.enabled=false \
  --set mariadb.externalSecret=true \
  --set mariadb.password=your_mariadb_password \
  --set redis.auth.password=your_redis_password
```

### Ingress & TLS

The Ingress is **disabled by default** — the chart only creates the cospace Pod, Service, and (optionally) MariaDB / Redis. To expose the app via your ingress controller, opt in explicitly with `ingress.enabled=true`.

**HTTP-only (no TLS):**

```sh
helm install cospace oci://ghcr.io/twigex/helm-charts/cospace --version 0.0.34 \
  --namespace cospace --create-namespace \
  --set domain=example.com \
  --set cospace.admin.password=your_password \
  --set cospace.server.encryptKey=$(openssl rand -hex 16) \
  --set mariadb.password=your_mariadb_password \
  --set mariadb.rootPassword=your_mariadb_root_password \
  --set redis.auth.password=your_redis_password \
  --set ingress.enabled=true \
  --set ingress.ingressClassName=traefik   # or nginx, etc.
```

**TLS with cert-manager (recommended for production):**

Requires a `ClusterIssuer` named `letsencrypt-prod` to already exist in the cluster (create it once per cluster per the cert-manager docs). The chart only adds the annotation — cert-manager creates the Secret on first request to the Ingress.

```sh
helm install cospace oci://ghcr.io/twigex/helm-charts/cospace --version 0.0.34 \
  --namespace cospace --create-namespace \
  --set domain=example.com \
  --set cospace.admin.password=your_password \
  --set cospace.server.encryptKey=$(openssl rand -hex 16) \
  --set mariadb.password=your_mariadb_password \
  --set mariadb.rootPassword=your_mariadb_root_password \
  --set redis.auth.password=your_redis_password \
  --set ingress.enabled=true \
  --set ingress.ingressClassName=traefik \
  --set ingress.tls.enabled=true \
  --set ingress.certManagerIssuer=letsencrypt-prod
```

**TLS with a manually-created Secret:**

If you don't use cert-manager, create the Secret yourself (e.g. with `kubectl create secret tls example-com --cert=... --key=...`) and use:

```sh
  --set ingress.enabled=true \
  --set ingress.ingressClassName=traefik \
  --set ingress.tls.enabled=true
```

The chart expects a Secret named `<sanitized-domain>` (e.g. `example-com` for `domain: example.com`, or `cospace-example-com` for `domain: cospace.example.com`).

---

## Install Optional Dependencies

### MariaDB Operator

Required only if you use `mariadbOperatorSingle.enabled=true` or `mariadbOperatorMaxscaleGalera.enabled=true`. The chart does NOT install the operator — you must do it separately.

**Official installation guide:** https://github.com/mariadb-operator/mariadb-operator?tab=readme-ov-file#installation

Quick install:

```sh
# 1. Add the Helm repo
helm repo add mariadb-operator https://helm.mariadb.com/mariadb-operator
helm repo update

# 2. Install CRDs (cluster-wide, separate release)
helm install mariadb-operator-crds mariadb-operator/mariadb-operator-crds \
  --namespace mariadb-operator --create-namespace

# 3. Install the operator
helm install mariadb-operator mariadb-operator/mariadb-operator \
  --namespace mariadb-operator --version 25.10.2
```

Verify:
```sh
kubectl get pods -n mariadb-operator
kubectl get crd | grep mariadb
```

### LiveKit (Channel / WebRTC)

LiveKit powers Cospace's real-time voice/video channel. The chart only configures the Cospace app to talk to LiveKit — you must install the LiveKit server separately.

**Official documentation:** https://docs.livekit.io/home/self-hosting/helm/
**Helm chart:** https://github.com/livekit/livekit-helm

Quick install:

```sh
# 1. Add the LiveKit Helm repo
helm repo add livekit https://helm.livekit.io
helm repo update

# 2. Generate API key/secret (you'll need these for cospace config)
lk api create-key
#   → API Key:    APIxxxxxxxxxxxx
#   → API Secret: xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx

# 3. Create a values file for LiveKit
cat > livekit-values.yaml <<EOF
livekit:
  rtc:
    tcpPort: 7881
    udpPort: 7882
  turn:
    enabled: true
  redis:
    address: cospace-redis-master:6379
    password: "redispassword"
  apiKey: APIxxxxxxxxxxxx
  apiSecret: xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
EOF

# 4. Install
helm install livekit livekit/livekit-server \
  --namespace livekit --create-namespace \
  --values livekit-values.yaml
```

Configure Cospace to use it:

```yaml
cospace:
  channel:
    enabled: true
    host: "ws://livekit.livekit.svc.cluster.local:7880"  # or your public hostname
    key: "APIxxxxxxxxxxxx"
    secret: "your-api-secret"
    linkPreviews: true
```

### EuroOffice (Office Document Editing)

EuroOffice is Cospace's office document editor (alternative to Collabora).

**Official project:** https://github.com/Euro-Office
**Container image:** `ghcr.io/euro-office/documentserver:latest`

#### Install with Helm

```sh
# 1. Create a values file
cat > eurooffice-values.yaml <<EOF
image:
  repository: ghcr.io/euro-office/documentserver
  tag: latest

service:
  type: ClusterIP
  port: 80

ingress:
  enabled: true
  className: nginx
  annotations:
    nginx.ingress.kubernetes.io/proxy-body-size: "100m"
  hosts:
    - host: eurooffice.example.com
      paths:
        - path: /
  tls:
    - hosts:
        - eurooffice.example.com
      secretName: eurooffice-tls

# Allow Cospace to connect from its domain
env:
  WOPI_ENABLED: "true"
  aliasgroup1: "https://cospace.example.com"
EOF

# 2. Install using the community-maintained chart, or deploy a custom
#    Helm chart wrapping the image. If you don't have one yet, you can
#    run it as a plain Deployment:
kubectl create namespace eurooffice
kubectl create deployment eurooffice \
  --image=ghcr.io/euro-office/documentserver:latest \
  -n eurooffice
kubectl expose deployment eurooffice --port=80 --target-port=80 -n eurooffice
```

> **Note:** The `aliasgroup1` (or equivalent WOPI allow-list) env var must contain the Cospace domain so EuroOffice accepts WOPI requests from it. The Cospace domain must be reachable from end-user browsers.

#### Configure Cospace

```yaml
cospace:
  office:
    enabled: true
    type: "eurooffice"   # or "collabora" — both use the same WOPI API
    host: "https://eurooffice.example.com"
    secret: ""           # JWT secret if configured
```

### Collabora (Office Document Editing — alternative)

Collabora Online is a popular open-source office editor. Cospace supports it as an alternative to EuroOffice.

**Official documentation:** https://sdk.collaboraoffice.com/Collabora-Online-Development-Edition.html
**Helm chart:** https://github.com/CollaboraOnline/online (Docker images available on Docker Hub)

Quick install using the official Docker image:

```sh
# 1. Create a values file
cat > collabora-values.yaml <<EOF
image:
  repository: collabora/code
  tag: latest
ingress:
  enabled: true
  className: nginx
  hosts:
    - host: collabora.example.com
      paths:
        - path: /
  tls:
    - hosts:
        - collabora.example.com
      secretName: collabora-tls

env:
  aliasgroup1: "https://cospace.example.com:443"
  extra_params: "--o:ssl.enable=false --o:ssl.termination=true"
  server_name: "Collabora"
EOF

# 2. Use the Collabora chart from the community, or deploy manually:
helm repo add collabora https://collaboraonline.github.io/online/
helm install collabora collabora/collabora-online \
  --namespace collabora --create-namespace \
  --values collabora-values.yaml
```

Configure Cospace:

```yaml
cospace:
  office:
    enabled: true
    type: "collabora"
    host: "https://collabora.example.com"
    secret: ""
```

> **Note:** The `aliasgroup1` env var must contain the Cospace domain so Collabora accepts WOPI requests from it. The Cospace domain must be reachable from end-user browsers.

---

## Post-Install Verification

After installation, verify that everything is healthy:

```sh
# Check pods
kubectl get pods -n cospace

# Check services
kubectl get svc -n cospace

# Tail application logs.
# The cospace pod label is `app: <sanitized-domain>`, where <sanitized-domain> is your
# `domain` value with dots replaced by dashes (e.g. domain: cospace.example.com → app=cospace-example-com).
# To find the exact label value, run: kubectl get pods -n cospace --show-labels
kubectl logs -n cospace -l app=<sanitized-domain> -f
# Example: kubectl logs -n cospace -l app=example-com -f
# Or, more robustly, by pod name:
#   POD=$(kubectl get pod -n cospace -l app=<sanitized-domain> -o jsonpath='{.items[0].metadata.name}')
#   kubectl logs -n cospace "$POD" -f

# Check the MariaDB CR (operator mode only)
kubectl get mariadb -n cospace
```

Expected healthy state:
- `<sanitized-domain>-...` pod: `Running`, logs show the app is listening on the configured port (`COSPACE_SERVER_PORT`, default 3000)
- `<release>-mariadb-0` (embedded Bitnami or operator single/galera): `Running`
- `<release>-redis-master-0`: `Running`

---

## Upgrading

```sh
helm upgrade cospace oci://ghcr.io/twigex/helm-charts/cospace --version 0.0.34 \
  --namespace cospace \
  -f your-values.yaml
```

⚠️ **Breaking changes** — read the [Changelog](README.md#changelog) in the README before upgrading, especially around the default database backend.

---

## Uninstalling

```sh
helm uninstall cospace --namespace cospace
kubectl delete namespace cospace
```

> **Note:** PVCs (`cospace-pvc`, `mariadb-pvc`) and Secrets (`<release>-cospace`, `<release>-mariadb`) have `helm.sh/resource-policy: keep` and are NOT deleted by `helm uninstall`. Delete them manually if you want a full cleanup.

For a complete wipe:

```sh
helm uninstall cospace -n cospace
kubectl delete pvc -n cospace --all
kubectl delete secret -n cospace -l app.kubernetes.io/instance=cospace
kubectl delete namespace cospace
```

---

## Configuration

Reference of all values exposed by `values.yaml`. See [Install Cospace](#install-cospace) for the install commands; this section documents what each value does and its default.

### Global

| Name | Description | Type | Default |
|------|-------------|------|---------|
| `domain` | FQDN for the Cospace application | string | `"example.com"` |
| `fullnameOverride` | Full resource name override | string | `""` |
| `nameOverride` | Name portion override | string | `""` |

### Ingress

| Name | Description | Type | Default |
|------|-------------|------|---------|
| `ingress.enabled` | Enable Ingress resource (opt-in; chart creates only the Pod + Service by default) | bool | `false` |
| `ingress.ingressClassName` | IngressClass (e.g. `nginx`, `traefik`) | string | `"nginx"` |
| `ingress.annotations` | Default Ingress annotations | object | See values.yaml |
| `ingress.customAnnotations` | Additional/overriding annotations | object | `{}` |
| `ingress.tls.enabled` | Render the TLS block in the Ingress. When false, the Ingress is HTTP-only. | bool | `false` |
| `ingress.certManagerIssuer` | Name of an existing cert-manager ClusterIssuer. When set, the chart adds the `cert-manager.io/cluster-issuer` annotation so cert-manager creates the TLS Secret named `<sanitized-domain>`. | string | `""` |

### Cospace Image

| Name | Description | Type | Default |
|------|-------------|------|---------|
| `cospace.image.repository` | Container image repository | string | `ghcr.io/twigex/cospace` |
| `cospace.image.tag` | Container image tag | string | `"latest"` |

### Server

| Name | Description | Type | Default |
|------|-------------|------|---------|
| `cospace.server.port` | Server listen port (used for the container, Service, and Ingress backend) | int | `3000` |
| `cospace.server.defaultLocale` | Default UI locale | string | `"en"` |
| `cospace.server.sessionLengthDays` | Session lifetime in days | int | `30` |
| `cospace.server.encryptKey` | **Exactly 32 hex characters (16 bytes).** Generate with `openssl rand -hex 16`. The chart fails the install at template-render time if this is missing, the wrong length, or contains non-hex characters. ⚠️ Change for production. | string | `"a1b2c3d4e5f6a1b2c3d4e5f6a1b2c3d4"` |
| `cospace.server.tlsCertFile` | Path to TLS cert file (optional) | string | `""` |
| `cospace.server.tlsKeyFile` | Path to TLS key file (optional) | string | `""` |

### Database

| Name | Description | Type | Default |
|------|-------------|------|---------|
| `cospace.db.host` | **DEPRECATED: ignored by the chart.** The MariaDB host is auto-derived from the release name (`<release>-mariadb`). Left in `values.yaml` to avoid breaking existing values files. | string | `"127.0.0.1"` |
| `cospace.db.port` | MariaDB port | int | `3306` |
| `cospace.db.user` | Database username | string | `"cospace"` |
| `cospace.db.name` | Database name | string | `"cospace"` |
| `cospace.db.driver` | Database driver | string | `"mysql"` |
| `cospace.db.readTimeout` | Read timeout | string | `"30s"` |
| `cospace.db.writeTimeout` | Write timeout | string | `"30s"` |
| `cospace.db.maxOpenConns` | Max open connections | int | `25` |
| `cospace.db.maxIdleConns` | Max idle connections | int | `10` |
| `cospace.db.connMaxLifetimeMinutes` | Connection max lifetime (minutes) | int | `60` |
| `cospace.db.connMaxIdleTimeMinutes` | Connection max idle time (minutes) | int | `30` |
| `cospace.db.queryTimeout` | Query timeout (seconds) | int | `30` |

> **Note:** The database password is controlled by `mariadb.password` (see [MariaDB Configuration](#mariadb-configuration)). Both the MariaDB instance and the Cospace app use the same password via a single source of truth.

### Redis

| Name | Description | Type | Default |
|------|-------------|------|---------|
| `cospace.redis.address` | **DEPRECATED: ignored by the chart.** The Redis address is auto-derived from the release name (`<release>-redis-master:6379`). Left in `values.yaml` to avoid breaking existing values files. | string | `"cospace-redis-master:6379"` |
| `cospace.redis.password` | **DEPRECATED: ignored by the chart.** The Cospace app's Redis password is now sourced from `redis.auth.password` so the two values stay in sync. Setting this has no effect. | string | `"redispassword"` |

### Storage

| Name | Description | Type | Default |
|------|-------------|------|---------|
| `cospace.storage.type` | Storage type: `local` or `s3` | string | `"local"` |
| `cospace.storage.name` | Human-readable storage name | string | `"Local storage"` |
| `cospace.storage.path` | Local storage path | string | `"data/files"` |
| `cospace.storage.endpoint` | S3 endpoint (required if type=s3) | string | `""` |
| `cospace.storage.accessKey` | S3 access key (required if type=s3) | string | `""` |
| `cospace.storage.secretKey` | S3 secret key (required if type=s3) | string | `""` |
| `cospace.storage.bucket` | S3 bucket name | string | `""` |
| `cospace.storage.ssl` | Use SSL for S3 | bool | `true` |

### SMTP

| Name | Description | Type | Default |
|------|-------------|------|---------|
| `cospace.smtp.host` | SMTP server host | string | `"smtp.example.com"` |
| `cospace.smtp.port` | SMTP server port | int | `1025` |
| `cospace.smtp.username` | SMTP username | string | `""` |
| `cospace.smtp.password` | SMTP password | string | `""` |
| `cospace.smtp.security` | SMTP security (e.g. `tls`, `starttls`) | string | `""` |
| `cospace.smtp.authEnabled` | Enable SMTP authentication | bool | `false` |

### Office (Collabora or EuroOffice)

| Name | Description | Type | Default |
|------|-------------|------|---------|
| `cospace.office.enabled` | Enable office document editing. Pick the editor via `cospace.office.type`. | bool | `false` |
| `cospace.office.type` | Office editor type. Valid values: `collabora` or `eurooffice`. Both expose the WOPI API. | string | `""` |
| `cospace.office.host` | Office editor URL. Required when `office.enabled=true`. | string | `"http://localhost:9980"` |
| `cospace.office.secret` | JWT secret if configured on the office editor. | string | `""` |

### Channel (LiveKit / WebRTC)

| Name | Description | Type | Default |
|------|-------------|------|---------|
| `cospace.channel.enabled` | Enable Channel (LiveKit) | bool | `false` |
| `cospace.channel.host` | LiveKit WebSocket URL. Required when `channel.enabled=true`. | string | `"ws://localhost:7880"` |
| `cospace.channel.key` | LiveKit API key | string | `""` |
| `cospace.channel.secret` | LiveKit API secret | string | `""` |
| `cospace.channel.linkPreviews` | Enable link previews in chat | bool | `true` |
| `cospace.channel.gif.enabled` | Enable GIF picker | bool | `false` |
| `cospace.channel.gif.apiKey` | GIF provider API key | string | `""` |
| `cospace.channel.gif.clientKey` | GIF provider client key | string | `""` |

### Gitea OAuth

| Name | Description | Type | Default |
|------|-------------|------|---------|
| `cospace.gitea.enabled` | Enable Gitea OAuth login | bool | `false` |
| `cospace.gitea.clientID` | Gitea OAuth client ID | string | `""` |
| `cospace.gitea.clientSecret` | Gitea OAuth client secret | string | `""` |
| `cospace.gitea.authEndpoint` | Gitea authorization endpoint | string | `"https://gitea.example.com/login/oauth/authorize"` |
| `cospace.gitea.tokenEndpoint` | Gitea token endpoint | string | `"https://gitea.example.com/login/oauth/access_token"` |
| `cospace.gitea.userAPIEndpoint` | Gitea user API endpoint | string | `"https://gitea.example.com/api/v1/user"` |
| `cospace.gitea.redirectURL` | OAuth redirect URL. Should match your domain when `gitea.enabled=true`. | string | `"http://localhost:3000/callback/gitea"` |
| `cospace.gitea.scopes` | OAuth scopes | string | `"user:email"` |

### LDAP

| Name | Description | Type | Default |
|------|-------------|------|---------|
| `cospace.ldap.enabled` | Enable LDAP authentication | bool | `false` |
| `cospace.ldap.serverURL` | LDAP server URL | string | `"ldap://ldap.example.com:389"` |
| `cospace.ldap.baseDN` | LDAP base DN | string | `"dc=example,dc=com"` |
| `cospace.ldap.bindDN` | LDAP bind DN | string | `"cn=admin,dc=example,dc=com"` |
| `cospace.ldap.bindPassword` | LDAP bind password | string | `""` |
| `cospace.ldap.insecureSkipVerify` | Skip TLS verification | bool | `false` |
| `cospace.ldap.userFilter` | LDAP user filter | string | `"(uid={0})"` |
| `cospace.ldap.userIDAttr` | User ID attribute | string | `"uid"` |
| `cospace.ldap.firstNameAttr` | First name attribute | string | `"givenName"` |
| `cospace.ldap.lastNameAttr` | Last name attribute | string | `"sn"` |
| `cospace.ldap.emailAttr` | Email attribute | string | `"mail"` |
| `cospace.ldap.allowedGroups` | Comma-separated allowed LDAP groups | string | `""` |

### Admin (First Boot)

The admin user is created only on first boot when no users exist in the database.

| Name | Description | Type | Default |
|------|-------------|------|---------|
| `cospace.admin.name` | Admin first name | string | `"Admin"` |
| `cospace.admin.lastName` | Admin last name | string | `"User"` |
| `cospace.admin.username` | Admin username | string | `"admin"` |
| `cospace.admin.email` | Admin email | string | `"admin@example.com"` |
| `cospace.admin.password` | Admin password. ⚠️ Change for production. | string | `"admin"` |

### License

| Name | Description | Type | Default |
|------|-------------|------|---------|
| `cospace.license` | License key | string | `""` |

### Persistence

| Name | Description | Type | Default |
|------|-------------|------|---------|
| `cospace.persistence.size` | PVC size | string | `"2Gi"` |
| `cospace.persistence.storageClass` | StorageClass (`""` = cluster default) | string | `""` |
| `cospace.persistence.accessModes` | PVC access modes | list | `[ReadWriteOnce]` |
| `cospace.persistence.annotations` | Additional PVC annotations | object | `{}` |
| `cospace.persistence.labels` | Additional PVC labels | object | `{}` |

### Redis Subchart

| Name | Description | Type | Default |
|------|-------------|------|---------|
| `redis.architecture` | Redis architecture: `standalone` or `replication` | string | `"standalone"` |
| `redis.auth.enabled` | Enable Redis authentication | bool | `true` |
| `redis.auth.password` | Redis password. **Used by both the Bitnami Redis subchart and the Cospace app** (via the chart's `cospace-secret`). This is the single source of truth for the Redis password — set this, not `cospace.redis.password`. | string | `"supersecretpassword"` |
| `redis.master.persistence.enabled` | Enable Redis persistence | bool | `true` |
| `redis.master.persistence.size` | Redis PVC size | string | `"1Gi"` |

### MariaDB Configuration

| Name | Description | Type | Default |
|------|-------------|------|---------|
| `mariadb.enabled` | Enable embedded MariaDB | bool | `true` |
| `mariadb.image.repository` | MariaDB image repository | string | `"mariadb"` |
| `mariadb.image.tag` | MariaDB image tag | string | `"latest"` |
| `mariadb.password` | MariaDB user password. Also used by the cospace app to connect. ⚠️ Change for production. **Do NOT change after initial install** with embedded MariaDB — the password is set in the database on first boot and cannot be changed without reinitializing (data loss). | string | `"mariadb"` |
| `mariadb.rootPassword` | MariaDB root password. ⚠️ Change for production. | string | `"mariadbroot"` |
| `mariadb.externalSecret` | Set to `true` to skip creating the chart-managed mariadb-secret. You must provide a secret named `<release>-mariadb` with keys: `mariadb-password`, `mariadb-root-password`. | bool | `false` |
| `mariadbOperatorSingle.enabled` | Deploy single MariaDB via operator. Requires the MariaDB operator and CRDs to be pre-installed in the cluster. | bool | `false` |
| `mariadbOperatorMaxscaleGalera.enabled` | Deploy Galera + MaxScale via operator. Requires the MariaDB operator and CRDs to be pre-installed in the cluster. | bool | `false` |

### External Secret Management

For production deployments using external secret management (e.g. [External Secrets Operator](https://external-secrets.io/), Vault, AWS Secrets Manager), you have two options:

**Option A — Let the chart create secrets, populate from external source:**

Set the password values in your ArgoCD Application or values file. The chart creates the secrets with those values. Your external secret manager syncs to the same secret names.

**Option B — Fully external secrets:**

Set `cospace.externalSecret: true` and/or `mariadb.externalSecret: true`. The chart will **not** create the corresponding secret. You must provide secrets with the expected names and keys:

- `<release>-cospace` — keys: `COSPACE_ADMIN_PASSWORD`, `COSPACE_SERVER_ENCRYPT_KEY`, `COSPACE_DB_PASSWORD`, `COSPACE_REDIS_PASSWORD`, `COSPACE_STORAGE_ACCESS_KEY`, `COSPACE_STORAGE_SECRET_KEY`, `COSPACE_SMTP_PASSWORD`, `COSPACE_OFFICE_SECRET`, `COSPACE_CHANNEL_SECRET`, `COSPACE_GITEA_CLIENT_SECRET`, `COSPACE_LDAP_BIND_PASSWORD`
- `<release>-mariadb` — keys: `mariadb-password`, `mariadb-root-password`
