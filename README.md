# kube-infra

Shared infrastructure Helm charts for the astra.home cluster.

| Chart | Helm release | Namespace | URL |
|-------|-------------|-----------|-----|
| `helm/charts/postgres` | `shared-postgres` | `shared-data` | — |
| `helm/charts/keycloak` | `keycloak` | `security` | `https://auth.astra.home` |
| `helm/charts/redis` | `shared-redis` | `shared-data` | — |
| `helm/charts/minio` | `shared-minio` | `shared-data` | `https://minio.astra.home` |
| `helm/charts/redisinsight` | `shared-redisinsight` | `shared-data` | `https://redisinsight.astra.home` |
| `helm/charts/langfuse` | `langfuse` | `langfuse` | `https://langfuse.astra.home` |

---

## Prerequisites

Namespaces must exist and be enrolled in Istio ambient mode:

```bash
kubectl create namespace shared-data
kubectl create namespace security
kubectl create namespace langfuse
kubectl label namespace shared-data istio.io/dataplane-mode=ambient
kubectl label namespace security istio.io/dataplane-mode=ambient
kubectl label namespace langfuse istio.io/dataplane-mode=ambient
```

## values.local.yaml

Each chart that handles secrets requires a `values.local.yaml` alongside its `values.yaml`.
These files are gitignored — create them locally before deploying, never commit them.

`helm/charts/postgres/values.local.yaml`:
```yaml
auth:
  password: "<postgres admin password>"
  postgresPassword: "<postgres admin password>"
```

`helm/charts/keycloak/values.local.yaml`:
```yaml
adminPassword: "<keycloak admin password>"
```

`helm/charts/redis/values.local.yaml`:
```yaml
auth:
  password: "<redis password>"
```

`helm/charts/minio/values.local.yaml`:
```yaml
auth:
  rootPassword: "<minio root password>"
```

`helm/charts/langfuse/values.local.yaml`:
```yaml
postgres:
  adminPassword: "<postgres admin password>"

redis:
  password: "<redis password>"

minio:
  rootPassword: "<minio root password>"

langfuse:
  langfuse:
    salt:
      value: "<openssl rand -hex 32>"
    nextauth:
      secret:
        value: "<openssl rand -hex 32>"
    encryptionKey:
      value: "<openssl rand -hex 32>"
  clickhouse:
    auth:
      password: "<openssl rand -hex 16>"
```

---

## Deploy PostgreSQL

```bash
helm install shared-postgres helm/charts/postgres \
  -n shared-data \
  -f helm/charts/postgres/values.local.yaml

helm upgrade shared-postgres helm/charts/postgres \
  -n shared-data \
  -f helm/charts/postgres/values.local.yaml
```

## Deploy Redis

```bash
helm install shared-redis helm/charts/redis \
  -n shared-data \
  -f helm/charts/redis/values.local.yaml

helm upgrade shared-redis helm/charts/redis \
  -n shared-data \
  -f helm/charts/redis/values.local.yaml
```

## Deploy MinIO

MinIO uses NFS to mount `/data2/minio` from the host (`192.168.122.1`).
The `langfuse` bucket must be created after the first install:

```bash
helm install shared-minio helm/charts/minio \
  -n shared-data \
  -f helm/charts/minio/values.local.yaml

# Create the langfuse bucket (one-time)
kubectl exec -n shared-data shared-minio-minio-0 -- \
  mc alias set local http://localhost:9000 admin <rootPassword>
kubectl exec -n shared-data shared-minio-minio-0 -- \
  mc mb local/langfuse

helm upgrade shared-minio helm/charts/minio \
  -n shared-data \
  -f helm/charts/minio/values.local.yaml
```

Console: `https://minio.astra.home` — user `admin`.

## Deploy RedisInsight

```bash
helm install shared-redisinsight helm/charts/redisinsight -n shared-data

helm upgrade shared-redisinsight helm/charts/redisinsight -n shared-data
```

UI: `https://redisinsight.astra.home`

Add a database connection in the UI:
- **Host**: `shared-redis-redis.shared-data.svc.cluster.local`
- **Port**: `6379`
- **Username**: `default`
- **Password**: value from `helm/charts/redis/values.local.yaml`

## Deploy Keycloak

```bash
helm install keycloak helm/charts/keycloak \
  -n security \
  -f helm/charts/keycloak/values.local.yaml

helm upgrade keycloak helm/charts/keycloak \
  -n security \
  -f helm/charts/keycloak/values.local.yaml
```

UI: `https://auth.astra.home`

## Deploy Langfuse

```bash
# First time only
helm dependency update helm/charts/langfuse

helm install langfuse helm/charts/langfuse \
  -n langfuse \
  -f helm/charts/langfuse/values.local.yaml

helm upgrade langfuse helm/charts/langfuse \
  -n langfuse \
  -f helm/charts/langfuse/values.local.yaml
```

UI: `https://langfuse.astra.home`

> Note: ClickHouse and ZooKeeper pods are opted out of Istio ambient
> (`istio.io/dataplane-mode: none`) due to a K3s + Istio 1.29 HBONE routing
> issue with same-node pod traffic. This is patched directly on the StatefulSets
> and will be lost on a full `helm upgrade` — re-apply with:
> ```bash
> kubectl patch statefulset langfuse-clickhouse-shard0 -n langfuse --type=json \
>   -p='[{"op":"add","path":"/spec/template/metadata/labels/istio.io~1dataplane-mode","value":"none"}]'
> kubectl patch statefulset langfuse-zookeeper -n langfuse --type=json \
>   -p='[{"op":"add","path":"/spec/template/metadata/labels/istio.io~1dataplane-mode","value":"none"}]'
> ```
