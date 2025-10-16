

#STEP 1 – DEFINE VARIABLES
$rg            = "RG-basicInfra"
$location      = "westeurope"
$workspace     = "LA-Security"
$subId         = "<_SUBSCRIPTION_ID_>"
$actionGroup   = "AG-Security"
$adminEmail    = "you@example.com"

#STEP 2 – REGISTER PROVIDERS
az provider register -n Microsoft.Network
az provider register -n Microsoft.Security
az provider register -n Microsoft.OperationalInsights
az provider register -n Microsoft.PolicyInsights
az provider register -n Microsoft.Insights   # required for monitoring/alerts

#STEP 3 – CORE RESOURCES
az group create -g $rg --location $location
az monitor log-analytics workspace create -g $rg -n $workspace --location $location

#STEP 4 – ENABLE DEFENDER FOR CLOUD
az security pricing create --name VirtualMachines --tier Standard
az security pricing create --name StorageAccounts --tier Standard
az security pricing create --name SqlServers --tier Standard
az security pricing create --name AppServices --tier Standard

#STEP 5 – ASSIGN MICROSOFT CLOUD SECURITY BENCHMARK (MCSB)
$initiativeId = az policy set-definition list --query "[?contains(displayName, 'Microsoft cloud security benchmark')].id | [0]" -o tsv

az policy assignment create \
  --name "Policy" \
  --display-name "Microsoft Cloud Security Benchmark - Baseline" \
  --scope "/subscriptions/$subId" \
  --location $location \
  --policy-set-definition $initiativeId \
  --identity-type SystemAssigned

#STEP 6 – CONFIGURE DIAGNOSTIC SETTINGS
$workspaceId = az monitor log-analytics workspace show -g $rg -n $workspace --query id -o tsv

az monitor diagnostic-settings create \
  --name "SubActivityToLA" \
  --resource /subscriptions/$subId \
  --workspace $workspaceId \
  --logs '[
    {"category":"Administrative","enabled":true},
    {"category":"Security","enabled":true},
    {"category":"Policy","enabled":true},
    {"category":"Alert","enabled":true}
  ]'

#STEP 7 – CREATE ACTION GROUP
az monitor action-group create \
  -g $rg \
  -n $actionGroup \
  --action email AdminEmail $adminEmail

#STEP 8 – CREATE SECURITY ALERT RULE
az monitor activity-log alert create \
  -g $rg \
  --name "SecurityAlert" \
  --scope /subscriptions/$subId \
  --condition category=Security and level=Warning \
  --action-group $actionGroup \
  --severity 0
