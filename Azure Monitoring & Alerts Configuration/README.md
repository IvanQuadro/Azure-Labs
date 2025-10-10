
```powershell
#STEP 1 – DEFINE VARIABLES
$rg		="RG-BasicInfra"
$location 	="westeurope"
$workspace	="Log-Workspace"
$vmFront	="VM-Frontend"
$vmBack		="VM-Backend"
$actionGroup 	="AC-Group"
$email		="email.com"

#STEP 2 – REGISTER PROVIDERS
az provider register -n Microsoft.Network
az provider register -n Microsoft.Compute
az provider register -n Microsoft.Insights
az provider register -n Microsoft.OperationalInsights


#STEP 3 – ENABLE VM MONITORING
$workspaceId = az monitor log-analytics workspace show -g $rg -n $workspace --query id -o tsv

az monitor diagnostic-settings create -g $rg -n Diag-Front --resource $vmFront --resource-type Microsoft.Compute/virtualMachines --workspace $workspaceId 

az monitor diagnostic-settings create -g $rg -n Diag-Back --resource $vmBack --resource-type Microsoft.Compute/virtualMachines --workspace $workspaceId 

#STEP 4 – CREATE ACTION GROUP
az monitor action-group create -g $rg -n $actionGroup --action email AdminEmail $email

#STEP 5 – CREATE ALERT RULE
$vmFrontId = az vm show -g $rg -n $vmFront --query id -o tsv 

az monitor metrics alert create -g $rg -n HighCPUAlert --scopes $vmFrontId --action-group $actionGroup --condition "avg Percentage CPU > 80" --window-size PT5M --evaluation-frequency PT1M --severity 2 --description "High CPU usage on frontend VM"

#STEP 6 – VALIDATE AND MONITOR
az monitor metrics alert list -g $rg -o table
```
