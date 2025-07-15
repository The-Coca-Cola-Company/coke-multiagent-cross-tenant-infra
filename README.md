# 🏢 Bottler Agent - Azure Deployment Guide

![Cross Tenant Architecture](Deployment/img/Image%20cross.png)

## 🚀 One-Click Deployment Options

Choose your deployment strategy and launch directly from Azure Portal:

### 1️⃣ Full Production Deployment
[![Deploy Full](https://aka.ms/deploytoazurebutton)](https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2FThe-Coca-Cola-Company%2Fcoke-multiagent-cross-tenant-infra%2Fmain%2FDeployment%2Fbottler-agent-infra-compliant.json)

Includes: VNET, Key Vault, AI Foundry, Cosmos DB, Function App, Insights.

---

### 2️⃣ Minimal Dev/Test Deployment
[![Deploy Minimal](https://aka.ms/deploytoazurebutton)](https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2FThe-Coca-Cola-Company%2Fcoke-multiagent-cross-tenant-infra%2Fmain%2FDeployment%2Fbottler-agent-infra-compliant.json)

Same template — use parameters to disable CosmosDB/AI Foundry.

---

### 3️⃣ Custom Parameterized Deployment
[![Deploy Custom](https://aka.ms/deploytoazurebutton)](https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2FThe-Coca-Cola-Company%2Fcoke-multiagent-cross-tenant-infra%2Fmain%2FDeployment%2Fbottler-agent-infra-compliant.json)

Edit parameters in Azure Portal before creating the deployment.

---

> ℹ️ All three buttons use the **same template** (`bottler-agent-infra.json`). You can adjust the parameters during deployment to match Full, Minimal, or Custom behavior.

## 📋 Prerequisites

- Active Bottler Continental Azure subscription
- Contributor or Owner permissions in the subscription
- Azure CLI installed and configured
- TCCC information (Tenant ID, App ID, Hub URL)

## 🚀 Deployment Options

### Option 1: Complete Infrastructure (Recommended for Production)

Includes **ALL** resources needed for a complete production environment.

```bash
# Login to Azure with your organization account
az login --tenant yourorg.onmicrosoft.com

# Create Resource Group
az group create --name rg-bottler-agent-prod --location eastus

# Complete deployment with ALL resources
az deployment group create \
  --resource-group rg-bottler-agent-prod \
  --template-file bottler-agent-infra.json \
  --parameters \
    environment=prod \
    deployNetworking=true \
    deployAIFoundry=true \
    deployCosmosDB=true \
    vnetAddressPrefix="10.2.0.0/16" \
    tcccTenantId="<TCCC_TENANT_ID>" \
    tcccAppId="<TCCC_APP_ID>" \
    tcccHubUrl="<TCCC_HUB_URL>" \
    enabledSpokes="arca-agent" \
    mockUnavailableSpokes=true \
    enableCrossTenantPeering=true \
    tcccVnetResourceId="<TCCC_VNET_RESOURCE_ID>"
```

**Included resources:**
- ✅ Virtual Network with subnets and NSGs
- ✅ Storage Account with network ACLs
- ✅ Key Vault
- ✅ Azure AI Foundry (AI Hub)
- ✅ Cosmos DB (Database)
- ✅ Function App with VNET Integration
- ✅ Application Insights
- ✅ Configuration for VNET Peering

---

### Option 2: Base Infrastructure (Development/Testing)

Minimal infrastructure **WITHOUT** Cosmos DB or AI Foundry to reduce costs.

```bash
# Login to Azure with your organization account
az login --tenant yourorg.onmicrosoft.com

# Create Resource Group
az group create --name rg-bottler-agent-dev --location eastus

# Deploy without Cosmos DB or AI Foundry
az deployment group create \
  --resource-group rg-bottler-agent-dev \
  --template-file bottler-agent-infra.json \
  --parameters \
    environment=dev \
    deployNetworking=true \
    deployAIFoundry=false \
    deployCosmosDB=false \
    vnetAddressPrefix="10.2.0.0/16" \
    tcccTenantId="<TCCC_TENANT_ID>" \
    tcccAppId="<TCCC_APP_ID>" \
    tcccHubUrl="<TCCC_HUB_URL>"
```

**Included resources:**
- ✅ Virtual Network with subnets and NSGs
- ✅ Storage Account
- ✅ Key Vault
- ✅ Function App with VNET Integration
- ✅ Application Insights
- ❌ Azure AI Foundry (not included)
- ❌ Cosmos DB (not included)

**Ideal for:**
- Local development
- Integration testing
- Low-cost environments

---

### Option 3: Custom Deployment

Select exactly which resources you need using parameters.

```bash
# Login to Azure with your organization account
az login --tenant yourorg.onmicrosoft.com

# Create Resource Group
az group create --name rg-bottler-agent-custom --location eastus

# Custom deployment with specific parameters
az deployment group create \
  --resource-group rg-bottler-agent-custom \
  --template-file bottler-agent-infra.json \
  --parameters \
    environment=test \
    deployNetworking=<true|false> \
    deployAIFoundry=<true|false> \
    deployCosmosDB=<true|false> \
    vnetAddressPrefix="10.2.0.0/16" \
    tcccTenantId="<TCCC_TENANT_ID>" \
    tcccAppId="<TCCC_APP_ID>" \
    tcccHubUrl="<TCCC_HUB_URL>" \
    enabledSpokes="arca-agent" \
    mockUnavailableSpokes=true \
    enableCrossTenantPeering=<true|false> \
    tcccVnetResourceId="<TCCC_VNET_ID_OPCIONAL>"
```

**Configurable parameters:**

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `environment` | string | dev | Environment: dev, test, prod |
| `deployNetworking` | bool | true | Create VNET and network components |
| `deployAIFoundry` | bool | true | Include Azure AI Foundry |
| `deployCosmosDB` | bool | true | Include Cosmos DB |
| `vnetAddressPrefix` | string | 10.2.0.0/16 | VNET IP range |
| `enableCrossTenantPeering` | bool | false | Enable peering with TCCC |
| `tcccVnetResourceId` | string | "" | TCCC VNET ID (if peering) |

## 📊 Options Comparison

| Resource | Option 1 (Full) | Option 2 (Base) | Option 3 (Custom) |
|---------|-----------------|-----------------|-------------------|
| VNET + NSGs | ✅ | ✅ | Configurable |
| Storage Account | ✅ | ✅ | ✅ Always |
| Key Vault | ✅ | ✅ | ✅ Always |
| Function App | ✅ | ✅ | ✅ Always |
| App Insights | ✅ | ✅ | ✅ Always |
| AI Foundry | ✅ | ❌ | Configurable |
| Cosmos DB | ✅ | ❌ | Configurable |
| VNET Peering | ✅ | Optional | Configurable |
| **Estimated Cost** | $500-700/month | $50-100/month | Variable |

## 🔧 Post-Deployment

### 1. Get deployment information:
```bash
# View deployment outputs
az deployment group show \
  --resource-group rg-bottler-agent-prod \
  --name bottler-agent-infra \
  --query properties.outputs
```

### 2. Configure App Registration (if it doesn't exist):
```bash
# Create App Registration for Bottler
az ad app create \
  --display-name "Bottler-Financial-Agent" \
  --sign-in-audience AzureADMultipleOrgs
```

### 3. Deploy Function App code:
```bash
cd ../../bottler-agent
func azure functionapp publish <NOMBRE_FUNCTION_APP>
```

### 4. Configure VNET Peering (if applicable):
```bash
# Get Bottler VNET ID
Bottler_VNET_ID=$(az network vnet show \
  --resource-group rg-bottler-agent-prod \
  --name bottler-agent-prod-vnet-* \
  --query id -o tsv)

# Share with TCCC team to establish peering
```

## ⚠️ Important Considerations

1. **IP Ranges**: Bottler VNET uses 10.2.0.0/16. DO NOT change if planning peering with TCCC.
2. **Unique names**: The template automatically adds a unique suffix.
3. **Secrets**: Never include secrets in parameters. Use Key Vault post-deployment.
4. **Costs**: Review cost estimation before choosing an option.

## 🆘 Support

- **TCCC Infrastructure Team**: TCCC Emerging Technology  
- **Technical documentation**: [Bottler-RESOURCES-DETAIL.md](Docs/BOTTLER-RESOURCES-DETAIL.md)  
- **Network architecture**: [CROSS-TENANT-VNET-ARCHITECTURE.pdf](Docs/CROSS-TENANT-VNET-ARCHITECTURE.pdf)  
- **Technical Security**: [CROSS-TENANT-AUTHENTICATION-GUIDE.pdf](Docs/CROSS-TENANT-AUTHENTICATION-GUIDE.pdf)  
- **Help**: [MULTI-TENANT-IMPORTANT-ARCHITECTURE-CLARIFICATION.pdf](Docs/MULTI-TENANT-IMPORTANT-ARCHITECTURE-CLARIFICATION.pdf)



---
*Last updated: 2025-07-13*

