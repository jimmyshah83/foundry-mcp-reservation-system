# Time-Series Forecasting with Azure ML AutoML

This notebook trains a time-series forecasting model for retail sales using Azure ML AutoML.

## Prerequisites

- Azure ML Workspace access
- Azure CLI installed and logged in (`az login`)
- Storage Blob Data Contributor role on the workspace storage account

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

## Troubleshooting

### Error: KeyBasedAuthenticationNotPermitted
This error occurs when trying to upload via the Python SDK. Use the Azure CLI commands above instead.

### Error: Storage Blob Data Contributor role required
Ensure you have the correct role:
```bash
az role assignment create \
  --role "Storage Blob Data Contributor" \
  --assignee-object-id $(az ad signed-in-user show --query id -o tsv) \
  --scope /subscriptions/<subscription-id>/resourceGroups/admin-rg/providers/Microsoft.Storage/storageAccounts/mldemowkspwus02609576373
```

### Error: Public network access disabled
Enable public network access on the storage account:
```bash
az storage account update \
  --name mldemowkspwus02609576373 \
  --resource-group admin-rg \
  --public-network-access Enabled
```

