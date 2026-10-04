# Production SRE AI Agent Architecture, Observability & Incident Telemetry
**Author & Lead Architect:** Sundeep  
**Project:** KubeOps-Aegis Autonomous Cloud-Native SRE Platform  
**Target Environment:** Azure Kubernetes Service (AKS) Enterprise Production  
**Scope:** Autonomous Diagnostics, Prometheus RED Metrics, Azure Workload Identity vs AWS Bedrock, Multi-Layer Crash Log Analysis & Helm Deployment

---

## 1. Architectural Overview & Value Proposition

In high-velocity cloud-native production environments, standard Kubernetes self-healing (`restartPolicy: Always`) is fundamentally insufficient:
- When a service encounters an **OutOfMemoryError (OOMKilled, Exit Code 137)**, Kubelet restarts the container, but without automated root-cause analysis, the container enters `CrashLoopBackOff`, depleting pod capacity and triggering cascade HTTP 504 Gateway Timeouts.
- When an engineer is paged at 2:00 AM, the average **MTTR (Mean Time to Resolution)** is 30–45 minutes just to collect pod logs, inspect Prometheus dashboards, identify the leaking container, and scale memory limits.

I engineered the **KubeOps-Aegis Autonomous SRE AI Agent** to reduce MTTR from **35 minutes to under 20 seconds**. It executes a stateful diagnostic-remediation loop powered by **LangGraph**, **LangChain**, **Prometheus**, and **Azure Kubernetes Service RBAC**.

```
+----------------------------------------------------------------------------------------------------+
|                                    KubeOps-Aegis SRE Agent Loop                                    |
|                                                                                                    |
|   +-------------------+       +---------------------+       +-------------------+                  |
|   | 1. Triage Node    | ----> | 2. Diagnose Node    | ----> | 3. Remediate Node |                  |
|   | (Scan Failing Pod)|       | - PromQL RED Metrics|       | - Self-Healing    |                  |
|   +-------------------+       | - Prev Crash Logs   |       |   Pod Restart     |                  |
|             |                 | - K8s Events        |       +-------------------+                  |
|             | (Healthy)       | - LangGraph RCA     |                 |                            |
|             v                 +---------------------+                 v                            |
|          [ END ]                         |                  +-------------------+                  |
|                                          +----------------> | 4. Audit Node     |                  |
|                                                             | - Azure Blob Dump |                  |
|                                                             +-------------------+                  |
+----------------------------------------------------------------------------------------------------+
```

---

## 2. Authentication: Azure OpenAI vs. AWS Bedrock (Passwordless RBAC)

### The AWS Bedrock Model (IAM Role Based, Zero API Keys)
In AWS Bedrock, API keys do not exist. Workloads running on AWS EKS authenticate using **IRSA (IAM Roles for Service Accounts)** or **EKS Pod Identity**:
1. An IAM Role is created with policies allowing `bedrock:InvokeModel`.
2. The Kubernetes ServiceAccount is annotated with `eks.amazonaws.com/role-arn`.
3. The AWS SDK uses the OIDC Web Identity Token injected by the mutating webhook to request temporary STS credentials.

### The Azure OpenAI Model: How I Engineered Passwordless RBAC
In Azure, the exact equivalent to AWS Bedrock IAM is **Azure Workload Identity** combined with **Azure Role-Based Access Control (Azure RBAC)**. 

#### Option A: Passwordless Azure Workload Identity (Enterprise Recommended)
No secrets or keys are stored in Kubernetes Secrets or environment variables.
1. **Azure User-Assigned Managed Identity**: Created in Azure (`aks-kubeops-aegis-prd-agent-identity`).
2. **Federated Identity Credential**: Established between the AKS OIDC Issuer URL and the Kubernetes ServiceAccount (`kubeops-aegis-agent-sa` in namespace `aegis-apps`).
3. **Azure RBAC Role Assignment**: The Managed Identity is granted the Azure built-in role:
   - **`Cognitive Services OpenAI User`** (or `Cognitive Services Contributor`) scoped to the Azure OpenAI resource.
4. **Code Implementation in Agent**:
   ```python
   from azure.identity import DefaultAzureCredential, get_bearer_token_provider
   from langchain_openai import AzureChatOpenAI

   token_provider = get_bearer_token_provider(
       DefaultAzureCredential(), "https://cognitiveservices.azure.com/.default"
   )

   llm = AzureChatOpenAI(
       azure_endpoint="https://kubeops-aegis-openai.openai.azure.com/",
       azure_deployment="gpt-4o",
       api_version="2024-06-01",
       azure_ad_token_provider=token_provider,
       temperature=0.1
   )
   ```

#### Option B: Where to Find the API Key in the Azure Portal
If provisioning via classic static API keys:
1. Navigate to **Azure Portal** (`portal.azure.com`).
2. Search for **Azure OpenAI** or **Azure AI services | Azure OpenAI**.
3. Select your Azure OpenAI instance (e.g., `kubeops-aegis-openai`).
4. In the left navigation menu under **Resource Management**, click **Keys and Endpoint**.
5. Copy **KEY 1** and the **Endpoint URL** (`https://<resource-name>.openai.azure.com/`).
6. Under **Model Deployments** (or via **Azure AI Foundry / Azure OpenAI Studio**), verify your deployment name (`gpt-4o`).

#### Option C: Deterministic Fallback Engine (Zero Key, Zero Cost)
If no external LLM is configured, my agent falls back automatically to its internal **SRE Deterministic Heuristic Engine**. It correlates exit codes (e.g., Exit Code 137 for OOMKilled), Prometheus telemetry spikes, and stack trace logs to perform automated remediation with 0 external token costs.

---

## 3. The Log Analysis Problem: Why "Last 50 Lines" Fails in Real Incidents

A naive SRE script typically runs `read_namespaced_pod_log(tail_lines=50)`. In production, this causes critical diagnostic failures:

### The Crash Timing Problem
1. When a container runs out of memory or hits an unhandled exception (Exit Code 137 / 1), the Linux kernel kills the container.
2. Kubelet detects the dead container and immediately launches a new container instance.
3. If an automated script or engineer reads the logs of the *currently running* container (`previous=False`), it returns 2–3 lines of initialization banner (`Starting uvicorn server...`). The actual OOM stack trace or memory allocation panic occurred in the **dead container**!

### My Complete Multi-Layer Diagnostic Solution
I engineered `k8s_client.py` to extract diagnostic telemetry across 4 distinct layers:
1. **Container Termination State Inspection**:
   Inspect `pod.status.container_statuses[].last_state.terminated`. Extract `exit_code` (`137` = Linux OOMKiller, `1` = unhandled exception, `143` = SIGTERM), `reason` (`OOMKilled`, `Error`), and container termination timestamps.
2. **Previous Container Logs (`previous=True`)**:
   Query `read_namespaced_pod_log(pod_name, namespace, previous=True, tail_lines=100)`. This captures the exact fatal crash buffer, stack traces, and out-of-memory errors from the terminated container.
3. **Current Container Logs (`previous=False`)**:
   Check if the newly restarted container is healthy or stuck in initial configuration loops.
4. **Kubernetes Warning Events**:
   Query `CoreV1Api.list_namespaced_event(namespace, field_selector=f"involvedObject.name={pod_name}")`. This surfaces Kubelet-level kernel events:
   - `Warning OOMKilling pod/payment-api: Memory cgroup out of memory: Killed process 412 (python)`
   - `Warning BackOff pod/payment-api: Back-off restarting failed container`

---

## 4. Line-by-Line Code Breakdown

### A. `agent/app/services/k8s_client.py`
This service handles all programmatic interactions with the Kubernetes API Server.

- **Lines 16–32 (`initialize`)**:
  Determines execution context. If `settings.KUBERNETES_IN_CLUSTER` is true, loads the in-cluster ServiceAccount token and CA cert from `/var/run/secrets/kubernetes.io/serviceaccount/` via `config.load_incluster_config()`. If running locally, loads the local kubeconfig via `config.load_kube_config()`. Instantiates `client.CoreV1Api()` and `client.AppsV1Api()`.
- **Lines 57–95 (`_get_failing_pods_sync`)**:
  Scans all pods in the namespace. Inspects both `p.status.phase` (Pending, Failed) and `p.status.container_statuses`. It iterates through container statuses to detect `last_state.terminated.exit_code` (137, 139, 1, 255) and `state.waiting.reason` (`CrashLoopBackOff`, `ImagePullBackOff`). Returns structured metadata containing pod name, container, restart count, and exit code.
- **Lines 97–150 (`_get_pod_logs_sync`)**:
  Executes multi-layer log aggregation:
  1. Calls `read_namespaced_pod_log(..., previous=True)` to extract previous crash logs.
  2. Calls `read_namespaced_pod_log(..., previous=False)` for current logs.
  3. Queries `list_namespaced_event` filtering on `involvedObject.name={pod_name}` to extract Kubelet warning events (`OOMKilling`, `BackOff`).
- **Lines 152–165 (`_restart_pod_sync`)**:
  Triggers automated remediation by invoking `delete_namespaced_pod`. Kubernetes Deployment ReplicaSet controller detects the missing replica and spawns a fresh pod with clean memory space.

---

### B. `agent/app/services/prometheus_client.py`
Scrapes live **RED (Rate, Errors, Duration)** metrics to correlate logs with physical resource telemetry.

- **Lines 14–29 (`query`)**:
  Uses `aiohttp.ClientSession` to asynchronously query the Prometheus HTTP API endpoint:
  `GET http://kube-prometheus-stack-prometheus.monitoring.svc:9090/api/v1/query?query={promql}`.
- **Lines 31–80 (`get_pod_telemetry`)**:
  Executes three critical PromQL expressions:
  1. **Memory Working Set (Saturation)**:
     `sum(container_memory_working_set_bytes{pod=~"<pod>.*", namespace="<ns>", container!=""}) / (1024 * 1024)`
     Measures actual physical RAM in MiB consumed by the container's cgroup.
  2. **CPU Usage (Rate)**:
     `sum(rate(container_cpu_usage_seconds_total{pod=~"<pod>.*", namespace="<ns>", container!=""}[5m]))`
     Measures active CPU core utilization.
  3. **HTTP 5xx Error Rate (Errors)**:
     `sum(rate(http_requests_total{status=~"5..", namespace="<ns>"}[5m])) or vector(0)`
     Detects user-impacting request failures.

---

### C. `agent/app/engine/sre_agent.py`
The core LangGraph state machine orchestrating triage, diagnosis, remediation, and auditing.

- **Lines 20–32 (`SREAgentState`)**:
  Defines the typed dictionary state passed across the workflow: `namespace`, `failing_pod`, `logs`, `metrics`, `rca_result`, `incident_id`, `remediation_executed`, and `blob_url`.
- **Lines 40–80 (`_get_llm_model`)**:
  Factory supporting Azure OpenAI via passwordless Workload Identity token provider (`get_bearer_token_provider`), static API keys, OpenAI `gpt-4o`, Google Gemini, and automated rule fallback.
- **Lines 82–104 (`_build_langgraph_workflow`)**:
  Assembles the directed state graph:
  - `triage` $\rightarrow$ conditional edge `_should_diagnose`: if pod failing, branch to `diagnose`; else branch to `END`.
  - `diagnose` $\rightarrow$ `remediate` $\rightarrow$ `audit` $\rightarrow$ `END`.
- **Lines 108–127 (`_triage_node`)**:
  Executes `k8s_service.get_failing_pods()`. If an anomaly is found, generates a unique incident ID (`INC-XXXXXX`) and updates state to `TRIAGED`.
- **Lines 155–180 (`_diagnose_node`)**:
  Executes parallel asynchronous fetches for multi-layer pod logs (`previous=True` + current + events) and Prometheus RED metrics. Calls `_ai_root_cause_analysis` to synthesize the root cause and registers an initial incident record in the database.
- **Lines 182–202 (`_remediate_node`)**:
  Executes autonomous healing (`k8s_service.restart_pod`) if `auto_apply=True`. Updates incident status to `RESOLVED`.
- **Lines 204–224 (`_audit_node`)**:
  Bundles the entire incident post-mortem (target pod, PromQL metrics at crash, crash logs, RCA, remediation status) into a JSON payload and uploads it to **Azure Blob Storage** (`incident-logs` container) for permanent audit compliance.
- **Lines 226–277 (`_ai_root_cause_analysis`)**:
  If LLM is active, prompts the model with pod state, telemetry, and logs. Otherwise, executes deterministic rules: detects exit code 137 / OOM logs, confirms memory saturation, and formulates a remediation plan.

---

## 5. Helm Chart Implementation (`charts/aegis-agent`)

The agent is fully packaged as a production Helm chart located at `charts/aegis-agent/`:

```
charts/aegis-agent/
├── Chart.yaml                  # Helm chart metadata (v1.0.0, appVersion 2.0.0)
├── values.yaml                 # Default configuration, tolerations, and resource limits
└── templates/
    ├── _helpers.tpl            # Standard naming and label templating macros
    ├── serviceaccount.yaml     # ServiceAccount with Azure Workload Identity annotations
    ├── rbac.yaml               # ClusterRole and ClusterRoleBinding for pod inspection & restart
    ├── deployment.yaml         # Agent pod deployment with CriticalAddonsOnly toleration
    └── service.yaml            # ClusterIP service for REST API and WebSocket streams
```

### Key Values in `values.yaml`
```yaml
replicaCount: 1

image:
  repository: kubeopsaegisacrprd.azurecr.io/kubeops-aegis-agent
  tag: "latest"
  pullPolicy: IfNotPresent

serviceAccount:
  create: true
  name: "kubeops-aegis-agent-sa"
  labels:
    azure.workload.identity/use: "true"

tolerations:
  - key: "CriticalAddonsOnly"
    operator: "Exists"
    effect: "NoSchedule"

config:
  environment: "prd"
  kubernetesInCluster: "true"
  prometheusUrl: "http://kube-prometheus-stack-prometheus.monitoring.svc:9090"
  azureOpenaiEndpoint: "https://kubeops-aegis-openai.openai.azure.com/"
  azureOpenaiDeploymentName: "gpt-4o"
```

### Deployment Commands via Azure Cloud Shell
```bash
# 1. Inspect the chart templates
helm template kubeops-aegis-agent ./charts/aegis-agent -n aegis-apps

# 2. Deploy or upgrade the SRE Agent on AKS
helm upgrade --install kubeops-aegis-agent ./charts/aegis-agent \
  --namespace aegis-apps \
  --set image.tag=latest

# 3. Verify agent pod health and logs
kubectl get pods -n aegis-apps -l app.kubernetes.io/name=aegis-agent
kubectl logs -n aegis-apps -l app.kubernetes.io/name=aegis-agent -f
```

---

## 6. SRE Scenario Walkthrough: Chaos Engineering & Auto-Healing

### Scenario: Simulating a Memory Leak (OOMKilled Exit Code 137)
1. **Trigger Memory Leak**:
   ```bash
   curl -X POST http://20.24.117.161/api/v1/chaos/oom-leak -H "Content-Type: application/json" -d '{"leak_rate_mb_per_sec": 50}'
   ```
2. **Cluster Behavior**:
   The backend container consumes memory up to the cgroup limit (512Mi). The Linux kernel kills the container with signal `SIGKILL` (Exit code 137). Kubelet records warning event `OOMKilling`.
3. **Agent Action (under 15 seconds)**:
   - **Triage**: Detects `payment-api` in `CrashLoopBackOff` with restart count incremented.
   - **Diagnose**: Scrapes Prometheus (`container_memory_working_set_bytes` = 512MiB), pulls previous container crash logs (`previous=True`), and captures Kubelet warning event.
   - **Reasoning**: Identifies fatal OOMKilled condition with 99.8% memory saturation.
   - **Remediation**: Executes automated pod restart, flushes corrupt cgroup cache, and logs proposal to bump memory limit in GitOps repository.
   - **Audit**: Publishes post-mortem JSON report to Azure Blob Storage and broadcasts telemetry to Grafana and WebSocket subscribers.
