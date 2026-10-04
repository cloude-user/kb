# 🛡️ KubeOps-Aegis: Infrastructure Issues, Incident Log & Cost Guide

This repository documents all critical issues encountered during the Azure infrastructure provisioning, their root causes, exact resolutions, browser Web Preview access troubleshooting, and the comprehensive 20-day cost estimation in Indian Rupees (INR).

---

## 📑 Table of Contents
1. [Grafana Asset Loading Failure & Cloud Shell Web Preview Issue](#1-grafana-asset-loading-failure--cloud-shell-web-preview-issue)
2. [Instant Resolution: Exposing Grafana via Public LoadBalancer](#2-instant-resolution-exposing-grafana-via-public-loadbalancer)
3. [Total Infrastructure Cost Analysis (20 Days in INR & USD)](#3-total-infrastructure-cost-analysis-20-days-in-inr--usd)
4. [Grafana & ArgoCD Access Cheatsheet](#4-grafana--argocd-access-cheatsheet)
5. [Complete Incident & Troubleshooting Post-Mortem](#5-complete-incident--troubleshooting-post-mortem)
   - [Issue 1: Regional Compute Quota Exhaustion](#issue-1-regional-compute-quota-exhaustion-errcode_insufficientvcpuquota)
   - [Issue 2: GitHub Actions SP Missing RBAC](#issue-2-github-actions-service-principal-missing-rbac-permissions-403-authorizationfailed)
   - [Issue 3: AKS Bring-Your-Own-VNet Role Missing](#issue-3-aks-bring-your-own-vnet-permission-missing-aks_network_contributor)
   - [Issue 4: Undeclared Variables in Root Module](#issue-4-undeclared-variables-in-root-module)
   - [Issue 5: Non-Short-Circuiting Boolean Index Error](#issue-5-non-short-circuiting-boolean-index-error-in-hcl-invalid-index)
   - [Issue 6: Deprecated / Unsupported VM SKU in East Asia](#issue-6-deprecated--unsupported-vm-sku-in-east-asia-standard_b2s-vs-standard_b2s_v2)
   - [Issue 7: Default Node Pool In-Place Rotation Error](#issue-7-default-node-pool-in-place-rotation-error-temporary_name_for_rotation)
   - [Issue 8: PostgreSQL Seed Data UUID Hex Syntax Error](#issue-8-postgresql-seed-data-uuid-hex-syntax-error)
   - [Issue 9: Prometheus Operator Webhook Hang & TLS Secret](#issue-9-prometheus-operator-admission-webhook-timeout--secret-hang)
   - [Issue 10: ArgoCD & Monitoring Pod Taints & Tolerations](#issue-10-argocd--monitoring-pods-unschedulable-due-to-system-pool-taint)
   - [Issue 11: Cloud Shell Reverse Proxy Subpath Asset Truncation](#issue-11-cloud-shell-relay-proxy-subpath-asset-truncation)
6. [Cost Optimization Playbook (Save up to 55%)](#6-cost-optimization-playbook-save-up-to-55)

---

## 1. Grafana Asset Loading Failure & Cloud Shell Web Preview Issue

### 🔍 Visual Symptom
When navigating to the login URL in Cloud Shell Web Preview (`https://ae2prodpncag-1-relay.servicebus.windows.net/<session-id>/proxy/3000/login/`), the browser displays an orange error screen:
```text
If you're seeing this Grafana has failed to load its application files

1. This could be caused by your reverse proxy settings.
2. If you host grafana under subpath make sure your grafana.ini root_url setting includes subpath. If not using a reverse proxy make sure to set serve_from_sub_path to true.
3. If you have a local dev build make sure you build frontend using: yarn start, or yarn build
4. Sometimes restarting grafana-server can help
5. Check if you are using a non-supported browser.
```

### 🔬 Deep Root Cause
* **Webpack Asset Path Resolution**: Grafana's React frontend compiles its JS chunks expecting to be fetched from `/public/build/app.<hash>.js` relative to the root domain.
* **Subpath Stripping**: Azure Cloud Shell proxies traffic under a dynamic, ephemeral subpath:
  ```text
  https://ae2prodpncag-1-relay.servicebus.windows.net/<random-guid>/proxy/3000/
  ```
* When the HTML loads, the client browser tries to fetch:
  ```text
  https://ae2prodpncag-1-relay.servicebus.windows.net/public/build/app.js
  ```
  This request strips the `/proxy/3000/` subpath completely and queries Microsoft's Service Bus Relay edge server directly, which returns a **404 Not Found**. Without its JavaScript application bundles, Grafana displays the "failed to load application files" fallback screen.
* Because the Cloud Shell session GUID changes unpredictably, configuring a static `root_url` in `grafana.ini` is brittle and breaks on new Cloud Shell sessions.

---

## 2. Instant Resolution: Exposing Grafana via Public LoadBalancer

The clean, production-grade fix is to expose the Grafana service as a **LoadBalancer** (identical to how ArgoCD is exposed). This provisions a dedicated Azure Public IP, completely bypassing Cloud Shell's subpath proxy.

### ⚡ Step 1: Run this 1-line command in Azure Cloud Shell
```bash
kubectl patch svc kube-prometheus-stack-grafana -n monitoring -p '{"spec": {"type": "LoadBalancer"}}'
```

### ⚡ Step 2: Watch for the External IP
```bash
kubectl get svc kube-prometheus-stack-grafana -n monitoring -w
```

Within 60-90 seconds, the `<pending>` external IP will resolve:
```text
NAME                            TYPE           CLUSTER-IP     EXTERNAL-IP     PORT(S)        AGE
kube-prometheus-stack-grafana   LoadBalancer   10.0.180.12    20.198.xxx.xxx   80:31234/TCP   25m
```

### ⚡ Step 3: Open in Any Browser
Navigate directly to:
```text
http://<EXTERNAL-IP>
```
* **No subpath glitches.**
* **All CSS & JS assets load instantly.**
* **No Cloud Shell 20-minute disconnects.**
* **Persistent WebSockets for live Prometheus charts.**

---

## 3. Total Infrastructure Cost Analysis (20 Days in INR & USD)

### 📊 Deployment Profile
- **Region**: Azure East Asia (`eastasia` - Hong Kong)
- **Architecture**: Single-Node Cluster Footprint (`Standard_D2s_v5`), Azure CNI Overlay with Cilium, NAT Gateway, PostgreSQL Flexible Server Burstable (`B_Standard_B1ms`), LoadBalancers for ArgoCD and Grafana.
- **Exchange Rate Benchmark**: $1.00 USD ≈ ₹83.50 INR.
- **Duration**: 20 Days = 480 Continuous Running Hours.

### 💰 Itemized Cost Breakdown

| Resource Component | Resource Name / Azure Type | Tier / SKU / Size | Hourly Rate (USD) | 20-Day Cost (USD) | 20-Day Cost (INR ₹) | Share (%) |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **AKS Worker Node** | `aks-systempool-vmss` | 1x `Standard_D2s_v5` (2 vCPU, 8 GB RAM) | $0.096 / hr | $46.08 | ₹3,848 | 43.5% |
| **Node Managed OS Disk** | OS Disk for Node Pool | 1x 128 GiB Premium SSD (P10) | $0.027 / hr ($19.71/mo) | $13.14 | ₹1,097 | 12.4% |
| **PostgreSQL Database** | `kubeops-aegis-psql-prd` | `Standard_B1ms` (1 vCPU, 2 GB RAM Burstable) | $0.021 / hr | $10.08 | ₹842 | 9.5% |
| **PostgreSQL Storage** | DB Managed Disk + Backup | 32 GiB Managed Storage | $0.005 / hr ($3.68/mo) | $2.45 | ₹205 | 2.3% |
| **Azure NAT Gateway** | `natgw-kubeops-aegis-prd` | 1 Managed NAT Gateway Instance | $0.045 / hr | $21.60 | ₹1,804 | 20.4% |
| **Data Processing (NAT)**| NAT Gateway Traffic | Ingress/Egress (~10 GB estimated) | $0.045 / GB | $0.45 | ₹38 | 0.4% |
| **Public IP Addresses** | NAT GW + ArgoCD + Grafana | 3x Standard Static Public IPs | $0.015 / hr ($0.005 each) | $7.20 | ₹601 | 6.8% |
| **Container Registry** | `kubeopsaegisacrprd` | Basic SKU Tier | $0.167 / day | $3.34 | ₹279 | 3.2% |
| **Storage Account** | `kubeopsaegisstprd` | Standard LRS Blob Storage (<10 GB) | Pay-as-you-go | $0.20 | ₹17 | 0.2% |
| **Virtual Network & Subnets**| `vnet-kubeops-aegis-prd` | 4 Subnets, Private DNS Zone | Free | $0.00 | ₹0 | 0.0% |
| **AKS Control Plane** | `aks-kubeops-aegis-prd` | Free Tier Cluster Management | Free | $0.00 | ₹0 | 0.0% |
| **Log Analytics & Monitor**| Default workspace | Metrics & Logs (<5 GB ingest/mo) | Free tier / minimal | ~$1.50 | ₹125 | 1.4% |
| **TOTAL OVERALL** | **Entire Azure Footprint** | **20 Days Continuous Operation** | **~$0.221 / hr** | **~$106.04 USD** | **~₹8,855 INR** | **100%** |

### 📈 Daily & Hourly Summary
* **Hourly Cost**: ~$0.221 USD / hour (~₹18.45 INR / hour)
* **Daily Cost (24 hours)**: ~$5.30 USD / day (**~₹442.75 INR / day**)
* **Total 20-Day Cost**: **~$106.04 USD (~₹8,855 INR)**

---

## 4. Grafana & ArgoCD Access Cheatsheet

### 🔐 1. Grafana
* **Username**: `admin`
* **Password Retrieve Command**:
  ```bash
  kubectl get secret --namespace monitoring kube-prometheus-stack-grafana -o jsonpath="{.data.admin-password}" | base64 -d
  echo ""
  ```
* **URL**:
  - Direct LoadBalancer IP: `http://<GRAFANA-EXTERNAL-IP>`

---

### 🐙 2. ArgoCD GitOps
* **Username**: `admin`
* **Password Retrieve Command**:
  ```bash
  kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
  echo ""
  ```
* **Direct Public LoadBalancer Access**:
  - URL: `https://20.187.177.88` or `http://20.187.177.88`

---

## 5. Complete Incident & Troubleshooting Post-Mortem

### Issue 1: Regional Compute Quota Exhaustion (`ErrCode_InsufficientVCPUQuota`)
* **Symptom**: `left regional vcpu quota 0, requested quota 4`.
* **Root Cause**: Subscription in `eastasia` had a cap of 4 vCPUs. Neo4j VM consumed 2 vCPUs (`Standard_D2s_v5`) and the AKS system pool consumed 2 vCPUs. Adding a user node pool with `Standard_D4s_v5` exceeded the quota.
* **Resolution**:
  1. Disabled Neo4j (`deploy_neo4j = false`), reclaiming 2 vCPUs.
  2. Single-Node Cluster Footprint (`deploy_user_node_pool = false`), running system and monitoring workloads on the single node.

---

### Issue 2: GitHub Actions Service Principal Missing RBAC Permissions (`403 AuthorizationFailed`)
* **Symptom**: `StatusCode=403 Code="AuthorizationFailed" Message="The client does not have authorization to perform action 'Microsoft.Authorization/roleAssignments/write'..."`
* **Root Cause**: Terraform runner had `Contributor` role. Azure `Contributor` cannot create or modify RBAC role assignments (`azurerm_role_assignment`).
* **Resolution**: Assigned **Owner** or **User Access Administrator** at the Subscription level to Service Principal `app-github-actions-kubeops`.

---

### Issue 3: AKS Bring-Your-Own-VNet Permission Missing (`aks_network_contributor`)
* **Symptom**: AKS cluster failed when binding to subnets `snet-aks-system` and `snet-aks-user`.
* **Root Cause**: AKS User-Assigned Managed Identity requires the `Network Contributor` role on the Virtual Network to join pods to subnets.
* **Resolution**: Added `azurerm_role_assignment.aks_network_contributor` in `aks.tf` granting `Network Contributor` on `module.vnet[0].vnet_id` to `aks_identity`.

---

### Issue 4: Undeclared Variables in Root Module
* **Symptom**: `Error: Value for undeclared variable: A variable named "deploy_networking" was assigned on the command line...`
* **Root Cause**: Accidental commit wiped out `terraform/variables.tf`.
* **Resolution**: Restored complete `terraform/variables.tf` with all variable definitions, types, and descriptions.

---

### Issue 5: Non-Short-Circuiting Boolean Index Error in HCL (`Invalid index`)
* **Symptom**: `Error: Invalid index on aks.tf: api_server_authorized_ip_ranges[0]`
* **Root Cause**: Terraform HCL evaluates all elements in expressions. An empty list `[]` indexing `[0]` immediately threw an evaluation error.
* **Resolution**: Used safe lookup: `try(var.api_server_authorized_ip_ranges[0], "") != ""` preventing indexing failures on empty lists.

---

### Issue 6: Deprecated / Unsupported VM SKU in East Asia (`Standard_B2s` vs `Standard_B2s_v2`)
* **Symptom**: `The VM size of Standard_B2s is not allowed in your subscription in location 'eastasia'.`
* **Root Cause**: Azure deprecated generation-1 B-series in East Asia data centers.
* **Resolution**: Migrated SKU specification to `Standard_B2s_v2`.

---

### Issue 7: Default Node Pool In-Place Rotation Error (`temporary_name_for_rotation`)
* **Symptom**: `Error: temporary_name_for_rotation must be specified when updating default_node_pool properties...`
* **Root Cause**: Modifying `only_critical_addons_enabled` or `vm_size` on an active default node pool initiates surge VM rolling rotation in Azure.
* **Resolution**: Kept `only_critical_addons_enabled = true` and `system_node_vm_size = "Standard_D2s_v5"` to match the live cluster with zero diff, using pod tolerations instead of node recreation.

---

### Issue 8: PostgreSQL Seed Data UUID Hex Syntax Error
* **Symptom**: `ERROR: invalid input syntax for type uuid: "p1111111-1111-1111-1111-111111111111"`
* **Root Cause**: RFC 4122 standard strictly mandates hexadecimal digits (`0-9`, `a-f`). Dummy UUIDs with `p` and `o` caused SQL execution failure.
* **Resolution**: Replaced prefixes with valid hex characters (`b` and `d`) in `schema.sql` and `init-db-job.yaml`.

---

### Issue 9: Prometheus Operator Admission Webhook Timeout & Secret Hang
* **Symptom**: Pod `kube-prometheus-stack-operator` stuck in `ContainerCreating` for >10 minutes.
* **Root Cause**: The admission webhook patch pre-install job hung waiting for TLS certificates and admission secret generation, which failed due to single-node scheduling constraints.
* **Resolution**:
  In `terraform/prometheus.tf`:
  1. Disabled admission webhook validation: `prometheusOperator.admissionWebhooks.enabled = false` and `prometheusOperator.admissionWebhooks.patch.enabled = false`.
  2. Set `prometheusOperator.tls.enabled = false`.
  3. Pre-created TLS secret via `openssl` in Cloud Shell.

---

### Issue 10: ArgoCD & Monitoring Pods Unschedulable due to System Pool Taint
* **Symptom**: ArgoCD and Prometheus pods in `Pending` state with `0/1 nodes available: 1 node(s) had untolerated taint {CriticalAddonsOnly: true}`.
* **Root Cause**: Single system node pool has taint `CriticalAddonsOnly=true:NoSchedule`.
* **Resolution**: Added toleration block to ArgoCD and Prometheus Helm release values:
  ```yaml
  tolerations:
    - key: "CriticalAddonsOnly"
      operator: "Exists"
      effect: "NoSchedule"
  ```
  All pods scheduled and reached `Running` status immediately.

---

### Issue 11: Cloud Shell Relay Proxy Subpath Asset Truncation
* **Symptom**: Browser shows `Found.` or `Grafana has failed to load its application files` when proxied through Cloud Shell.
* **Root Cause**: Cloud Shell relay subpath `/proxy/3000/` strips webpack asset calls (`/public/build/app.js`), causing 404s.
* **Resolution**: Expose Grafana as `LoadBalancer` service to obtain a dedicated public IP.

---

## 6. Cost Optimization Playbook (Save up to 55%)

To reduce your 20-day Azure spend from **₹8,855 INR** down to **~₹5,000 INR**:

### 1. Stop AKS Cluster When Inactive
When not developing or testing, stop the AKS cluster:
```bash
az aks stop --resource-group kubeops-aegis-rg-compute-prd --name aks-kubeops-aegis-prd
```
* **Savings**: Halts the `Standard_D2s_v5` compute charge ($0.096/hr). If stopped 14 hours/day and weekends, saves **~₹2,500 INR**!
* **Resume anytime**:
  ```bash
  az aks start --resource-group kubeops-aegis-rg-compute-prd --name aks-kubeops-aegis-prd
  ```

### 2. Stop PostgreSQL Database When Inactive
```bash
az postgres flexible-server stop --resource-group kubeops-aegis-rg-data-prd --name kubeops-aegis-psql-prd
```
* Can remain stopped for up to 7 days, eliminating compute charges.

### 3. Clean up Completed Database Jobs
Remove temporary init pods:
```bash
kubectl delete job psql-init-db -n data-layer
```
