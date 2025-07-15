# 🔧 Cross-Tenant Troubleshooting Guide

This guide helps diagnose and resolve common issues when deploying and operating the multi-tenant TCCC-Bottler architecture.

## 🚨 Common Issues and Solutions

### 1. VNET Peering Issues

#### 🔴 Problem: "Peering status shows as 'Disconnected'"

**Symptoms:**
- Peering shows "Initiated" on one side, "Disconnected" on the other
- Cannot ping between VNETs
- Function Apps cannot communicate

**Solutions:**

```bash
# Check peering status on both sides
# TCCC side
az network vnet peering show \
  --resource-group rg-tccc-hub-prod \
  --vnet-name tccc-hub-vnet \
  --name tccc-to-bottler \
  --query peeringState

# Bottler side
az network vnet peering show \
  --resource-group rg-bottler-agent-prod \
  --vnet-name bottler-agent-vnet \
  --name bottler-to-tccc \
  --query peeringState
```

**Fix:**
1. Delete peering on both sides
2. Recreate peering connections
3. Ensure both sides are created within 5 minutes

#### 🔴 Problem: "Address space overlap detected"

**Error Message:**
```
The address space '10.1.0.0/16' in virtual network 'bottler-vnet' overlaps with '10.1.0.0/16' in peered virtual network 'tccc-vnet'
```

**Solution:**
- TCCC must use: `10.1.0.0/16`
- Bottler must use: `10.2.0.0/16`
- Never change these ranges after deployment

### 2. Authentication Failures

#### 🔴 Problem: "401 Unauthorized" when calling cross-tenant

**Symptoms:**
```json
{
  "error": {
    "code": "Unauthorized",
    "message": "IDX10214: Audience validation failed"
  }
}
```

**Diagnostic Steps:**

```bash
# 1. Decode the JWT token
echo $TOKEN | cut -d. -f2 | base64 --decode | jq

# 2. Check the audience (aud) claim
# Should match the target app's ID

# 3. Verify issuer (iss) claim
# Should be: https://sts.windows.net/{tenant-id}/
```

**Solutions:**

```python
# Correct token request
scope = f"{target_app_id}/.default"  # NOT "https://graph.microsoft.com/.default"

# Correct validation
expected_audience = YOUR_APP_ID  # The receiving app's ID
expected_issuer = f"https://sts.windows.net/{sender_tenant_id}/"
```

#### 🔴 Problem: "AADSTS700016: Application not found in directory"

**Causes:**
1. Service principal not created in target tenant
2. Wrong tenant ID in configuration
3. App registration deleted

**Fix:**

```bash
# Verify app exists in correct tenant
az ad app show --id <APP_ID>

# Create service principal if missing
az ad sp create --id <APP_ID>

# Verify tenant ID
az account show --query tenantId
```

### 3. Network Connectivity Issues

#### 🔴 Problem: "Connection timeout" between Function Apps

**Diagnostic Tools:**

```bash
# 1. Test NSG rules
az network nsg rule list \
  --resource-group rg-bottler-agent-prod \
  --nsg-name bottler-nsg-functions

# 2. Check effective routes
az network nic show-effective-route-table \
  --resource-group rg-bottler-agent-prod \
  --name <nic-name>

# 3. Verify DNS resolution
nslookup tccc-hub-func.azurewebsites.net
```

**Common Fixes:**

```bash
# Update NSG rule to allow TCCC traffic
az network nsg rule update \
  --resource-group rg-bottler-agent-prod \
  --nsg-name bottler-nsg-functions \
  --name AllowTCCC \
  --source-address-prefixes "10.1.0.0/16" \
  --destination-port-ranges 443
```

#### 🔴 Problem: "Private endpoint connection failed"

**Symptoms:**
- Storage account not accessible
- Key Vault connection errors
- Cosmos DB unreachable

**Solution:**

```bash
# Verify private endpoint status
az network private-endpoint show \
  --resource-group rg-bottler-agent-prod \
  --name pe-storage

# Check DNS configuration
nslookup yourstorage.blob.core.windows.net

# Should resolve to private IP (10.x.x.x)
```

### 4. Function App Configuration Issues

#### 🔴 Problem: "Function runtime is unreachable"

**Check List:**
1. VNET integration enabled?
2. Correct subnet selected?
3. Storage account accessible?
4. Application Insights connected?

```bash
# Verify VNET integration
az functionapp vnet-integration list \
  --resource-group rg-bottler-agent-prod \
  --name bottler-func-app

# Check app settings
az functionapp config appsettings list \
  --resource-group rg-bottler-agent-prod \
  --name bottler-func-app \
  --output table
```

#### 🔴 Problem: "Key Vault reference not resolved"

**Error:**
```
@Microsoft.KeyVault(SecretUri=https://kv.vault.azure.net/secrets/secret-name/)
```

**Fix:**

```bash
# Grant Key Vault access
az keyvault set-policy \
  --name bottler-kv \
  --object-id <function-app-identity> \
  --secret-permissions get list

# Verify managed identity
az functionapp identity show \
  --resource-group rg-bottler-agent-prod \
  --name bottler-func-app
```

### 5. Cross-Tenant Service Principal Issues

#### 🔴 Problem: "Insufficient privileges to complete the operation"

**When Creating Cross-Tenant Access:**

```bash
# Check your permissions
az ad signed-in-user show --query objectId

# Need at least:
# - Application Administrator
# - Cloud Application Administrator
# - Global Administrator (for some operations)
```

**Solution:**
Contact Azure AD administrator to:
1. Grant necessary roles
2. Create service principals
3. Configure cross-tenant access settings

### 6. Deployment Failures

#### 🔴 Problem: "Deployment failed with multiple errors"

**Common ARM Template Errors:**

```json
{
  "code": "InvalidTemplateDeployment",
  "message": "The template deployment failed because of policy violation"
}
```

**Diagnostic Commands:**

```bash
# Get detailed error
az deployment group show \
  --resource-group rg-bottler-agent-prod \
  --name deployment-name \
  --query properties.error

# List all operations
az deployment operation group list \
  --resource-group rg-bottler-agent-prod \
  --name deployment-name \
  --query "[?properties.provisioningState=='Failed']"
```

## 🛠️ Diagnostic Scripts

### Complete Health Check Script:

```bash
#!/bin/bash

echo "=== Cross-Tenant Health Check ==="

# Check Resource Groups
echo "1. Checking Resource Groups..."
az group show --name rg-bottler-agent-prod &>/dev/null && echo "✓ Bottler RG exists" || echo "✗ Bottler RG missing"

# Check VNET Peering
echo "2. Checking VNET Peering..."
PEERING_STATE=$(az network vnet peering show \
  --resource-group rg-bottler-agent-prod \
  --vnet-name bottler-agent-vnet \
  --name bottler-to-tccc \
  --query peeringState -o tsv 2>/dev/null)
[[ "$PEERING_STATE" == "Connected" ]] && echo "✓ Peering connected" || echo "✗ Peering issue: $PEERING_STATE"

# Check Function App
echo "3. Checking Function App..."
FUNC_STATE=$(az functionapp show \
  --resource-group rg-bottler-agent-prod \
  --name bottler-func-app \
  --query state -o tsv 2>/dev/null)
[[ "$FUNC_STATE" == "Running" ]] && echo "✓ Function App running" || echo "✗ Function App issue: $FUNC_STATE"

# Check Authentication
echo "4. Testing Authentication..."
# Add token acquisition and validation test

echo "=== Health Check Complete ==="
```

### Network Connectivity Test:

```python
import socket
import ssl
import requests
from urllib.parse import urlparse

def test_connectivity(url, timeout=5):
    """Test network connectivity to endpoint"""
    try:
        parsed = urlparse(url)
        
        # DNS resolution
        ip = socket.gethostbyname(parsed.hostname)
        print(f"✓ DNS resolved {parsed.hostname} to {ip}")
        
        # TCP connection
        sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        sock.settimeout(timeout)
        result = sock.connect_ex((ip, 443))
        sock.close()
        
        if result == 0:
            print(f"✓ TCP connection successful")
        else:
            print(f"✗ TCP connection failed: {result}")
            
        # HTTPS request
        response = requests.get(url, timeout=timeout)
        print(f"✓ HTTPS request successful: {response.status_code}")
        
    except Exception as e:
        print(f"✗ Connectivity test failed: {str(e)}")

# Test endpoints
test_connectivity("https://tccc-hub-func.azurewebsites.net/api/health")
```

## 📊 Monitoring Queries

### Application Insights Queries:

```kusto
// Cross-tenant communication failures
requests
| where success == false
| where customDimensions.SourceTenant != customDimensions.TargetTenant
| summarize 
    FailureCount = count(), 
    FailureRate = (count() * 100.0) / count()
    by bin(timestamp, 5m), resultCode
| render timechart

// Authentication errors
traces
| where message contains "Authentication" or message contains "Unauthorized"
| where severityLevel >= 3
| project timestamp, message, customDimensions
| order by timestamp desc

// Network timeouts
dependencies
| where success == false
| where duration > 30000
| where type == "HTTP"
| summarize 
    TimeoutCount = count(),
    AvgDuration = avg(duration)
    by bin(timestamp, 5m), target
```

## 🚀 Performance Optimization

### Slow Cross-Tenant Communication:

1. **Enable Connection Pooling:**
```python
session = requests.Session()
adapter = requests.adapters.HTTPAdapter(
    pool_connections=10,
    pool_maxsize=50,
    max_retries=3
)
session.mount('https://', adapter)
```

2. **Implement Token Caching:**
```python
from functools import lru_cache
from datetime import datetime, timedelta

@lru_cache(maxsize=128)
def get_cached_token(tenant_id, client_id):
    token = acquire_token()
    return token, datetime.now() + timedelta(hours=1)
```

3. **Use Regional Deployments:**
- Deploy TCCC Hub and Bottler agents in same region
- Use Azure Front Door for global distribution
- Implement geo-redundancy for critical components

## 📋 Escalation Matrix

| Issue Type | First Contact | Escalation | Critical |
|------------|--------------|------------|----------|
| Network/VNET | Network Admin | Azure Support | Network Architect |
| Authentication | Identity Team | Security Team | CISO |
| Function App | DevOps Team | Platform Team | Site Reliability |
| ARM Templates | Cloud Team | Azure TAM | Microsoft Support |

## 🆘 Getting Help

### Before Escalating:
1. Collect all error messages
2. Run diagnostic scripts
3. Check monitoring dashboards
4. Review recent changes

### Information to Provide:
- Subscription IDs (both tenants)
- Resource Group names
- Exact error messages
- Time of occurrence
- Steps to reproduce
- Diagnostic script output

---
*Last updated: December 2024*