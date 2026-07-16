# Challenge 04 — Workload Identity & Secrets — Coach Solution

[< Previous Solution](./Solution-03.md) | [Home](../../README.md) | [Next Solution >](./Solution-05.md)

## Notes & Guidance

- The most common failure is forgetting to **annotate the Kubernetes ServiceAccount** with
  `azure.workload.identity/client-id`. The pod gets a token but it belongs to no identity.
- The `SecretProviderClass` requires the exact `tenantId` and `clientID` — teams copy-paste
  errors here constantly. A `describe` on the pod shows the mount error clearly.
- On AKS, prefer the managed `azure-keyvault-secrets-provider` add-on instead of installing
  the Azure provider via Helm. The managed add-on supports `secretObjects` syncing, so no
  Helm-specific `syncSecret.enabled` setting is required.
- **Secrets layered approach**: Kubernetes Secret objects are base64-encoded (not encrypted at rest
  by default). They are better than plaintext env vars but still accessible to anyone with `kubectl get secret`.
  Key Vault + CSI driver is the production-grade approach: secrets never land in etcd, and access
  is fully audited. If teams ask why the `secretObjects` sync creates a K8s Secret, clarify that
  this is a compatibility bridge for legacy apps that expect `secretKeyRef` — the canonical path
  remains the CSI volume mount directly from Key Vault.
- The `azure.workload.identity/use: "true"` label must be on the **Pod** (not Deployment
  selector labels alone) — it flows via the pod template spec labels.
- Verify CSI driver running: `kubectl get pods -n kube-system | grep secrets-store`

## Solution

### Part 1: Key Vault and Secret

```bash
RG=rg-frontier-aks
LOCATION=swedencentral
KV_NAME=kv-frontier-$RANDOM
# Notice that te connection string uses the $DB_PASS from Challenge 03

az keyvault create \
  --resource-group $RG \
  --name $KV_NAME \
  --location $LOCATION \
  --enable-rbac-authorization true

# Store the secret (use the in-cluster connection string from Challenge 03)
az keyvault secret set \
  --vault-name $KV_NAME \
  --name "db-connection-string" \
  --value "postgresql://fabadmin:${DB_PASS}@fabtech-pg-postgresql.fabtech.svc.cluster.local:5432/fabtech"

echo "Key Vault: $KV_NAME"
```

### Part 2: Managed Identity and Federated Credential

```bash
CLUSTER_NAME=aks-frontier
NAMESPACE=fabtech
SA_NAME=fabtech-api-sa
MI_NAME=mi-fabtech-api

# Create the identity
az identity create --resource-group $RG --name $MI_NAME
MI_CLIENT_ID=$(az identity show --resource-group $RG --name $MI_NAME --query clientId -o tsv)
MI_OBJECT_ID=$(az identity show --resource-group $RG --name $MI_NAME --query principalId -o tsv)

# Grant Key Vault Secrets User role
KV_ID=$(az keyvault show --name $KV_NAME --query id -o tsv)
az role assignment create \
  --role "Key Vault Secrets User" \
  --assignee-object-id $MI_OBJECT_ID \
  --assignee-principal-type ServicePrincipal \
  --scope $KV_ID

# Get cluster OIDC issuer
OIDC_ISSUER=$(az aks show \
  --resource-group $RG \
  --name $CLUSTER_NAME \
  --query "oidcIssuerProfile.issuerUrl" -o tsv)

# Create Kubernetes ServiceAccount
kubectl create namespace $NAMESPACE --dry-run=client -o yaml | kubectl apply -f -
kubectl create serviceaccount $SA_NAME --namespace $NAMESPACE

# Annotate it
kubectl annotate serviceaccount $SA_NAME \
  --namespace $NAMESPACE \
  "azure.workload.identity/client-id=$MI_CLIENT_ID"

# Create federated credential
az identity federated-credential create \
  --name fc-fabtech-api \
  --identity-name $MI_NAME \
  --resource-group $RG \
  --issuer $OIDC_ISSUER \
  --subject "system:serviceaccount:${NAMESPACE}:${SA_NAME}" \
  --audience api://AzureADTokenExchange
```

### Part 3: Secrets Store CSI Driver

```bash
# Enable Azure Key Vault Secrets Provider add-on (managed by AKS) if not already enabled
az aks enable-addons \
  --resource-group $RG \
  --name $CLUSTER_NAME \
  --addons azure-keyvault-secrets-provider

# Verify the add-on pods are running
kubectl get pods -n kube-system -l app=secrets-store-csi-driver
kubectl get pods -n kube-system -l app=secrets-store-provider-azure
```

`SecretProviderClass` manifest:

```yaml
# secretproviderclass.yaml
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: fabtech-secrets
  namespace: fabtech
spec:
  provider: azure
  parameters:
    usePodIdentity: "false"
    useVMManagedIdentity: "false"
    clientID: "<MI_CLIENT_ID>"
    keyvaultName: "<KV_NAME>"
    cloudName: ""
    objects: |
      array:
        - |
          objectName: db-connection-string
          objectType: secret
    tenantId: "<TENANT_ID>"
  secretObjects:
    - data:
        - key: connectionString
          objectName: db-connection-string
      secretName: fabtech-db-secret
      type: Opaque
```

```bash
TENANT_ID=$(az account show --query tenantId -o tsv)
sed -e "s/<MI_CLIENT_ID>/$MI_CLIENT_ID/g" \
    -e "s/<KV_NAME>/$KV_NAME/g" \
    -e "s/<TENANT_ID>/$TENANT_ID/g" \
    secretproviderclass.yaml | kubectl apply -f -
```

### Part 4: Update Deployment

Key additions to the Deployment (the label **must** be in `template.metadata.labels`, not
in the pod spec itself):

```yaml
  template:
    metadata:
      labels:
        app: fabtech-api
        azure.workload.identity/use: "true"   # Required — injects the federated token
    spec:
      serviceAccountName: fabtech-api-sa
      volumes:
      - name: secrets-store
        csi:
          driver: secrets-store.csi.k8s.io
          readOnly: true
          volumeAttributes:
            secretProviderClass: fabtech-secrets
      containers:
      - name: api
        securityContext:
          runAsNonRoot: true
          runAsUser: 100
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
          capabilities:
            drop: ["ALL"]
        volumeMounts:
        - name: secrets-store
          mountPath: /mnt/secrets
          readOnly: true
        env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: fabtech-db-secret
              key: connectionString
```

Notice that this Deployment mounts the secret as a volume and also exposes it as an environment variable. The application can use either method to access the database connection string.

Since we have been deploying the `fabtech-api` using helm, you can update the helm chart values to include the new service account and volume mounts, or you can patch the existing deployment with `kubectl edit deployment fabtech-api -n $NAMESPACE` and add the necessary fields.

Example with helm

- Edit `api-deployment.yaml` in the helm chart templates to include the service account and volume mounts as shown above.
- Upgrade the helm release:
```bash
ACR_LOGIN_SERVER=$(az acr show --name $ACR_NAME --query loginServer -o tsv)
CHART_PATH="./Student/Resources/src/manifests/chart"
helm upgrade --install fabtech $CHART_PATH \
  --namespace $NAMESPACE \
  --set api.image.repository=$ACR_LOGIN_SERVER/fabtech-api \
  --set web.image.repository=$ACR_LOGIN_SERVER/fabtech-web 
```

### Verify

```bash
# check rollout status
kubectl rollout status deployment/fabtech-api -n $NAMESPACE
# when ready, exec into the pod and check the secret mounted from Key Vault
POD=$(kubectl get pod -n fabtech -l app=fabtech-api -o jsonpath='{.items[0].metadata.name}')
kubectl exec -n fabtech $POD -- cat /mnt/secrets/db-connection-string
```
