# Challenge 03 — App Deployment & Gateway API — Coach Solution

[< Previous Solution](./Solution-02.md) | [Home](../../README.md) | [Next Solution >](./Solution-04.md)

## Notes & Guidance

- Teams must complete the Helm chart and get the app running before choosing a routing option.
- Option A (Gateway API via App Routing) is the recommended first path — same add-on, no extra infra.
- Option B (AGC) requires a Managed Identity and ALB Controller install — allow 20–30 extra minutes.
- For the database, use **in-cluster PostgreSQL** via the Bitnami Helm chart — no Azure resource provisioning needed, and the connection string stays cluster-internal.

### Common Issues

- **App Routing already enabled:** AKS Automatic clusters may have it on by default.
  Check: `kubectl get gatewayclass` before enabling it again.
- **GatewayClass not found:** Run `az aks update --resource-group $RG --name $CLUSTER_NAME --enable-gateway-api --enable-app-routing-istio` and wait 2–3 minutes for the
  GatewayClass to be registered.
- **Helm chart rendering errors:** Run `helm template ./chart` to debug before installing.
- **Gateway API — route not accepted:** Confirm the `parentRef` gateway name matches exactly
  and the `allowedRoutes.namespaces` selector includes the `fabtech` namespace.
- **AGC — frontend not resolving:** ALB Controller pods must be running in `azure-alb-system`
  before the `ApplicationLoadBalancer` resource is created.

## Solution

### Part 0 — Deploy In-Cluster PostgreSQL

Deploy PostgreSQL into the `fabtech` namespace using the Bitnami Helm chart. This keeps setup
fast and self-contained — no Azure resource provisioning required.

```bash
DB_PASS=<choose-a-password>
NAMESPACE=fabtech

helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update

helm upgrade --install fabtech-pg bitnami/postgresql \
  --namespace $NAMESPACE \
  --create-namespace \
  --set auth.database=fabtech \
  --set auth.username=fabadmin \
  --set auth.password=$DB_PASS

# Verify the pod is running
kubectl get pods -n $NAMESPACE -l app.kubernetes.io/name=postgresql

# Connection string (used in Challenge 04)
echo "postgresql://fabadmin:${DB_PASS}@fabtech-pg-postgresql.${NAMESPACE}.svc.cluster.local:5432/fabtech"
```

> **Note:** The API falls back to serving data from bundled JSON files when `DATABASE_URL` is not
> set. The connection string will be stored in Key Vault and injected via CSI driver in Challenge 04.

### Part 1 — Enable App Routing & Deploy the Helm Chart

```bash
RG=rg-frontier-aks
CLUSTER_NAME=aks-frontier
ACR_NAME=<ACR_NAME>
NAMESPACE=fabtech

# On AKS Automatic, App Routing may already be enabled.
# Check for the GatewayClass first:
kubectl get gatewayclass

# Enable App Routing if the GatewayClass is not present
az aks update \
  --resource-group $RG \
  --name $CLUSTER_NAME \
  --enable-gateway-api \
  --enable-app-routing-istio

# Verify Gateway API support is ready before applying gateway.yaml
kubectl get gatewayclass
# Expected: approuting-istio

ACR_LOGIN_SERVER=$(az acr show --name $ACR_NAME --query loginServer -o tsv)
CHART_PATH="./Student/Resources/src/manifests/chart"

helm upgrade --install fabtech $CHART_PATH \
  --namespace $NAMESPACE \
  --create-namespace \
  --set api.image.repository=$ACR_LOGIN_SERVER/fabtech-api \
  --set web.image.repository=$ACR_LOGIN_SERVER/fabtech-web

kubectl get pods,svc -n $NAMESPACE
```

### Part 2 — Option A: Gateway API via App Routing

> **Note:** `az aks update --enable-app-routing-istio` installs the Gateway API CRDs
> and registers the `approuting-istio` `GatewayClass`. No manual CRD install
> is needed, but allow a couple of minutes for the `GatewayClass` to appear.

```yaml
# gateway.yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: fabtech-gateway
  namespace: fabtech
  annotations:
    service.beta.kubernetes.io/azure-load-balancer-internal: "false"
spec:
  gatewayClassName: approuting-istio
  listeners:
  - name: http
    protocol: HTTP
    port: 80
    allowedRoutes:
      namespaces:
        from: Same
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: fabtech-route
  namespace: fabtech
spec:
  parentRefs:
  - name: fabtech-gateway
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /
    backendRefs:
    - name: fabtech-web
      port: 3000
```

```bash
kubectl apply -f gateway.yaml
kubectl get gateway -n fabtech          # wait for READY=True
kubectl get httproute -n fabtech        # confirm status: Accepted
```

### Part 3 — Option B: Gateway API via Application Gateway for Containers (AGC)

```bash
# Install the ALB Controller via Helm with Workload Identity
IDENTITY_NAME=alb-controller-identity
CLUSTER_RG=$(az aks show -g $RG -n $CLUSTER_NAME \
  --query nodeResourceGroup -o tsv)

az identity create -g $RG -n $IDENTITY_NAME
IDENTITY_CLIENT_ID=$(az identity show -g $RG -n $IDENTITY_NAME \
  --query clientId -o tsv)

# Assign AppGw for Containers Configuration Manager role
az role assignment create \
  --assignee-object-id $(az identity show -g $RG -n $IDENTITY_NAME \
    --query principalId -o tsv) \
  --role "AppGw for Containers Configuration Manager" \
  --scope $(az group show -g $RG --query id -o tsv)

# Federate the identity
OIDC_ISSUER=$(az aks show -g $RG -n $CLUSTER_NAME \
  --query oidcIssuerProfile.issuerUrl -o tsv)

az identity federated-credential create \
  --name alb-controller \
  --identity-name $IDENTITY_NAME \
  --resource-group $RG \
  --issuer $OIDC_ISSUER \
  --subject "system:serviceaccount:azure-alb-system:alb-controller-sa"

# Install ALB Controller
helm install alb-controller oci://mcr.microsoft.com/application-lb/charts/alb-controller \
  --namespace azure-alb-system \
  --create-namespace \
  --set albController.namespace=azure-alb-system \
  --set albController.podIdentity.clientID=$IDENTITY_CLIENT_ID

kubectl wait --namespace azure-alb-system \
  --for=condition=ready pod \
  --selector=app=alb-controller \
  --timeout=90s
```

AGC bring-your-own mode needs three resources:

1. `ApplicationLoadBalancer`
2. `Gateway` bound to that ALB
3. `HTTPRoute` that points to the `Gateway`

```yaml
# agc.yaml — ApplicationLoadBalancer + Gateway + HTTPRoute
apiVersion: alb.networking.azure.io/v1
kind: ApplicationLoadBalancer
metadata:
  name: fabtech-alb
  namespace: azure-alb-system
spec:
  associations:
  - /subscriptions/<SUB_ID>/resourceGroups/<NODE_RG>/providers/Microsoft.Network/virtualNetworks/<VNET>/subnets/<AGC_SUBNET>
---
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: fabtech-agc-gateway
  namespace: fabtech
  annotations:
    alb.networking.azure.io/alb-namespace: azure-alb-system
    alb.networking.azure.io/alb-name: fabtech-alb
spec:
  gatewayClassName: azure-alb-external
  listeners:
  - name: http
    port: 80
    protocol: HTTP
    allowedRoutes:
      namespaces:
        from: Same
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: fabtech-agc-route
  namespace: fabtech
spec:
  parentRefs:
  - name: fabtech-agc-gateway
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /
    backendRefs:
    - name: fabtech-web
      port: 3000
```

### Helm Upgrade & Rollback

```bash
helm upgrade fabtech ./chart \
  --namespace $NAMESPACE \
  --set api.replicaCount=4

kubectl rollout status deployment/fabtech-api -n $NAMESPACE

helm rollback fabtech --namespace $NAMESPACE
helm history fabtech --namespace $NAMESPACE
```

### Production Readiness: PodDisruptionBudget

A `PodDisruptionBudget` ensures at least one replica stays available during node
drains and voluntary disruptions (upgrades, scale-down):

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: fabtech-api-pdb
  namespace: fabtech
spec:
  minAvailable: 1
  selector:
    matchLabels:
      app: fabtech-api
---
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: fabtech-web-pdb
  namespace: fabtech
spec:
  minAvailable: 1
  selector:
    matchLabels:
      app: fabtech-web
```

### Production Readiness: Zone Spread

Add `topologySpreadConstraints` to each Deployment's pod spec to distribute
pods evenly across availability zones:

```yaml
topologySpreadConstraints:
- maxSkew: 1
  topologyKey: topology.kubernetes.io/zone
  whenUnsatisfiable: DoNotSchedule
  labelSelector:
    matchLabels:
      app: fabtech-api
```

```bash
kubectl apply -f pdb.yaml
kubectl get pdb -n fabtech
```
