# kube-infra

Shared infrastructure Helm charts for the astra.home cluster.

| Chart | Helm release | Namespace |
|-------|-------------|-----------|
| `helm/charts/postgres` | `shared-postgres` | `shared-data` |
| `helm/charts/keycloak` | `keycloak` | `security` |

---

## Prerequisites

Both namespaces must exist and be enrolled in Istio ambient mode:

```bash
kubectl create namespace shared-data
kubectl create namespace security
kubectl label namespace shared-data istio.io/dataplane-mode=ambient
kubectl label namespace security istio.io/dataplane-mode=ambient
```

## Secrets

Create these secrets before the first `helm install`. Use the same password for all three.

```bash
kubectl create secret generic shared-postgres-postgresql \
  -n shared-data --from-literal=password=<PASSWORD>

kubectl create secret generic db-credentials \
  -n personal-finance --from-literal=password=<PASSWORD>

kubectl create secret generic db-credentials \
  -n security --from-literal=password=<PASSWORD>
```

## values.local.yaml

Each chart requires a `values.local.yaml` alongside its `values.yaml`. These files are
gitignored — create them locally before deploying, never commit them.

`helm/charts/postgres/values.local.yaml`:
```yaml
auth:
  password: "<production password>"
  postgresPassword: "<production password>"
```

`helm/charts/keycloak/values.local.yaml`:
```yaml
adminPassword: "<keycloak admin password>"
```

## Deploy PostgreSQL

```bash
# Fresh install
helm install shared-postgres helm/charts/postgres \
  -n shared-data \
  -f helm/charts/postgres/values.local.yaml

# Upgrade
helm upgrade shared-postgres helm/charts/postgres \
  -n shared-data \
  -f helm/charts/postgres/values.local.yaml
```

## Deploy Keycloak

```bash
# Fresh install
helm install keycloak helm/charts/keycloak \
  -n security \
  -f helm/charts/keycloak/values.local.yaml

# Upgrade
helm upgrade keycloak helm/charts/keycloak \
  -n security \
  -f helm/charts/keycloak/values.local.yaml
```

## Keycloak external URL

`https://auth.astra.home` — covered by the cluster wildcard `*.astra.home` certificate.
