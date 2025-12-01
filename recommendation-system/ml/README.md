# Time-Series Forecasting with Azure ML AutoML

This notebook trains a time-series forecasting model for retail sales using Azure ML AutoML.

## Prerequisites

- Azure ML Workspace access
- Azure CLI installed and logged in (`az login`)
- Required roles on the workspace storage account:
  - **Storage Blob Data Contributor**
  - **Storage File Data Privileged Contributor**

## Data Upload (Manual Step Required)

Due to Azure Policy restrictions that disable SAS token authentication on the storage account, data must be uploaded manually using Azure CLI with OAuth authentication before running the notebook.

### Step 1: Prepare the Data

Run cells 10-19 in the notebook to:
1. Load and merge the retail datasets
2. Perform feature engineering
3. Create train/validation split
4. Save data as MLTable format locally

### Step 2: Upload Data to Blob Storage

Run these commands from the `recommendation-system/ml/` directory:

```bash
# Get the datastore container name
az ml datastore show --name workspaceblobstore \
  --resource-group admin-rg \
  --workspace-name ml-demo-wksp-wus-01 \
  --query "container_name" -o tsv

# Upload training data
az storage blob upload-batch \
  --account-name mldemowkspwus02609576373 \
  --destination "azureml-blobstore-cff56e3a-d016-4526-aa58-71c460675066/retail-training-data" \
  --source ./data/training-mltable-folder \
  --auth-mode login \
  --overwrite

# Upload validation data
az storage blob upload-batch \
  --account-name mldemowkspwus02609576373 \
  --destination "azureml-blobstore-cff56e3a-d016-4526-aa58-71c460675066/retail-validation-data" \
  --source ./data/validation-mltable-folder \
  --auth-mode login \
  --overwrite
```

### Step 3: Run the Notebook

After uploading, run the remaining cells (20+) to:
1. Reference the uploaded data
2. Configure and submit the AutoML forecasting job
3. Download and evaluate the trained model

## Why Manual Upload?

The Azure ML workspace has an Azure Policy that disables key-based authentication (SAS tokens) on the storage account. This is a security best practice but prevents the Python SDK from uploading data directly.

The workaround uses `az storage blob upload-batch --auth-mode login` which authenticates using your Azure AD identity (OAuth) instead of SAS tokens.

## Identity-Based Datastore Setup

The default `workspaceblobstore` datastore is configured to use Account Key authentication, which fails when key-based auth is disabled. To browse data in ML Studio and use the data in jobs, create an identity-based datastore:

### Create Identity-Based Datastore

```bash
# Create a YAML file for the identity-based datastore
cat > /tmp/datastore-identity.yaml << 'EOF'
$schema: https://azuremlschemas.azureedge.net/latest/azureBlob.schema.json
name: workspaceblobstore_identity
type: azure_blob
account_name: mldemowkspwus02609576373
container_name: azureml-blobstore-cff56e3a-d016-4526-aa58-71c460675066
EOF

# Create the datastore (no credentials = uses Azure AD identity)
az ml datastore create --file /tmp/datastore-identity.yaml \
  --resource-group admin-rg \
  --workspace-name ml-demo-wksp-wus-01
```

### Browse Data in ML Studio

1. Go to [Azure ML Studio](https://ml.azure.com)
2. Navigate to **Data** → **Datastores**
3. Click on **workspaceblobstore_identity** (the identity-based datastore)
4. Click **Browse** to see uploaded data:
   - `retail-training-data/`
   - `retail-validation-data/`

### Reference Data in Notebook

The notebook uses the identity-based datastore to reference data:

```python
my_training_data_input = Input(
    type=AssetTypes.MLTABLE, 
    path="azureml://datastores/workspaceblobstore_identity/paths/retail-training-data"
)
```

## Compute Cluster Storage Access

When running AutoML jobs, the compute cluster uses its managed identity to access storage - not your user identity. You must grant the compute cluster and workspace managed identities access to the storage account.

### Grant Compute Cluster Access

```bash
# Get the compute cluster's managed identity principal ID
COMPUTE_IDENTITY=$(az ml compute show --name teslat4-gpu-wus \
  --resource-group admin-rg \
  --workspace-name ml-demo-wksp-wus-01 \
  --query "identity.principal_id" -o tsv)

echo "Compute identity: $COMPUTE_IDENTITY"

# Grant Storage Blob Data Contributor to the compute cluster
az role assignment create \
  --role "Storage Blob Data Contributor" \
  --assignee-object-id $COMPUTE_IDENTITY \
  --assignee-principal-type ServicePrincipal \
  --scope /subscriptions/57123c17-af1a-4ec2-9494-a214fb148bf4/resourceGroups/admin-rg/providers/Microsoft.Storage/storageAccounts/mldemowkspwus02609576373
```

### Grant Workspace Managed Identity Access

```bash
# Get the workspace's managed identity principal ID
WORKSPACE_IDENTITY=$(az ml workspace show --name ml-demo-wksp-wus-01 \
  --resource-group admin-rg \
  --query "identity.principal_id" -o tsv)

echo "Workspace identity: $WORKSPACE_IDENTITY"

# Grant Storage Blob Data Contributor to the workspace
az role assignment create \
  --role "Storage Blob Data Contributor" \
  --assignee-object-id $WORKSPACE_IDENTITY \
  --assignee-principal-type ServicePrincipal \
  --scope /subscriptions/57123c17-af1a-4ec2-9494-a214fb148bf4/resourceGroups/admin-rg/providers/Microsoft.Storage/storageAccounts/mldemowkspwus02609576373
```

**Note:** Role assignments can take 2-5 minutes to propagate. Wait before resubmitting the job.

## Troubleshooting

### Error: KeyBasedAuthenticationNotPermitted
This error occurs when trying to upload via the Python SDK. Use the Azure CLI commands above instead.

### Error: You don't have permissions to access this datastore
This occurs when browsing `workspaceblobstore` in ML Studio because it uses Account Key auth. Use the identity-based datastore `workspaceblobstore_identity` instead (see Identity-Based Datastore Setup above).

### Error: Storage Blob Data Contributor role required
Ensure you have both required roles:
```bash
# Storage Blob Data Contributor
az role assignment create \
  --role "Storage Blob Data Contributor" \
  --assignee-object-id $(az ad signed-in-user show --query id -o tsv) \
  --assignee-principal-type User \
  --scope /subscriptions/57123c17-af1a-4ec2-9494-a214fb148bf4/resourceGroups/admin-rg/providers/Microsoft.Storage/storageAccounts/mldemowkspwus02609576373

# Storage File Data Privileged Contributor
az role assignment create \
  --role "Storage File Data Privileged Contributor" \
  --assignee-object-id $(az ad signed-in-user show --query id -o tsv) \
  --assignee-principal-type User \
  --scope /subscriptions/57123c17-af1a-4ec2-9494-a214fb148bf4/resourceGroups/admin-rg/providers/Microsoft.Storage/storageAccounts/mldemowkspwus02609576373
```

### Error: Public network access disabled
Enable public network access on the storage account:
```bash
az storage account update \
  --name mldemowkspwus02609576373 \
  --resource-group admin-rg \
  --public-network-access Enabled
```

### Error: Permission denied when accessing MLTable during job execution
```
PermissionDenied(Some(This request is not authorized to perform this operation using this permission.))
```

This occurs when the compute cluster's managed identity doesn't have access to the storage account. The job runs as the compute cluster's identity, not your user identity.

**Solution:** Grant the compute cluster and workspace managed identities access to storage. See the "Compute Cluster Storage Access" section above.

