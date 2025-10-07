===== AZ-104 Basic Infra (PowerShell + Azure CLI) =====

🧩 Project Overview — AZ-104 Basic Infrastructure

This PowerShell + Azure CLI project automates the deployment of a basic cloud infrastructure in Azure, suitable for system administration and AZ-104 practice.
It creates a complete environment including:

. A Resource Group and Virtual Network with front/back subnets

. A Network Security Group with an RDP rule

. Two Virtual Machines (Windows frontend, Linux backend)

. A Storage Account and Log Analytics Workspace for monitoring

✅ Goal: Learn to provision, secure, and manage core Azure components using scripting and automation rather than the portal.

```powershell
Subscription (set the active one)

az account set --subscription "Azure subscription 1"

Variables

$rg         = "RG-BasicInfra"
$location   = "westeurope"      # fallback: "northeurope"
$vnet       = "VNet-Company"
$frontSubnet= "frontend"
$backSubnet = "backend"
$nsg        = "NSG-frontend"
$vm1        = "VM-Frontend"
$vm2        = "VM-Backend"
$storage    = ("st{0}company" -f (Get-Random)).ToLower()
$workspace  = "LAW-Company"

# Ensure required resource providers are registered (idempotent)
az provider register -n Microsoft.Network
az provider register -n Microsoft.Compute
az provider register -n Microsoft.Storage
az provider register -n Microsoft.OperationalInsights
az provider register -n Microsoft.Insights

# 1) Resource Group
az group create --name $rg --location $location

# 2) VNet + Subnets
az network vnet create --resource-group $rg --name $vnet --address-prefix 10.0.0.0/16 --subnet-name $frontSubnet --subnet-prefix 10.0.1.0/24
az network vnet subnet create --resource-group $rg --vnet-name $vnet --name $backSubnet --address-prefix 10.0.2.0/24

# 3) NSG + RDP rule
az network nsg create --resource-group $rg --name $nsg
az network nsg rule create --resource-group $rg --nsg-name $nsg --name Allow-RDP --protocol Tcp --direction Inbound --priority 1000 --source-address-prefix Internet --destination-port-range 3389 --access Allow

# 4) Windows VM (public) — force an available size
az vm create --resource-group $rg --name $vm1 --image Win2022Datacenter `
  --vnet-name $vnet --subnet $frontSubnet --nsg $nsg `
  --size Standard_B2s `
  --location $location `
  --admin-username azureuser --admin-password 'Password123!'

# 5) Linux VM (private only) — NIC first, no public IP
az network nic create --resource-group $rg --name nic-backend --vnet-name $vnet --subnet $backSubnet
az vm create --resource-group $rg --name $vm2 --image Ubuntu2204 `
  --nics nic-backend `
  --size Standard_B1s `
  --location $location `
  --admin-username azureuser --generate-ssh-keys

# 6) Storage Account
az storage account create --name $storage --resource-group $rg --sku Standard_LRS --kind StorageV2 --location $location

# 7) Log Analytics Workspace
az monitor log-analytics workspace create --resource-group $rg --workspace-name $workspace --location $location

Write-Host "Deployment complete. RG: $rg, Region: $location"

```
