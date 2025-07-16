# 🍾 Bottler Agent - Azure Deployment Guide

![Cross Tenant Architecture](Deployment/img/Image%20cross.png)

> 🚨 **CRITICAL: Two-Stage Deployment Required**  
> You **MUST** deploy in two separate steps to prevent Key Vault access failures and deployment race conditions.

---

## 🔄 Two-Stage Deployment Process

### Why Two Stages?
The Function App requires access to Key Vault secrets during deployment. A single-stage deployment creates a race condition where the Function App tries to access secrets before permissions are properly configured, causing this error:

> ❌ **AccessToKeyVaultDenied** – Unable to resolve setting: WEBSITE_CONTENTAZUREFILECONNECTIONSTRING

---

## Step 1: 🔐 Pre-Configuration Setup

This deploys foundational infrastructure and configures permissions:

**Includes:**
- ✅ Key Vault + managed identity permissions
- ✅ Storage Account + connection strings 
- ✅ Application Insights + secrets
- ✅ App Service Plan (EP1 for VNet, Y1 for basic)
- ✅ VNet, NSGs, and networking (if enabled)
- ✅ Azure AI Foundry (if enabled)
- ✅ Cosmos DB (if enabled)

### 1️⃣ Full Production Pre-Config
[![Deploy Pre-Config to Azure](https://aka.ms/deploytoazurebutton)](https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2FThe-Coca-Cola-Company%2Fcoke-multiagent-cross-tenant-infra%2Fmain%2FDeployment%2Fbottler-agent-preconfig.json)

### 2️⃣ Minimal Dev/Test Pre-Config  
[![Deploy Pre-Config Minimal](https://aka.ms/deploytoazurebutton)](https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2FThe-Coca-Cola-Company%2Fcoke-multiagent-cross-tenant-infra%2Fmain%2FDeployment%2Fbottler-agent-preconfig.json)

*Use same template - disable AI Foundry/Cosmos DB in parameters*

---

### ⏳ **Wait 60-90 seconds** before proceeding to Step 2

This allows Azure to:
- Complete role assignments
- Propagate permissions
- Initialize Key Vault access

---

## Step 2: 🚀 Bottler Agent Function Deployment

This deploys the Function App using the infrastructure from Step 1:

**Includes:**
- ✅ Function App with system-assigned identity
- ✅ VNet integration (if networking enabled)
- ✅ Key Vault references for app settings
- ✅ Role assignments for storage and Key Vault access

### 1️⃣ Full Production Function App
[![Deploy Function App](https://aka.ms/deploytoazurebutton)](https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2FThe-Coca-Cola-Company%2Fcoke-multiagent-cross-tenant-infra%2Fmain%2FDeployment%2Fbottler-agent-main.json)

### 2️⃣ Minimal Dev/Test Function App
[![Deploy Function App Minimal](https://aka.ms/deploytoazurebutton)](https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2FThe-Coca-Cola-Company%2Fcoke-multiagent-cross-tenant-infra%2Fmain%2FDeployment%2Fbottler-agent-main.json)

**📋 Required Parameters for Step 2:**
- `preDeployedStorageAccountName`: From Step 1 outputs
- `preDeployedKeyVaultName`: From Step 1 outputs  
- `preDeployedAppServicePlanName`: From Step 1 outputs
- `preDeployedVnetName`: From Step 1 outputs (if networking enabled)

---

## 📋 Prerequisites

- Active Bottler Continental Azure subscription
- Contributor or Owner permissions in the subscription
- Azure CLI installed and configured
- TCCC information (Tenant ID, App ID, Hub URL)

---

## 🎯 CLI Deployment (Advanced)

### Option 1: Complete Production Deployment

```bash
# Login to Azure with your organization account
az login --tenant yourorg.onmicrosoft.com

# Create Resource Group
az group create --name rg-bottler-agent-prod --location eastus

# STEP 1: Pre-Configuration
az deployment group create \
  --resource-group rg-bottler-agent-prod \
  --template-file bottler-agent-preconfig.json \
  --parameters \
    environment=prod \
    deployNetworking=true \
    deployAIFoundry=true \
    deployCosmosDB=true \
    vnetAddressPrefix="10.2.0.0/16" \
    tcccTenantId="<TCCC_TENANT_ID>" \
    tcccAppId="<TCCC_APP_ID>" \
    tcccHubUrl="<TCCC_HUB_URL>" \
    enableCrossTenantPeering=true \
    tcccVnetResourceId="<TCCC_VNET_RESOURCE_ID>"

# Get outputs from Step 1
STORAGE_NAME=$(az deployment group show --resource-group rg-bottler-agent-prod --name bottler-agent-preconfig --query properties.outputs.storageAccountName.value -o tsv)
KV_NAME=$(az deployment group show --resource-group rg-bottler-agent-prod --name bottler-agent-preconfig --query properties.outputs.keyVaultName.value -o tsv)
ASP_NAME=$(az deployment group show --resource-group rg-bottler-agent-prod --name bottler-agent-preconfig --query properties.outputs.appServicePlanName.value -o tsv)
VNET_NAME=$(az deployment group show --resource-group rg-bottler-agent-prod --name bottler-agent-preconfig --query properties.outputs.vnetName.value -o tsv)

# Add required secrets to Key Vault
az keyvault secret set --vault-name $KV_NAME --name storage-connection-string --value "$(az storage account show-connection-string --name $STORAGE_NAME --resource-group rg-bottler-agent-prod --query connectionString -o tsv)"

# Wait 60 seconds for permissions to propagate
sleep 60

# STEP 2: Function App Deployment
az deployment group create \
  --resource-group rg-bottler-agent-prod \
  --template-file bottler-agent-main.json \
  --parameters \
    environment=prod \
    deployNetworking=true \
    deployAIFoundry=true \
    deployCosmosDB=true \
    tcccTenantId="<TCCC_TENANT_ID>" \
    tcccAppId="<TCCC_APP_ID>" \
    tcccHubUrl="<TCCC_HUB_URL>" \
    preDeployedStorageAccountName=$STORAGE_NAME \
    preDeployedKeyVaultName=$KV_NAME \
    preDeployedAppServicePlanName=$ASP_NAME \
    preDeployedVnetName=$VNET_NAME
```

### Option 2: Development/Testing Deployment

```bash
# Login to Azure with your organization account
az login --tenant yourorg.onmicrosoft.com

# Create Resource Group
az group create --name rg-bottler-agent-dev --location eastus

# STEP 1: Pre-Configuration (minimal)
az deployment group create \
  --resource-group rg-bottler-agent-dev \
  --template-file bottler-agent-preconfig.json \
  --parameters \
    environment=dev \
    deployNetworking=true \
    deployAIFoundry=false \
    deployCosmosDB=false \
    vnetAddressPrefix="10.2.0.0/16" \
    tcccTenantId="<TCCC_TENANT_ID>" \
    tcccAppId="<TCCC_APP_ID>" \
    tcccHubUrl="<TCCC_HUB_URL>"

# Get outputs and add secrets (same as above)
# Wait 60 seconds
# Deploy Step 2 with same parameters
```

---

## 📊 Deployment Comparison

 < /dev/null |  Resource | Step 1 (Pre-Config) | Step 2 (Function App) |
|----------|---------------------|----------------------|
| Storage Account | ✅ | References Step 1 |
| Key Vault | ✅ | References Step 1 |
| App Service Plan | ✅ | References Step 1 |
| VNet + NSGs | ✅ | References Step 1 |
| AI Foundry | ✅ | References Step 1 |
| Cosmos DB | ✅ | References Step 1 |
| Function App | ❌ | ✅ Deployed |
| Role Assignments | ❌ | ✅ Deployed |

---

## 🔧 Post-Deployment Steps

### 1. Get deployment information:
```bash
# View Step 2 outputs
az deployment group show \
  --resource-group rg-bottler-agent-prod \
  --name bottler-agent-main \
  --query properties.outputs
```

### 2. Deploy Function App code:
```bash
cd ../../bottler-agent
func azure functionapp publish <FUNCTION_APP_NAME_FROM_OUTPUT>
```

### 3. Test deployment:
```bash
# Test Function App endpoint
curl https://<FUNCTION_APP_NAME>.azurewebsites.net/api/health
```

---

## ⚠️ Important Notes

1. **Mandatory Two-Stage Process**: Never skip Step 1 or combine steps
2. **Wait Time**: Always wait 60-90 seconds between stages
3. **Parameter Consistency**: Use same parameters for both stages
4. **Secret Management**: Step 1 creates secrets, Step 2 consumes them
5. **IP Ranges**: Bottler VNET uses `10.2.0.0/16` - don't change for TCCC peering

---

## 🆘 Troubleshooting

### Common Issues:

**❌ AccessToKeyVaultDenied**
- **Cause**: Deployed in single stage or insufficient wait time
- **Solution**: Redeploy using two-stage process with proper wait

**❌ Function App deployment fails**
- **Cause**: Missing Step 1 outputs or incorrect parameter names
- **Solution**: Verify Step 1 completed and use exact output names

**❌ VNet integration fails**
- **Cause**: Y1 SKU used with networking enabled
- **Solution**: Templates auto-select EP1 SKU when networking=true

---

## 🆘 Support

- **TCCC Infrastructure Team**: TCCC Emerging Technology  
- **Technical documentation**: [BOTTLER-RESOURCES-DETAIL.md](Docs/BOTTLER-RESOURCES-DETAIL.md)  
- **Network architecture**: [CROSS-TENANT-VNET-ARCHITECTURE.md](Docs/CROSS-TENANT-VNET-ARCHITECTURE.md)  
- **Security guide**: [CROSS-TENANT-AUTHENTICATION-GUIDE.md](Docs/CROSS-TENANT-AUTHENTICATION-GUIDE.md)

---

*Last updated: 2025-07-16*
*Critical deployment fix: Two-stage process to prevent Key Vault access failures*
