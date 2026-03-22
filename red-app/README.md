# RED App Helm Chart

Helm chart for deploying the RED Risk Management Platform on Kubernetes.

## Prerequisites

- Kubernetes 1.24+
- Helm 3.x
- A valid RED license key

## Quick Start

```bash
# Install from public OCI registry (no authentication required)
helm install red-app oci://public.ecr.aws/t3v3f1b2/red-app \
  --version 0.1.0 \
  -f values.yaml \
  -n red-system \
  --create-namespace
```

## Installing from Local Chart

```bash
# Build dependencies (downloads PostgreSQL subchart)
helm dependency build .

# Install
helm install red-app . -f my-values.yaml -n red-system --create-namespace
```

## Configuration

Create a `values.yaml` file with your configuration:

```yaml
image:
  tag: "v1.0.0"

env:
  DATABASE_URL: postgresql://red_user:YOUR_PASSWORD@red-app-postgresql:5432/red_production
  LICENSE_SERVER_URL: https://backoffice.redrisk.eu
  LICENSE_KEY: YOUR-LICENSE-KEY
  SESSION_SECRET: your-session-secret
  JWT_SECRET: your-jwt-secret
  ALLOWED_ORIGINS: https://red.your-domain.com
  APP_URL: https://red.your-domain.com

ingress:
  enabled: true
  className: nginx
  hosts:
    - host: red.your-domain.com
      paths:
        - path: /
          pathType: Prefix
  tls:
    - secretName: red-tls
      hosts:
        - red.your-domain.com

postgresql:
  auth:
    password: YOUR_DB_PASSWORD
    postgresPassword: YOUR_POSTGRES_PASSWORD
```

## Using Kubernetes Secrets

For production, use an existing Secret instead of plaintext values:

```bash
kubectl create secret generic red-app-secrets -n red-system \
  --from-literal=DATABASE_URL='postgresql://red_user:pass@red-app-postgresql:5432/red_production' \
  --from-literal=SESSION_SECRET='your-session-secret' \
  --from-literal=JWT_SECRET='your-jwt-secret' \
  --from-literal=LICENSE_KEY='YOUR-LICENSE-KEY'
```

Then reference it in `values.yaml`:

```yaml
existingSecret: red-app-secrets
```

When `existingSecret` is set, the chart reads `DATABASE_URL`, `SESSION_SECRET`, `JWT_SECRET`, and `LICENSE_KEY` from the Secret instead of `env.*`.

## Parameters

### Application

| Parameter | Description | Default |
|---|---|---|
| `replicaCount` | Number of replicas | `1` |
| `image.repository` | Image repository | `public.ecr.aws/t3v3f1b2/red-app` |
| `image.tag` | Image tag | `v1.0.0` |
| `image.pullPolicy` | Image pull policy | `IfNotPresent` |
| `imagePullSecrets` | Image pull secrets | `[]` |

### Required Environment Variables

| Parameter | Description | Default |
|---|---|---|
| `env.DATABASE_URL` | PostgreSQL connection string | `""` |
| `env.LICENSE_SERVER_URL` | License server URL | `""` |
| `env.LICENSE_KEY` | License key | `""` |
| `env.SESSION_SECRET` | Session encryption secret | `""` |
| `env.JWT_SECRET` | JWT signing secret | `""` |
| `env.ALLOWED_ORIGINS` | CORS allowed origins | `""` |
| `env.APP_URL` | Public application URL | `""` |
| `env.NODE_ENV` | Node environment | `production` |
| `env.PORT` | Application port | `5000` |

### Optional Environment Variables

Set via `optionalEnv` map:

| Key | Description |
|---|---|
| `RATE_LIMIT_MAX` | Max requests per rate limit window |
| `SMTP_HOST` | SMTP server hostname |
| `SMTP_PORT` | SMTP server port |
| `SMTP_USER` | SMTP username |
| `SMTP_PASSWORD_BASE64` | SMTP password (base64 encoded) |
| `SMTP_FROM` | Sender email address |

Example:

```yaml
optionalEnv:
  SMTP_HOST: smtp.example.com
  SMTP_PORT: "465"
  SMTP_USER: noreply@example.com
  SMTP_PASSWORD_BASE64: cGFzc3dvcmQ=
  SMTP_FROM: noreply@example.com
```

### Service & Ingress

| Parameter | Description | Default |
|---|---|---|
| `service.type` | Service type | `ClusterIP` |
| `service.port` | Service port | `5000` |
| `ingress.enabled` | Enable ingress | `false` |
| `ingress.className` | Ingress class | `nginx` |
| `ingress.hosts` | Ingress hosts | see `values.yaml` |
| `ingress.tls` | Ingress TLS config | `[]` |

### Persistence

| Parameter | Description | Default |
|---|---|---|
| `persistence.reports.enabled` | Enable PVC for generated reports | `true` |
| `persistence.reports.size` | Reports volume size | `2Gi` |
| `persistence.reports.storageClass` | Storage class (empty = default) | `""` |

### Autoscaling

| Parameter | Description | Default |
|---|---|---|
| `autoscaling.enabled` | Enable HPA | `false` |
| `autoscaling.minReplicas` | Minimum replicas | `1` |
| `autoscaling.maxReplicas` | Maximum replicas | `3` |
| `autoscaling.targetCPUUtilizationPercentage` | Target CPU utilization | `80` |

### PostgreSQL (Bitnami Subchart)

| Parameter | Description | Default |
|---|---|---|
| `postgresql.enabled` | Deploy bundled PostgreSQL | `true` |
| `postgresql.auth.database` | Database name | `red_production` |
| `postgresql.auth.username` | Database user | `red_user` |
| `postgresql.auth.password` | Database password | `""` |
| `postgresql.primary.persistence.size` | Database volume size | `10Gi` |

Set `postgresql.enabled: false` to use an external database. When disabled, provide the full connection string via `env.DATABASE_URL`.

For all PostgreSQL options, see the [Bitnami PostgreSQL chart documentation](https://github.com/bitnami/charts/tree/main/bitnami/postgresql).

## Upgrading

```bash
helm upgrade red-app oci://public.ecr.aws/t3v3f1b2/red-app \
  --version 0.2.0 \
  -f values.yaml \
  -n red-system
```

## Uninstalling

```bash
helm uninstall red-app -n red-system
```

Note: PVCs are not deleted automatically. To remove all data:

```bash
kubectl delete pvc -n red-system -l app.kubernetes.io/instance=red-app
```

## Health Checks

The chart configures liveness and readiness probes on `/api/health`:

- **Readiness**: starts after 20s, checks every 10s
- **Liveness**: starts after 40s, checks every 30s

## Architecture

```
                    ┌─────────┐
                    │ Ingress │
                    └────┬────┘
                         │
                    ┌────▼────┐
                    │ Service │
                    │  :5000  │
                    └────┬────┘
                         │
               ┌─────────▼─────────┐
               │    RED App Pod     │
               │  (Node.js/Express) │
               └─────────┬─────────┘
                         │
               ┌─────────▼─────────┐
               │    PostgreSQL      │
               │  (Bitnami subchart)│
               └───────────────────┘
```
