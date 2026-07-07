# Challenge 09 — Storage — Coach Solution

[< Previous Solution](./Solution-08.md) | [Home](../../README.md) | [Next Solution >](./Solution-10.md)

## Notes & Guidance

- **Pedagogical intent:** This challenge is a storage deep-dive that builds on the in-cluster PostgreSQL from Challenge 03 (deployed via the **Bitnami Helm chart**, whose PVC abstracted storage away). Here teams deploy PostgreSQL as an **explicit StatefulSet with an Azure Disk-backed StorageClass** so they can observe PVC provisioning, Azure Disk CSI attachment, and data persistence through pod recreation first-hand. Remind teams that production workloads should use a managed **Azure Database for PostgreSQL Flexible Server**.

- Key concept to drive home: **ReadWriteOnce (RWO)** for Azure Disk (one pod at a time,
  not shareable across nodes); **ReadWriteMany (RWX)** for Azure Files (multiple pods,
  multiple nodes).
- `StatefulSet` with `volumeClaimTemplates` creates one PVC per pod replica. This is
  fundamentally different from a Deployment with one PVC — emphasize this pattern for
  stateful workloads (databases, message queues).
- Azure Backup for AKS is relatively new. The Portal wizard is easier than pure CLI.
  Good to show for completeness but not a blocker for the challenge.
- The `managed-csi-premium` storage class (Premium SSD) is preferred for production
  database workloads; `managed-csi` (Standard SSD) is fine for the hackathon.

### Common Issues

- **Pod stuck in Pending due to PVC not bound:** Check `kubectl describe pvc`. Common causes:
  wrong storage class name, zone mismatch between PVC and node.
- **Azure Files RWX mount failing:** Ensure the `azurefile-csi` or `azurefile-csi-premium`
  storage class is used.
- **Data lost after pod deletion:** Usually means the pod was using `emptyDir` instead of a
  PVC, or the PVC was deleted along with the pod (StatefulSet `--cascade=orphan` avoids this).
- **`CreateContainerConfigError: couldn't find key password in Secret`:** The existing
  `fabtech-db-secret` from Challenge 04 only contains `connectionString`, not `password`.
  Either create a new secret or patch the existing one:
  ```bash
  kubectl patch secret fabtech-db-secret -n fabtech --type=json \
    -p='[{"op":"add","path":"/data/password","value":"'$(echo -n $DB_PASS | base64 -w0)'"}]'
  ```
- **`Permission denied` creating `/var/lib/postgresql/data/pgdata`:** The official `postgres`
  image runs as UID 999 but Azure Disk volumes are mounted with root ownership. Add
  `securityContext.fsGroup: 999` at the **pod** level (not container level) so Kubernetes
  `chown`s the volume on attach. Also set `PGDATA=/var/lib/postgresql/data/pgdata` env var
  to avoid the "data directory is not empty" error on first init.
- **`runAsUser: 100` breaks the postgres container:** The guide's security context uses
  `runAsUser: 100` which is the `games` user — postgres requires UID 999. Use `runAsUser: 999`
  for the postgres container. `readOnlyRootFilesystem: true` also must be omitted — PostgreSQL
  writes to its data directory and `/tmp` at runtime.

## Solution

### Storage Classes Reference

```bash
kubectl get storageclass
# Key classes:
# managed-csi              → Standard SSD Azure Disk (RWO)
# managed-csi-premium      → Premium SSD Azure Disk (RWO)
# azurefile-csi            → Standard Azure Files (RWX)
# azurefile-csi-premium    → Premium Azure Files SMB (RWX)
```

### Part 1: PostgreSQL StatefulSet with Azure Disk (RWO)

```yaml
# postgres-statefulset.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
  namespace: fabtech
spec:
  serviceName: postgres
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      nodeSelector:
        kubernetes.io/arch: amd64
      securityContext:
        fsGroup: 999        # chown volume to postgres group on attach
      containers:
      - name: postgres
        image: postgres:16
        securityContext:
          runAsNonRoot: true
          runAsUser: 999    # postgres UID — NOT 100
          allowPrivilegeEscalation: false
          capabilities:
            drop: ["ALL"]
          # readOnlyRootFilesystem is intentionally omitted for database workloads
          # PostgreSQL requires write access to its data directory and /tmp
        env:
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: fabtech-db-secret
              key: password
        - name: POSTGRES_DB
          value: fabtech
        - name: PGDATA
          value: /var/lib/postgresql/data/pgdata   # subdirectory avoids "not empty" init error
        ports:
        - containerPort: 5432
        volumeMounts:
        - name: data
          mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: managed-csi-premium
      resources:
        requests:
          storage: 10Gi
---
apiVersion: v1
kind: Service
metadata:
  name: postgres
  namespace: fabtech
spec:
  selector:
    app: postgres
  ports:
  - port: 5432
  clusterIP: None
```

```bash
kubectl apply -f postgres-statefulset.yaml
kubectl get pvc -n fabtech
kubectl get pods -n fabtech -l app=postgres
```

### Part 2: Data Persistence Test

```bash
# Insert test data
kubectl exec -n fabtech postgres-0 -- \
  psql -U postgres -d fabtech -c "CREATE TABLE test (id serial, val text);"
kubectl exec -n fabtech postgres-0 -- \
  psql -U postgres -d fabtech -c "INSERT INTO test (val) VALUES ('persistent-data');"

# Delete the pod (StatefulSet will recreate it)
kubectl delete pod postgres-0 -n fabtech
kubectl wait --for=condition=Ready pod/postgres-0 -n fabtech --timeout=60s

# Verify data survived
kubectl exec -n fabtech postgres-0 -- \
  psql -U postgres -d fabtech -c "SELECT * FROM test;"
```

### Part 3: Azure Files Share for Shared Config (RWX)

```yaml
# shared-config-pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: shared-config
  namespace: fabtech
spec:
  accessModes:
  - ReadWriteMany
  storageClassName: azurefile-csi
  resources:
    requests:
      storage: 5Gi
```

```bash
kubectl apply -f shared-config-pvc.yaml
kubectl get pvc shared-config -n fabtech
# STATUS should be Bound
```

Mount in the API deployment to demonstrate shared access:

```yaml
volumes:
- name: shared-config
  persistentVolumeClaim:
    claimName: shared-config
containers:
- name: api
  volumeMounts:
  - name: shared-config
    mountPath: /app/config
```

### Part 4: Azure Backup for AKS (Optional)

> **Coach Note:** The Azure Portal wizard is the most reliable way to complete this step end-to-end.
> The CLI path for creating backup instances has schema bugs that make it error-prone (see Common Issues).
> Use CLI for infrastructure (extension, vault, policy, roles) and the Portal for backup instance creation.

```bash
# Step 1: Create a storage account for backup staging
SA_NAME="stbackup$RANDOM"
az storage account create \
  --name $SA_NAME \
  --resource-group $RG \
  --location $LOCATION \
  --sku Standard_LRS --kind StorageV2

az storage container create \
  --name backup --account-name $SA_NAME --auth-mode login

# Step 2: Install the AKS Backup extension on the cluster
az k8s-extension create \
  --name azure-aks-backup \
  --extension-type microsoft.dataprotection.kubernetes \
  --scope cluster \
  --cluster-type managedClusters \
  --cluster-name $CLUSTER_NAME \
  --resource-group $RG \
  --release-train stable \
  --configuration-settings \
    blobContainer=backup \
    storageAccount=$SA_NAME \
    storageAccountResourceGroup=$RG \
    storageAccountSubscriptionId=$(az account show --query id -o tsv)

# Verify backup agent pods are running
kubectl get pods -n dataprotection-microsoft

# Step 3: Create Backup Vault with system-assigned identity
az dataprotection backup-vault create \
  --resource-group $RG \
  --vault-name "bv-$CLUSTER_NAME" \
  --location $LOCATION \
  --storage-settings datastore-type="VaultStore" type="LocallyRedundant"

# Enable system-assigned identity (required for backup operations)
az dataprotection backup-vault update \
  --resource-group $RG --vault-name "bv-$CLUSTER_NAME" --type SystemAssigned

# Step 4: Assign required roles
VAULT_PRINCIPAL=$(az dataprotection backup-vault show \
  --resource-group $RG --vault-name "bv-$CLUSTER_NAME" \
  --query identity.principalId -o tsv)
CLUSTER_ID=$(az aks show --resource-group $RG --name $CLUSTER_NAME --query id -o tsv)
SA_ID=$(az storage account show --name $SA_NAME --resource-group $RG --query id -o tsv)
KUBELET_PRINCIPAL=$(az aks show --resource-group $RG --name $CLUSTER_NAME \
  --query "identityProfile.kubeletidentity.objectId" -o tsv)

az role assignment create --assignee-object-id $VAULT_PRINCIPAL \
  --assignee-principal-type ServicePrincipal --role "Reader" --scope $CLUSTER_ID
az role assignment create --assignee-object-id $VAULT_PRINCIPAL \
  --assignee-principal-type ServicePrincipal \
  --role "Storage Blob Data Contributor" --scope $SA_ID
az role assignment create --assignee-object-id $KUBELET_PRINCIPAL \
  --assignee-principal-type ServicePrincipal \
  --role "Storage Blob Data Contributor" --scope $SA_ID

# Step 5: Enable trusted access between vault and cluster
az aks trustedaccess rolebinding create \
  --resource-group $RG --cluster-name $CLUSTER_NAME \
  --name backup-access \
  --source-resource-id $(az dataprotection backup-vault show \
    --resource-group $RG --vault-name "bv-$CLUSTER_NAME" --query id -o tsv) \
  --roles Microsoft.DataProtection/backupVaults/backup-operator

# Step 6: Create backup policy (4-hour incremental, 7-day retention)
az dataprotection backup-policy create \
  --resource-group $RG --vault-name "bv-$CLUSTER_NAME" --name "daily-7d" \
  --policy '{
    "datasourceTypes":["Microsoft.ContainerService/managedClusters"],
    "name":"daily-7d","objectType":"BackupPolicy",
    "policyRules":[
      {"backupParameters":{"backupType":"Incremental","objectType":"AzureBackupParams"},
       "dataStore":{"dataStoreType":"OperationalStore","objectType":"DataStoreInfoBase"},
       "name":"BackupHourly","objectType":"AzureBackupRule",
       "trigger":{"objectType":"ScheduleBasedTriggerContext",
         "schedule":{"repeatingTimeIntervals":["R/2026-07-01T02:00:00+00:00/PT4H"],"timeZone":"UTC"},
         "taggingCriteria":[{"isDefault":true,"tagInfo":{"id":"Default_","tagName":"Default"},"taggingPriority":99}]}},
      {"isDefault":true,
       "lifecycles":[{"deleteAfter":{"duration":"P7D","objectType":"AbsoluteDeleteOption"},
         "sourceDataStore":{"dataStoreType":"OperationalStore","objectType":"DataStoreInfoBase"}}],
       "name":"Default","objectType":"AzureRetentionRule"}
    ]}'
```

**Step 7 — Create backup instance via Azure Portal** (recommended — CLI has schema bugs):

Navigate to: **Azure Portal → AKS cluster → Backup → Configure backup**
- Select vault: `bv-<cluster>`
- Select policy: `daily-7d`
- Select namespaces: `fabtech`
- Enable volume snapshots: Yes
