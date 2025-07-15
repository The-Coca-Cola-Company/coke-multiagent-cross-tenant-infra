# 🏢 Generic Bottler - Azure Deployment Guide

This guide is for deploying a bottler agent in your organization's Azure subscription/tenant. This is a generic template that can be customized for any bottler organization.

> **Note**: Replace placeholder values with your actual organization information throughout this guide.

## 📋 Prerequisites

- Active Azure subscription in your bottler organization
- Contributor or Owner permissions in the subscription
- Azure CLI installed and configured
- TCCC integration information (provided by TCCC team)

## 🚀 Deployment Options

### Option 1: Complete Infrastructure (Recommended for Production)

Includes **ALL** resources needed for a complete production environment.

[![Deploy to Azure](https://aka.ms/deploytoazurebutton)](https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2Fyour-org%2Fyour-repo%2Fmain%2Fbottler-agent-infra.json)

```bash
# Login to Azure with your organization account
az login --tenant yourbottler.onmicrosoft.com

# Set your subscription
az account set --subscription "Your-Subscription-Name"

# Create Resource Group
az group create \
  --name rg-bottler-agent-prod \
  --location eastus

# Complete deployment with ALL resources
az deployment group create \
  --resource-group rg-bottler-agent-prod \
  --template-file bottler-agent-infra-compliant.json \
  --parameters \
    environment=prod \
    deployNetworking=true \
    deployAIFoundry=true \
    deployCosmosDB=true \
    vnetAddressPrefix="10.X.0.0/16" \
    tcccTenantId="<PROVIDED_BY_TCCC>" \
    tcccAppId="<PROVIDED_BY_TCCC>" \
    tcccHubUrl="<PROVIDED_BY_TCCC>" \
    enableCrossTenantPeering=true \
    tcccVnetResourceId="<PROVIDED_BY_TCCC>"
```

**Included resources:**
- ✅ Virtual Network with subnets and NSGs
- ✅ Storage Account with network ACLs
- ✅ Key Vault for secrets management
- ✅ Azure AI Foundry (AI Hub)
- ✅ Cosmos DB (State management)
- ✅ Function App with VNET Integration
- ✅ Application Insights
- ✅ Cross-tenant VNET Peering capability

**Monthly Cost Estimate**: $500-700

---

### Option 2: Base Infrastructure (Development/Testing)

Minimal infrastructure **WITHOUT** Cosmos DB or AI Foundry to reduce costs.

```bash
# Login to Azure with your organization account
az login --tenant yourbottler.onmicrosoft.com

# Create Resource Group
az group create \
  --name rg-bottler-agent-dev \
  --location eastus

# Deploy without Cosmos DB or AI Foundry
az deployment group create \
  --resource-group rg-bottler-agent-dev \
  --template-file bottler-agent-infra-compliant.json \
  --parameters \
    environment=dev \
    deployNetworking=true \
    deployAIFoundry=false \
    deployCosmosDB=false \
    vnetAddressPrefix="10.X.0.0/16" \
    tcccTenantId="<PROVIDED_BY_TCCC>" \
    tcccAppId="<PROVIDED_BY_TCCC>" \
    tcccHubUrl="<PROVIDED_BY_TCCC>"
```

**Included resources:**
- ✅ Virtual Network with subnets and NSGs
- ✅ Storage Account
- ✅ Key Vault
- ✅ Function App with VNET Integration
- ✅ Application Insights
- ❌ Azure AI Foundry (not included)
- ❌ Cosmos DB (not included)

**Monthly Cost Estimate**: $50-100

**Ideal for:**
- Proof of concepts
- Development environments
- Integration testing
- Cost-sensitive deployments

---

### Option 3: Custom Deployment

Select exactly which resources you need using parameters.

```bash
# Custom deployment with specific parameters
az deployment group create \
  --resource-group rg-bottler-agent-custom \
  --template-file bottler-agent-infra-compliant.json \
  --parameters \
    environment=test \
    deployNetworking=<true|false> \
    deployAIFoundry=<true|false> \
    deployCosmosDB=<true|false> \
    vnetAddressPrefix="10.X.0.0/16" \
    tcccTenantId="<PROVIDED_BY_TCCC>" \
    tcccAppId="<PROVIDED_BY_TCCC>" \
    tcccHubUrl="<PROVIDED_BY_TCCC>" \
    enableCrossTenantPeering=<true|false> \
    tcccVnetResourceId="<OPTIONAL_TCCC_VNET_ID>"
```

## 📊 Deployment Parameters

| Parameter | Type | Default | Description | Example |
|-----------|------|---------|-------------|---------|
| `environment` | string | dev | Environment name | prod, test, dev |
| `location` | string | [resourceGroup().location] | Azure region | eastus, westeurope |
| `deployNetworking` | bool | true | Deploy VNET and network components | true |
| `deployAIFoundry` | bool | true | Include Azure AI Foundry | true/false |
| `deployCosmosDB` | bool | true | Include Cosmos DB | true/false |
| `vnetAddressPrefix` | string | 10.2.0.0/16 | VNET IP range (coordinate with TCCC) | 10.3.0.0/16 |
| `tcccTenantId` | string | required | TCCC Azure AD Tenant ID | xxxxxxxx-xxxx-... |
| `tcccAppId` | string | required | TCCC App Registration ID | xxxxxxxx-xxxx-... |
| `tcccHubUrl` | string | required | TCCC Hub Function URL | https://... |
| `enableCrossTenantPeering` | bool | false | Enable VNET peering with TCCC | true/false |
| `tcccVnetResourceId` | string | "" | TCCC VNET Resource ID (if peering) | /subscriptions/... |

## 🔧 Post-Deployment Configuration

### 1. Retrieve Deployment Outputs

```bash
# Get all deployment outputs
az deployment group show \
  --resource-group rg-bottler-agent-prod \
  --name bottler-agent-infra-compliant \
  --query properties.outputs -o json

# Get specific values
FUNCTION_APP_NAME=$(az deployment group show \
  --resource-group rg-bottler-agent-prod \
  --name bottler-agent-infra-compliant \
  --query properties.outputs.functionAppName.value -o tsv)

KEY_VAULT_NAME=$(az deployment group show \
  --resource-group rg-bottler-agent-prod \
  --name bottler-agent-infra-compliant \
  --query properties.outputs.keyVaultName.value -o tsv)
```

### 2. Configure App Registration

If you don't already have an App Registration:

```bash
# Create App Registration for your bottler
az ad app create \
  --display-name "YourBottler-Agent-MultiTenant" \
  --sign-in-audience AzureADMultipleOrgs

# Get the Application ID
APP_ID=$(az ad app list \
  --display-name "YourBottler-Agent-MultiTenant" \
  --query "[0].appId" -o tsv)

# Create a client secret
az ad app credential reset \
  --id $APP_ID \
  --credential-description "TCCC-Integration" \
  --years 2
```

### 3. Store Secrets in Key Vault

```bash
# Store application secrets
az keyvault secret set \
  --vault-name $KEY_VAULT_NAME \
  --name "app-client-secret" \
  --value "<YOUR_CLIENT_SECRET>"

az keyvault secret set \
  --vault-name $KEY_VAULT_NAME \
  --name "storage-connection-string" \
  --value "<STORAGE_CONNECTION_STRING>"

az keyvault secret set \
  --vault-name $KEY_VAULT_NAME \
  --name "appinsights-connection-string" \
  --value "<APP_INSIGHTS_CONNECTION_STRING>"
```

### 4. Deploy Function App Code

```bash
# Navigate to your function app directory
cd path/to/your/bottler-agent

# Install dependencies
pip install -r requirements.txt

# Deploy to Azure
func azure functionapp publish $FUNCTION_APP_NAME --python
```

### 5. Configure VNET Peering (if applicable)

```bash
# Get your VNET ID
BOTTLER_VNET_ID=$(az network vnet show \
  --resource-group rg-bottler-agent-prod \
  --name bottler-agent-prod-vnet-* \
  --query id -o tsv)

echo "Share this VNET ID with TCCC team: $BOTTLER_VNET_ID"

# After receiving confirmation from TCCC, verify peering
az network vnet peering show \
  --resource-group rg-bottler-agent-prod \
  --vnet-name bottler-agent-prod-vnet-* \
  --name bottler-to-tccc \
  --query peeringState
```

## 📊 Cost Optimization Tips

1. **Development Environments**
   - Set `deployAIFoundry=false` and `deployCosmosDB=false`
   - Use consumption plan for Function App
   - Implement auto-shutdown policies

2. **Production Environments**
   - Enable autoscaling for Function App
   - Use reserved capacity for consistent workloads
   - Implement cost alerts and budgets

3. **Monitor Usage**
   ```bash
   # Set up cost alert
   az monitor action-group create \
     --resource-group rg-bottler-agent-prod \
     --name CostAlertGroup \
     --short-name CostAlert
   ```

## ⚠️ Important Considerations

### Network Configuration
- **IP Range**: Coordinate with TCCC to avoid conflicts
  - TCCC uses: 10.1.0.0/16
  - Default bottler: 10.2.0.0/16
  - Your range: 10.X.0.0/16 (where X is assigned by TCCC)

### Security
- All secrets must be stored in Key Vault
- Enable managed identity for all services
- Review NSG rules based on your security requirements
- Enable Azure Defender for critical resources

### Compliance
- Ensure data residency requirements are met
- Review and apply necessary Azure Policies
- Enable diagnostic logging for all resources
- Implement backup and disaster recovery

## 🆘 Troubleshooting

### Common Issues:

1. **Deployment Fails**
   ```bash
   # Get detailed error
   az deployment group show \
     --resource-group rg-bottler-agent-prod \
     --name bottler-agent-infra-compliant \
     --query 'properties.error'
   ```

2. **Function App Not Starting**
   - Check VNET integration settings
   - Verify Key Vault access permissions
   - Review Application Insights for errors

3. **Cross-Tenant Authentication Issues**
   - Verify App Registration configuration
   - Check tenant IDs and app IDs
   - Review [Authentication Guide](./CROSS-TENANT-AUTHENTICATION-GUIDE.md)

## 📚 Additional Resources

- [Bottler Onboarding Checklist](./BOTTLER-ONBOARDING-CHECKLIST.md)
- [Cross-Tenant Authentication Guide](./CROSS-TENANT-AUTHENTICATION-GUIDE.md)
- [Troubleshooting Guide](./TROUBLESHOOTING-CROSS-TENANT.md)
- [Architecture Overview](./IMPORTANT-ARCHITECTURE-CLARIFICATION.md)
- [Network Architecture](./CROSS-TENANT-VNET-ARCHITECTURE.md)

## 📞 Support Contacts

### Your Organization:
- **Infrastructure Team**: infra@yourbottler.com
- **Security Team**: security@yourbottler.com
- **Azure Support**: Via Azure Portal

### TCCC Integration:
- **Integration Team**: integration@tccc.com
- **Technical Support**: support@tccc.com
- **Emergency**: On-call via PagerDuty

---
*Template Version: 1.0*
*Last Updated: December 2024*

> **Next Steps**: After successful deployment, follow the [Bottler Onboarding Checklist](./BOTTLER-ONBOARDING-CHECKLIST.md) for complete integration.