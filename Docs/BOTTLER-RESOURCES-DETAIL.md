# **Azure Infrastructure Specification for the Bottler Agent**

### **Document Version: 1.1**
### **Date: July 15, 2025**

> **Abstract**: This document outlines the standardized, generic Azure infrastructure for all Bottler Agents. The architecture is designed for security, scalability, and dynamic configuration, enabling seamless integration within the Coca-Cola ecosystem. For the initial Proof of Concept (POC), the configuration is scoped to a single agent.

---

## **1. Architecture Overview**

The Bottler Agent infrastructure is a secure and scalable solution deployed within the Bottler's Azure subscription. It establishes a robust foundation for communication with the central TCCC hub while ensuring resource isolation and data privacy. The core components include a dedicated Virtual Network, serverless computing via Azure Functions, and state management through Cosmos DB, eliminating the need for a separate Redis cache.

The following diagram illustrates the high-level architecture and the relationships between the resources.

---

## **2. Resource Specification**

Executing the `bottler-agent-infra.json` template will provision the following Azure resources. The naming convention for all resources is `bottler-agent-{env}-{resourceType}-{uniqueId}`.

### **2.1. Networking**

A dedicated Virtual Network (VNET) isolates the agent's resources and controls traffic flow.

*   **Virtual Network (VNET)**
    *   **Name**: `bottler-agent-{env}-vnet-{uniqueId}`
    *   **Address Space**: `10.2.0.0/16` (Provides 65,536 IP addresses)
    *   **Subnets**:
        *   `snet-functions` (`10.2.1.0/24`): For the Function App's VNET integration.
        *   `snet-private-endpoints` (`10.2.2.0/24`): For securing access to PaaS services.

*   **Network Security Groups (NSGs)**
    *   `bottler-agent-{env}-nsg-functions-{uniqueId}`: Secures the Function App subnet.
        *   **Inbound Rule**: Allow HTTPS (port 443) traffic originating from the TCCC VNET (`10.1.0.0/16`).
    *   `bottler-agent-{env}-nsg-pe-{uniqueId}`: Secures the Private Endpoints subnet (default rules apply).

*   **VNET Peering (Conditional)**
    *   **Name**: `bottler-to-tccc`
    *   **Purpose**: Establishes a direct, private connection to the TCCC VNET, enabling cross-tenant communication and traffic forwarding.

### **2.2. Storage Account**

A general-purpose storage account for the Function App's operational data, logs, and other artifacts.

*   **Name**: `stbottleragent{env}{uniqueId}`
*   **Account Type**: StorageV2 (General Purpose v2)
*   **Performance Tier**: Standard
*   **Redundancy**: Locally-Redundant Storage (Standard_LRS)
*   **Security Features**:
    *   Secure transfer required (HTTPS only).
    *   Minimum TLS version set to 1.2.
    *   Network access is restricted to specific subnets via Network ACLs.

### **2.3. Key Vault**

A secure repository for managing application secrets, keys, and certificates.

*   **Name**: `bottler-agent-{env}-kv-{uniqueId}`
*   **SKU**: Standard
*   **Access Control**: Role-Based Access Control (RBAC) is enabled for granular permissions.
*   **Integration**: The Function App accesses the vault using its Managed Identity.

### **2.4. Azure AI Foundry (Optional)**

A hub for developing and deploying AI models, primarily for financial analysis.

*   **Name**: `bottler-agent-{env}-aihub-{uniqueId}`
*   **Workspace Type**: Hub
*   **Integration**: Natively connects to the provisioned Storage Account and Key Vault.
*   **Networking**: Deployed within a managed network with outbound internet access enabled.

### **2.5. Cosmos DB (Optional)**

A serverless NoSQL database for managing the agent's state and conversation history.

*   **Name**: `bottler-agent-{env}-cosmos-{uniqueId}`
*   **API**: NoSQL (Core SQL)
*   **Capacity Mode**: Serverless
*   **Database**: `AgentStateDB` (created automatically on first use)
*   **Consistency Level**: Session
*   **Purpose**:
    *   Stores agent state and notifications.
    *   Maintains conversation history.
    *   **Note**: This component replaces the need for a dedicated Redis cache.

### **2.6. Function App**

The serverless compute core of the Bottler Agent.

*   **App Service Plan**
    *   **Name**: `bottler-agent-{env}-asp-{uniqueId}`
    *   **SKU**: Consumption Plan (Y1) - Dynamic, serverless scaling.

*   **Function App**
    *   **Name**: `bottler-agent-{env}-func-{uniqueId}`
    *   **Runtime Stack**: Python 3.11
    *   **Identity**: System-Assigned Managed Identity is enabled for secure, password-less access to other Azure resources.
    *   **Networking**:
        *   Integrated with the `snet-functions` subnet.
        *   All outbound traffic is routed through the VNET.

### **2.7. Application Insights**

Provides comprehensive monitoring, logging, and diagnostics for the Function App.

*   **Name**: `bottler-agent-{env}-ai-{uniqueId}`
*   **Type**: Web Application Monitoring
*   **Integration**: Linked to the Function App for automatic instrumentation.

---

## **3. Security and Identity**

Security is managed through a **System-Assigned Managed Identity** on the Function App. This identity is automatically granted the necessary RBAC permissions to access:
*   **Azure Key Vault**: To read secrets.
*   **Azure Storage Account**: To read/write data.
*   **TCCC Hub**: To authenticate and communicate via its App Registration.

---

## **4. Dynamic Configuration**

The agent's behavior is controlled via Application Settings, allowing for dynamic activation of different bottler integrations without infrastructure changes.

*   **Key Application Settings**:
    *   `AGENT_TYPE`: Set to `"BOTTLER_AGENT"`.
    *   `ENABLED_SPOKES`: A comma-separated list of active agents (e.g., `"agent-one"`).
    *   `MOCK_UNAVAILABLE_SPOKES`: If `true`, simulates responses from disabled spokes during development.
    *   `TCCC_TENANT_ID`: The tenant ID for the central TCCC services.
    *   `TCCC_APP_ID`: The Application ID for the TCCC App Registration.
    *   `TCCC_HUB_URL`: The endpoint for the central TCCC Function App.
    *   `KEY_VAULT_URI`: The URI of the provisioned Key Vault.

*   **Example: POC Configuration (Single Agent)**
    ```json
    {
      "ENABLED_SPOKES": "agent-one",
      "MOCK_UNAVAILABLE_SPOKES": "true"
    }
    ```

*   **Example: Multi-Bottler Production Configuration**
    ```json
    {
      "ENABLED_SPOKES": "agent-one,agent-two,agent-three",
      "MOCK_UNAVAILABLE_SPOKES": "false"
    }
    ```

---

## **5. Monthly Cost Estimation**

The following table provides an estimated monthly cost. Actual costs will vary based on usage, region, and data volume.

| Resource          | SKU / Tier     | Estimated Monthly Cost (USD) | Notes                                    |
| ----------------- | -------------- | ---------------------------- | ---------------------------------------- |
| Function App      | Consumption    | ~$20                         | Based on 1 million executions.           |
| Storage Account   | Standard LRS   | ~$25                         | Based on 100 GB of data stored.          |
| Key Vault         | Standard       | ~$5                          | Based on 10,000 operations.              |
| Cosmos DB         | Serverless     | ~$25                         | Based on 1 million Request Units (RUs).  |
| AI Foundry        | Pay-as-you-go  | ~$100 - $500+                | Highly variable based on usage.          |
| Application Insights | Basic       | ~$5                          | Based on 100 MB of log ingestion.        |
| Virtual Network   | -              | $0                           | VNET itself is free; peering has costs.  |
| **TOTAL ESTIMATE**|                | **~$180 - $700+ / month**    |                                          |

---

## **6. Key Architectural Highlights**

1.  **No Redis Dependency**: State management is fully handled by Cosmos DB, simplifying the architecture.
2.  **Generic and Reusable**: A single, unified template is used for all bottler deployments.
3.  **Dynamic Spoke Management**: Agents can be enabled or disabled via configuration, providing operational flexibility.
4.  **Secure by Design**: Implements hub-spoke security patterns and cross-tenant access controls.
5.  **Cloud-Native and Serverless**: Leverages serverless and PaaS services for scalability and cost-efficiency.

*This configuration provides a complete and secure infrastructure for any Bottler agent in the Coca-Cola ecosystem*