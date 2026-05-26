# --- Variables ---
$RG       = "rg-la-demo"
$LOC      = "westus"
$ST       = "stladdemo$(Get-Random -Maximum 99999)"
$PLAN     = "plan-la-demo$(Get-Random -Maximum 99)"
$APP      = "logic-la-demo-$(Get-Random -Maximum 999)"
$UAMI     = "uami-la-demo"
$KV       = "kvla$(Get-Random -Maximum 99999)"

# --- Auth ---
az login
az account set --subscription "<your-sub-id-or-name>"

# --- Core resources ---
az group create -n $RG -l $LOC
az storage account create -n $ST -g $RG -l $LOC --sku Standard_LRS --allow-shared-key-access false
az functionapp plan create -g $RG -n $PLAN -l $LOC --sku WS1
az identity create -g $RG -n $UAMI

$UAMI_ID  = az identity show -g $RG -n $UAMI --query id -o tsv
$UAMI_CID = az identity show -g $RG -n $UAMI --query clientId -o tsv
$UAMI_PID = az identity show -g $RG -n $UAMI --query principalId -o tsv
$ST_ID    = az storage account show -g $RG -n $ST --query id -o tsv

# --- RBAC: UAMI -> Storage ---
foreach ($r in "Storage Blob Data Owner","Storage Queue Data Contributor","Storage Table Data Contributor","Storage Account Contributor","Storage File Data Privileged Contributor") {
  az role assignment create --assignee-object-id $UAMI_PID --assignee-principal-type ServicePrincipal --role $r --scope $ST_ID
}

# --- Key Vault for secret storage (Functions host needs this when keyless) ---
az keyvault create -g $RG -n $KV -l $LOC --enable-rbac-authorization true
$KVID = az keyvault show -g $RG -n $KV --query id -o tsv
az role assignment create --assignee-object-id $UAMI_PID --assignee-principal-type ServicePrincipal --role "Key Vault Secrets Officer" --scope $KVID

# --- Provision Logic App site via Bicep ---
$PLAN_ID = az functionapp plan show -g $RG -n $PLAN --query id -o tsv
cd .\samples\sample-logicapp-workspace\LogicApps
az deployment group create -g $RG -f logicapp.bicep -p name=$APP planId=$PLAN_ID uamiId=$UAMI_ID uamiClientId=$UAMI_CID

# --- Wire up all app settings (storage MI + KV secrets) ---
az webapp config appsettings set -g $RG -n $APP --settings `
  "AzureWebJobsStorage__accountName=$ST" `
  "AzureWebJobsStorage__credential=managedidentity" `
  "AzureWebJobsStorage__clientId=$UAMI_CID" `
  "AzureWebJobsStorage__credentialType=managedIdentity" `
  "AzureWebJobsStorage__managedIdentityResourceId=$UAMI_ID" `
  "AzureWebJobsStorage__blobServiceUri=https://$ST.blob.core.windows.net" `
  "AzureWebJobsStorage__queueServiceUri=https://$ST.queue.core.windows.net" `
  "AzureWebJobsStorage__tableServiceUri=https://$ST.table.core.windows.net" `
  "AzureWebJobsSecretStorageType=keyvault" `
  "AzureWebJobsSecretStorageKeyVaultUri=https://$KV.vault.azure.net" `
  "AzureWebJobsSecretStorageKeyVaultClientId=$UAMI_CID"

# --- Deploy workflow code ---
Compress-Archive -Path .\host.json,.\connections.json,.\Stateful1 -DestinationPath app.zip -Force
az logicapp deployment source config-zip -g $RG -n $APP --src app.zip
az logicapp restart -g $RG -n $APP

# --- Wait & verify ---
Start-Sleep 90
curl "https://$APP.azurewebsites.net"

# --- Get callback URL & invoke ---
$SUB = az account show --query id -o tsv
$cb = az rest --method post --uri "https://management.azure.com/subscriptions/$SUB/resourceGroups/$RG/providers/Microsoft.Web/sites/$APP/hostruntime/runtime/webhooks/workflow/api/management/workflows/Stateful1/triggers/When_a_HTTP_request_is_received/listCallbackUrl?api-version=2018-11-01" --query value -o tsv
Invoke-RestMethod -Method Post -Uri $cb -Body "{}" -ContentType "application/json"
# Expect: Hello world

# --- Teardown when done ---
# az group delete -n $RG --yes --no-wait