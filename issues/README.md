# 🛠️ Azure Infrastructure Troubleshooting & Post-Mortem Guide

This repository documents every real-world error, root cause, architectural analysis, and production resolution encountered during the deployment of **KubeOps-Aegis** on Microsoft Azure using Terraform, GitHub Actions, and Azure Resource Manager (ARM).

---

## 📑 Table of Contents

1. [Storage Account Naming Constraints (`kubeops-aegis-st-prd`)](#1-storage-account-naming-constraints)
2. [Storage 403 Authorization on Container Creation](#2-storage-403-authorization-on-container-creation)
3. [Neo4j VM SKU Capacity Restrictions in East US (`SkuNotAvailable`)](#3-neo4j-vm-sku-capacity-restrictions)
4. [PostgreSQL Flexible Server Version Empty Array (`Version should be in: []`)](#4-postgresql-version-empty-array)
5. [Cross-Region Re-creation Race Conditions (`SubnetNotFound`, `ParentResourceNotFound`)](#5-cross-region-re-creation-race-conditions)
6. [Orphaned Azure Resources State Desync (`Resource Already Exists`)](#6-orphaned-azure-resources-state-desync)
7. [PostgreSQL VNet vs Public Access Conflict (`ConflictingPublicNetworkAccess`)](#7-postgresql-vnet-vs-public-access-conflict)
8. [PostgreSQL Availability Zone Mutation Drift (`zone can only be changed...`)](#8-postgresql-availability-zone-mutation-drift)
9. [Architecture Best Practices & Cheatsheet](#9-architecture-best-practices--cheatsheet)

---

## 1. Storage Account Naming Constraints

### 🔴 The Error
```text
Error: creating Azure Storage Account: "kubeops-aegis-st-prd":
storage account name can only consist of lowercase letters and numbers, and must be between 3 and 24 characters long.
```

### 🔍 Root Cause
Unlike AWS S3 buckets (which permit hyphens and dots), Azure Storage account names form a global DNS subdomain (`<name>.blob.core.windows.net`). Azure enforces strict constraints:
- Length: 3–24 characters.
- Characters: Numbers (`0-9`) and lowercase letters (`a-z`) **only**.
- Hyphens (`-`), underscores (`_`), and uppercase letters are strictly prohibited.

### 🟢 The Resolution
In `terraform/main.tf`:
```hcl
module "storage" {
  source               = "./modules/storage"
  # Strip hyphens dynamically using replace()
  storage_account_name = "${replace(var.prefix, "-", "")}st${var.environment}" # Result: kubeopsaegisstprd
  ...
}
```

---

## 2. Storage 403 Authorization on Container Creation

### 🔴 The Error
```text
Error: creating Container "order-receipts" in Storage Account "kubeopsaegisstprd":
blobs.Client#Create: Failure responding to request: StatusCode=403:
AuthorizationFailure: This request is not authorized to perform this operation.
```

### 🔍 Root Cause
When Azure Storage Accounts are configured with `default_action = "Deny"` or `public_network_access_enabled = false` without pre-authorizing the GitHub Actions runner public IP, the ARM control plane / data plane blocks API requests originating from outside the virtual network.

### 🟢 The Resolution
In `terraform/modules/storage/main.tf`:
```hcl
resource "azurerm_storage_account" "storage" {
  name                          = var.storage_account_name
  public_network_access_enabled = true # Allows deployment runners to provision containers

  network_rules {
    default_action = "Allow" # Allows initial automated container provisioning
    bypass         = ["AzureServices"]
  }
}
```
*Internal cluster traffic still routes privately over the **Private Endpoint** (`privatelink.blob.core.windows.net`).*

---

## 3. Neo4j VM SKU Capacity Restrictions

### 🔴 The Error
```text
Error: creating Linux Virtual Machine "kubeops-aegis-vm-neo4j-prd": performing CreateOrUpdate:
unexpected status 409 (409 Conflict) with error: SkuNotAvailable:
The requested VM size for resource 'Following SKUs have failed for Capacity Restrictions: Standard_B2ms'
is currently not available in location 'eastus'.
```

### 🔍 Root Cause
Azure regions periodically exhaust capacity for specific VM families—especially burstable B-series (`Standard_B2ms`, `Standard_B2s`). Even some D-series (`Standard_D2s_v5`) encounter subscription-specific quotas or physical rack constraints in heavily populated regions like `East US`.

### 🟢 The Resolution
1. **Decoupled Deployment:** Added a dedicated `deploy_neo4j` feature toggle so core data and networking layers can deploy independently:
   ```hcl
   # terraform/main.tf
   module "neo4j" {
     count  = var.deploy_data_layer && var.deploy_networking && var.deploy_neo4j ? 1 : 0
     source = "./modules/neo4j"
   }
   ```
2. **Environment Variable Control:** Set `deploy_neo4j = false` in `terraform/env/prd.tfvars` until compute capacity is verified or Neo4j AuraDB managed cloud is connected.

---

## 4. PostgreSQL Version Empty Array

### 🔴 The Error
```text
Error: creating Flexible Server "kubeops-aegis-psql-prd": performing Create:
unexpected status 400 (400 Bad Request) with error: ParameterOutOfRange:
The value of the 'Version' should be in: []. Verify that the specified parameter value is correct.
```

### 🔍 Root Cause
The empty brackets (`[]`) mean that the Azure ARM regional capability matrix returned **zero valid PostgreSQL versions**. 
- In the Azure Portal UI, selecting `East US` displayed:
  > ❌ **"Subscription 'Azure subscription 1' is not allowed to provision in 'East US'."**
- Microsoft Azure subscriptions frequently restrict managed database services in primary overloaded regions (`East US`) while keeping them open in secondary regions.
- Because the region was policy-restricted for that subscription, the API returned no valid version options.

### 🟢 The Resolution
1. Tested regional availability in Azure Portal and discovered **East Asia (`eastasia`)** has full, unrestricted access to PostgreSQL Flexible Server versions 14, 15, 16, 17, and 18.
2. Updated deployment location in `terraform/env/prd.tfvars`:
   ```hcl
   location = "eastasia"
   ```
3. Parameterized database version and SKU to avoid hardcoding:
   ```hcl
   db_sku_name = "GP_Standard_D2s_v3"
   db_version  = "16"
   ```

---

## 5. Cross-Region Re-creation Race Conditions

### 🔴 The Error
```text
Error: Subnet "snet-aks-system" was not found
Error: ParentResourceNotFound: Failed to perform 'write' on resource(s) of type 'privateDnsZones/virtualNetworkLinks'
Error: Provider produced inconsistent result after apply ... Root object was present, but now absent.
```

### 🔍 Root Cause
When switching regions from `eastus` to `eastasia`:
- Terraform initiates a **parallel destroy and create** lifecycle.
- It began deleting the old Resource Groups in `eastus` while creating the new VNet and subnets in `eastasia`.
- Because Azure resource group deletion is asynchronous, Terraform attempted to link child resources (DNS links, NAT associations) before the newly targeted subnets finished registering in ARM.

### 🟢 The Resolution
1. Allowed the destroy operation to complete fully.
2. Triggered a clean re-run of `apply`. Once all old regional resources were deleted, the second apply provisioned all resources sequentially without dependency collisions.

---

## 6. Orphaned Azure Resources State Desync

### 🔴 The Error
```text
Error: A resource with the ID "/.../virtualNetworks/kubeops-aegis-vnet-prd" already exists -
to be managed via Terraform this resource needs to be imported into the State.
Error: A resource with the ID "/.../privatelink.vaultcore.azure.net/virtualNetworkLinks/kubeops-aegis-kv-prd-dns-link" already exists.
```

### 🔍 Root Cause
When a previous Terraform run succeeds in provisioning a resource (like a VNet or DNS Link) but fails on a downstream resource (e.g. PostgreSQL), Terraform terminates execution before updating the remote `.tfstate` blob with the IDs of those completed items. On the subsequent run, Terraform tries to `Create` them again, triggering Azure ARM's 409 Conflict.

### 🟢 The Resolution
**Fastest Fix (Portal / CLI):**
1. Opened Azure Portal $\rightarrow$ Resource Group `kubeops-aegis-rg-network-prd`.
2. Deleted the orphaned unlinked resources:
   - `kubeops-aegis-nat-gw-prd` (NAT Gateway)
   - `kubeops-aegis-nat-gw-prd-pip` (Public IP)
   - `kubeops-aegis-vnet-prd` (VNet)
   - `kubeops-aegis-kv-prd-dns-link` (DNS Virtual Network Link)
3. Re-ran `terraform apply` — Terraform created them fresh and recorded them properly into the remote state.

---

## 7. PostgreSQL VNet vs Public Access Conflict

### 🔴 The Error
```text
Error: creating Flexible Server "kubeops-aegis-psql-prd": performing Create:
unexpected status 400 (400 Bad Request) with error:
ConflictingPublicNetworkAccessAndVirtualNetworkConfiguration: Conflicting configuration is detected
between Public Network Access and Virtual Network arguments. Public Network Access is not supported along with Virtual Network feature.
```

### 🔍 Root Cause
Azure Database for PostgreSQL Flexible Server enforces mutually exclusive networking modes:
- **Private Access (VNet Integration):** Requires a dedicated delegated subnet (`delegated_subnet_id`) and private DNS zone (`private_dns_zone_id`).
- **Public Access:** Firewall rules with public IP ranges.

In newer Azure API versions, `public_network_access_enabled` defaults to `true` (Enabled) if omitted. Since our configuration supplied `delegated_subnet_id`, Azure detected conflicting intent.

### 🟢 The Resolution
In `terraform/modules/postgresql/main.tf`:
```hcl
resource "azurerm_postgresql_flexible_server" "psql" {
  name                          = var.server_name
  delegated_subnet_id           = var.subnet_id
  private_dns_zone_id           = azurerm_private_dns_zone.dns_psql.id
  public_network_access_enabled = false # Explicitly disable public access
  ...
}
```

---

## 8. PostgreSQL Availability Zone Mutation Drift

### 🔴 The Error
```text
Error: `zone` can only be changed when exchanged with the zone specified in `high_availability.0.standby_availability_zone`
with module.postgresql[0].azurerm_postgresql_flexible_server.psql
```

### 🔍 Root Cause
When creating a single-node PostgreSQL Flexible Server without pinning `zone`, Azure automatically picks an active Availability Zone (e.g. Zone 1 or Zone 2). On subsequent `plan` / `apply` executions, Terraform sees `zone` defined in state but absent in the HCL file, so it attempts to update `zone = null`. Azure ARM explicitly forbids changing the availability zone of an existing single-instance server.

### 🟢 The Resolution
In `terraform/modules/postgresql/main.tf`:
```hcl
resource "azurerm_postgresql_flexible_server" "psql" {
  ...
  lifecycle {
    ignore_changes = [
      zone,
      high_availability
    ]
  }
}
```

---

## 9. Architecture Best Practices & Cheatsheet

| Component | Golden Rule |
| :--- | :--- |
| **Storage Account Naming** | Alphanumeric only, 3–24 chars, no hyphens (`kubeopsaegisstprd`). |
| **Private Subnet Delegation** | Must have `Microsoft.DBforPostgreSQL/flexibleServers` service delegation. |
| **PostgreSQL Private Access** | Always set `public_network_access_enabled = false` and `lifecycle { ignore_changes = [zone] }`. |
| **Regional Quotas** | If a region shows `Version should be in: []`, check the Azure Portal creation wizard for regional subscription restrictions. |
| **Cross-Region Rule** | Delegated subnets **cannot** span regions; VNet and PostgreSQL must share the identical Azure region. |
