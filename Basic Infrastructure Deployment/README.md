🧩 AZ-104 Basic Infrastructure Deployment (PowerShell + Azure CLI)
📘 Overview

This project automates the deployment of a basic Azure infrastructure using PowerShell and Azure CLI, ideal for AZ-104 practice and hands-on learning in system administration.
It demonstrates how to create, secure, and monitor essential Azure resources entirely through scripting — no portal interaction needed.

🏗️ What It Deploys

🗂️ Resource Group — centralized container for all resources

🌐 Virtual Network (VNet) — includes 2 subnets: frontend and backend

🔒 Network Security Group (NSG) — allows RDP access for the Windows VM

💻 Compute —

1× Windows Server 2022 (public frontend)

1× Ubuntu Server 22.04 (private backend)

💾 Storage Account — general-purpose Standard_LRS

📊 Log Analytics Workspace — for monitoring and diagnostics

🎯 Goal

Build a complete, minimal infrastructure-as-code (IaC) setup using Azure CLI.
Learn how networking, compute, storage, and monitoring integrate in a real environment.

🧱 Architecture Summary
Component	Name	Purpose
Resource Group	RG-BasicInfra	Logical container
VNet	VNet-Company	Internal network with two subnets
NSG	NSG-frontend	RDP/SSH traffic control
VMs	VM-Frontend, VM-Backend	Windows + Linux hosts
Storage	stcompany[random]	Data + diagnostics storage
Log Analytics	LAW-Company	Log and performance analysis
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
```powershell
# Run from PowerShell
.\create_infra.ps1
```
