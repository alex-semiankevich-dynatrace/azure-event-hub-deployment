# Create ARM template and UI Definition for Event Hub deployment

## Description

Deploy Event Hub namespaces across multiple Azure locations with resource groups, standardized config for Dynatrace log/event ingestion, and proper RBAC assignments for the Dynatrace monitoring service principal. Follow Azure CAF and WAF naming conventions for the resources we provision.

Provide a UI definition with the ARM template, see: https://github.com/dynatrace-oss/cloud-snippets/tree/main/azure/azure-activation-templates

## Open Questions

- Do we have the dtMonitoringServicePrincipalId (object id), or only the app id / client id? For role assignment, we need object id.
  ```bash
  # Get the Service Principal Object ID from the App ID
  az ad sp show --id <app-id> --query id -o tsv
  ```

## Acceptance Criteria

### ✅ Support of Subscription and Management Group scope

- For the Management Group scope, a subscription needs to be selected.

### ✅ Resource Group Creation

- Create a resource group in each specified location
- Naming convention: rg-dt-{dtTenantId}-{location}
- Example: rg-dt-abc12345-eastus, rg-dt-abc12345-westeurope
- Resource groups must be created before namespaces

### ✅ Namespace Deployment

- Create Event Hub namespace in each location's resource group
- Naming convention: evhns-dt-{dtTenantId}-{location}
- Example: evhns-dt-abc12345-eastus, evhns-dt-abc12345-westeurope
- Tag each namespace with `dt-log-ingest-activated: {monitoring-config-id}`
- Configure auto-inflate with configurable max throughput units
- Support additional optional tags?:
  - environment (e.g., prod, dev, test)
  - owner or managed-by
  - cost-center (optional)

### ✅ Event Hubs per Namespace

**dt-logs-evh:**

- Naming convention: dt-logs-evh
- Default: 4 partitions
- Configurable via parameter (1-32)

**dt-events-evh:**

- Naming convention: dt-events-evh
- Default: 1 partition
- Parameter validation to allow only 1 or 2

### ✅ RBAC Assignment

- Assign the "Azure Event Hubs Data Receiver" role to the Dynatrace monitoring service principal
- Scope: Resource group level (covers all Event Hubs in the RG)
- Service principal must be provided as a parameter (object ID)
- Role assignment must depend on resource group creation
- Follow the principle of least privilege (read-only access)

### ✅ Template Parameters

```
- locations (array) - List of Azure locations
- dtTenantId (string) - Dynatrace tenant ID for naming
- dtConfigId (string) - Monitoring ID for tagging and autodiscovery
- environment (string) - Environment identifier (prod/dev/test, default: prod)
- dtMonitoringServicePrincipalId (string) - Service principal object ID
- dtLogsPartitionCount (int, default: 8)
- dtEventsPartitionCount (int, 1 or 2, default: 2)
- skuName (Basic/Standard/Premium, default: Standard) - Unclear if we should support Basic
- skuCapacity (int, 1-20, default: 1) - This is the minimum/baseline TUs
- maximumThroughputUnits (int, default: 10) - This is the maximum TUs autoinflate can scale up to
- messageRetentionInDays (int, 1-7, default: 1)
- zoneRedundant (bool, default: false) - Can be activated for Standard or Premium SKUs
- tags (object, optional) - Additional custom tags
```

### ✅ Template Outputs

- **ONLY** return an array of deployed namespace names (no sensitive data like connectionStrings).
- Format: ["evhns-dt-abc12345-eastus", "evhns-dt-abc12345-westeurope", ...]

## Technical Notes

### Role Definition

- Built-in role: [Azure Event Hubs Data Receiver](https://learn.microsoft.com/en-us/azure/role-based-access-control/built-in-roles/analytics#azure-event-hubs-data-receiver)
- Role ID: a638d3c7-ab3a-418d-83e6-5f17a39d4fde
- Scope: /subscriptions/{subscriptionId}/resourceGroups/rg-dt-{dtTenantId}-{location}

### Deployment Structure

This requires a **subscription-level deployment** since resource groups are being created.

Template type: Microsoft.Resources/deployments at subscription scope

### Sample Deployment

```bash
# Production deployment
az deployment sub create \
  --location eastus \
  --template-file template.json \
  --parameters dtTenantId=gmg80500 \
               dtConfigId="cfc78e0e-a116-3289-bba1-ac6ad7e81c1f" \
               locations='["eastus","westeurope"]' \
               dtMonitoringServicePrincipalId="<service-principal-object-id>" 

# Custom configuration
az deployment sub create \
  --location eastus \
  --template-file template.json \
  --parameters dtTenantId=gmg80500 \
               dtConfigId="cfc78e0e-a116-3289-bba1-ac6ad7e81c1f" \
               locations='["eastus","westus","westeurope"]' \
               dtMonitoringServicePrincipalId="<service-principal-object-id>" \
               dtLogsPartitionCount=8 \
               dtEventsPartitionCount=2 \
               maximumThroughputUnits=20 \
```

## Naming Convention (CAF Compliant)

Following Azure Cloud Adoption Framework naming conventions:

| Resource Type       | Abbreviation | Format                           | Example                  |
|---------------------|--------------|----------------------------------|--------------------------|
| Resource Group      | rg           | rg-dt-{dtTenantId}-{location}    | rg-dt-abc12345-eastus    |
| Event Hub Namespace | evhns        | evhns-dt-{dtTenantId}-{location} | evhns-dt-abc12345-eastus |
| Event Hub (logs)    | evh          | dt-logs-evh                      | dt-logs-evh              |
| Event Hub (events)  | evh          | dt-events-evh                    | dt-events-evh            |
