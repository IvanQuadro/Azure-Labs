# ============================
# Project 2: Advanced NSG & Diagnostics (clean version)
# ============================

```powershell
$ErrorActionPreference = "Stop"

# ---- 0) CONTEXT ----
az account set --subscription "Azure subscription 1"

# ---- 1) VARIABLES ----
$rg          = "RG-BasicInfra"
$location    = "westeurope"
$vnet        = "VNet-Company"
$nsg         = "NSG-frontend"
$vmFront     = "VM-Frontend"
$vmBack      = "VM-Backend"
$asgFrontend = "ASG-Frontend"
$asgBackend  = "ASG-Backend"
$workspace   = "LAW-Company"
$myIP        = "93.40.208.91"  

# ---- 2) PROVIDERS & NETWORK WATCHER ----
az provider register -n Microsoft.Network
az provider register -n Microsoft.Insights
az provider register -n Microsoft.OperationalInsights
az provider register -n Microsoft.Compute
az provider register -n Microsoft.Storage

az network watcher configure --locations $location --enabled true

# ---- 3) DISCOVER EXISTING RESOURCES ----
$subId      = az account show --query id -o tsv
$storage    = az storage account list -g $rg --query "[0].name" -o tsv
$lawId      = az monitor log-analytics workspace show -g $rg -n $workspace --query id -o tsv

$nicFrontId = az vm show -g $rg -n $vmFront --query "networkProfile.networkInterfaces[0].id" -o tsv
$nicBackId  = az vm show -g $rg -n $vmBack  --query "networkProfile.networkInterfaces[0].id" -o tsv
$nicFront   = Split-Path -Leaf $nicFrontId
$nicBack    = Split-Path -Leaf $nicBackId
$ipcfgFront = az network nic show -g $rg -n $nicFront --query "ipConfigurations[0].name" -o tsv
$ipcfgBack  = az network nic show -g $rg -n $nicBack  --query "ipConfigurations[0].name" -o tsv

# ---- 4) CREATE ASGs & ASSOCIATE TO NICs ----
az network asg create -g $rg -n $asgFrontend --location $location
az network asg create -g $rg -n $asgBackend  --location $location

$asgFrontendId = az network asg show -g $rg -n $asgFrontend --query id -o tsv
$asgBackendId  = az network asg show -g $rg -n $asgBackend  --query id -o tsv

az network nic ip-config update -g $rg --nic-name $nicFront -n $ipcfgFront --application-security-groups $asgFrontendId
az network nic ip-config update -g $rg --nic-name $nicBack  -n $ipcfgBack  --application-security-groups $asgBackendId

# ---- 5) CLEAN OLD RULES (OPTIONAL) ----
az network nsg rule delete -g $rg --nsg-name $nsg -n Allow-RDP 2>$null

# ---- 6) NSG RULES ----
az network nsg rule create -g $rg --nsg-name $nsg -n Allow-HTTP `
  --priority 1000 --direction Inbound --protocol Tcp --access Allow `
  --source-address-prefixes Internet --destination-asgs $asgFrontendId --destination-port-ranges 80

az network nsg rule create -g $rg --nsg-name $nsg -n Allow-HTTPS `
  --priority 1010 --direction Inbound --protocol Tcp --access Allow `
  --source-address-prefixes Internet --destination-asgs $asgFrontendId --destination-port-ranges 443

az network nsg rule create -g $rg --nsg-name $nsg -n Allow-RDP-MyIP `
  --priority 1020 --direction Inbound --protocol Tcp --access Allow `
  --source-address-prefixes $myIP --destination-asgs $asgFrontendId --destination-port-ranges 3389

az network nsg rule create -g $rg --nsg-name $nsg -n Allow-SSH-MyIP `
  --priority 1030 --direction Inbound --protocol Tcp --access Allow `
  --source-address-prefixes $myIP --destination-asgs $asgBackendId  --destination-port-ranges 22

az network nsg rule create -g $rg --nsg-name $nsg -n Allow-FrontToBack `
  --priority 1040 --direction Inbound --protocol Tcp --access Allow `
  --source-asgs $asgFrontendId --destination-asgs $asgBackendId --destination-port-ranges 22

az network nsg rule list -g $rg --nsg-name $nsg -o table

# ---- 7) FLOW LOGS + TRAFFIC ANALYTICS ----
az network watcher flow-log configure `
  --resource-group $rg --nsg $nsg --enabled true `
  --storage-account $storage `
  --traffic-analytics true `
  --workspace $workspace --workspace-resource-group $rg --workspace-subscription $subId `
  --retention 7

az network watcher flow-log show -g $rg --nsg $nsg -o table

# ---- 8) DIAGNOSTIC SETTINGS (NSG + VMs -> LAW) ----
$nsgId     = az network nsg show -g $rg -n $nsg --query id -o tsv
$vmFrontId = az vm show -g $rg -n $vmFront --query id -o tsv
$vmBackId  = az vm show -g $rg -n $vmBack  --query id -o tsv

az monitor diagnostic-settings create --name diag-nsg `
  --resource $nsgId --workspace $lawId `
  --logs '[
    {"category":"NetworkSecurityGroupEvent","enabled":true},
    {"category":"NetworkSecurityGroupRuleCounter","enabled":true}
  ]'

az monitor diagnostic-settings create --name diag-vm-frontend `
  --resource $vmFrontId --workspace $lawId `
  --metrics '[{"category":"AllMetrics","enabled":true}]'

az monitor diagnostic-settings create --name diag-vm-backend `
  --resource $vmBackId --workspace $lawId `
  --metrics '[{"category":"AllMetrics","enabled":true}]'

az monitor diagnostic-settings list --resource $nsgId -o table
```
