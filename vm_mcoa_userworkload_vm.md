# VM Monitoring with RHACM & MCOA
### Java Applications on OpenShift Virtualization
 
This tutorial covers the workflow for monitoring Java applications running inside Virtual Machines on OpenShift Virtualization, using Red Hat Advanced Cluster Management (RHACM) and the Multi-Cluster Observability Addon (MCOA).
 
MCOA uses the Prometheus Agent mode, which does not store data locally on managed nodes. Instead it streams metrics directly to the Hub cluster, reducing disk pressure on managed nodes while centralising observability.
 
---
 
## Prerequisites
 
- RHACM installed and at least one managed cluster attached
- OpenShift Virtualization installed on the managed cluster
- A Java VM (`my-java-vm`) running in namespace `my-java-app-ns` and exposing a Prometheus-compatible `/metrics` endpoint on port `8080`
- `oc` CLI access to both the Hub and managed clusters
---
 
## Phase 1: Managed Cluster Preparation
 
All resources in this phase are applied to the **managed cluster** where your VM lives.
 
### Step 1 — Enable User Workload Monitoring (UWM)
 
MCOA collects metrics from the local Prometheus stack. If User Workload Monitoring is disabled, metrics from user-defined namespaces (including your Java VM) are invisible to the scraper.
 
Create or patch the `cluster-monitoring-config` ConfigMap in the `openshift-monitoring` namespace:
 
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: cluster-monitoring-config
  namespace: openshift-monitoring
data:
  config.yaml: |
    enableUserWorkload: true
```
 
> **Note:** The key must be exactly `config.yaml` under `data`. The resource uses `data`, not `spec` — `ConfigMap` has no `spec` field. After applying, the `prometheus-user-workload` pods in `openshift-user-workload-monitoring` will start within a few minutes.
 
Apply it:
 
```bash
oc apply -f cluster-monitoring-config.yaml
```
 
Verify UWM is running:
 
```bash
oc get pods -n openshift-user-workload-monitoring
```
 
---
 
### Step 2 — Label the Namespace for UWM Discovery
 
By default, the User Workload Monitoring Prometheus only discovers `ServiceMonitor` resources in namespaces that opt in. Label your application namespace:
 
```bash
oc label namespace my-java-app-ns openshift.io/cluster-monitoring=true
```
 
---
 
### Step 3 — Bridge the VM to the Network via a Service
 
OpenShift Virtualization runs each VM inside a **virt-launcher pod**. To scrape the Java application, you need a Kubernetes `Service` that routes traffic to that pod using the KubeVirt label that is automatically applied to it.
 
```yaml
apiVersion: v1
kind: Service
metadata:
  name: java-vm-metrics-service
  namespace: my-java-app-ns
  labels:
    monitor: "true"   # This label is matched by the ServiceMonitor below
spec:
  ports:
    - name: metrics
      port: 8080        # Port exposed by the Service
      targetPort: 8080  # Port the Java app exposes inside the VM — adjust if different
  selector:
    # Targets the virt-launcher pod for this specific VMI
    vm.kubevirt.io/name: my-java-vm
```
 
> **Note:** The `selector` targets the virt-launcher pod via the `vm.kubevirt.io/name` label, which KubeVirt applies automatically. Verify the label is present on your VM's pod with:
> ```bash
> oc get pod -n my-java-app-ns --show-labels | grep virt-launcher
> ```
 
---
 
## Phase 2: Scrape Configuration
 
### Step 4 — Create a ServiceMonitor
 
Create a `ServiceMonitor` to tell the User Workload Monitoring Prometheus which services to scrape. Apply this to the managed cluster in your VM's namespace.
 
```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: java-app-scrape
  namespace: my-java-app-ns
spec:
  selector:
    matchLabels:
      monitor: "true"   # Matches the label on the Service created in Step 3
  endpoints:
    - port: metrics     # Matches the port name defined in the Service
      interval: 30s
      path: /metrics
```
 
Apply it:
 
```bash
oc apply -f java-app-scrape.yaml
```
 
> **RBAC note:** If the MCOA collector does not have permission to read `ServiceMonitor` resources in this namespace, scraping will silently fail. Ensure the MCOA collector's `ClusterRole` includes `get`, `list`, and `watch` on `servicemonitors` in the `monitoring.coreos.com` API group. In most standard RHACM installations this is granted automatically, but verify if you see no targets appearing.
 
---
 
## Phase 3: Hub-Side Federation
 
Apply these resources to the **Hub cluster**.
 
### Step 5 — Allow Your Java Metrics Through to the Hub
 
By default, MCOA filters out user-defined metrics to protect the Hub's storage from cardinality explosion. You must explicitly allowlist your Java/JVM metrics.
 
This is configured via the `observability-metrics-allowlist` ConfigMap in the `open-cluster-management-observability` namespace on the Hub:
 
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: observability-metrics-allowlist
  namespace: open-cluster-management-observability
data:
  metrics_list.yaml: |
    additionalMetrics:
      - jvm_memory_used_bytes
      - java_app_transactions_total
      - process_cpu_seconds_total
```
 
Apply it:
 
```bash
oc apply -f observability-metrics-allowlist.yaml
```
 
> **Note:** If this ConfigMap already exists in your cluster, patch it rather than replacing it to avoid removing existing allowlisted metrics from other applications:
> ```bash
> oc edit configmap observability-metrics-allowlist -n open-cluster-management-observability
> ```
 
Changes take effect within one to two scrape intervals without requiring a restart.
 
---
 
## Phase 4: Verification & Visualization
 
### Step 6 — Verify Local Scraping on the Managed Cluster
 
In the OpenShift Console on the managed cluster:
 
1. Navigate to **Observe → Targets**
2. Search for `java-app-scrape`
3. Confirm the target shows a status of **UP**
Alternatively, query the User Workload Prometheus directly:
 
```bash
oc exec -n openshift-user-workload-monitoring \
  $(oc get pod -n openshift-user-workload-monitoring -l app.kubernetes.io/name=prometheus -o name | head -1) \
  -- curl -s http://localhost:9090/api/v1/targets | jq '.data.activeTargets[] | select(.scrapePool | contains("java-app-scrape")) | .health'
```
 
Expected output: `"up"`
 
---
 
### Step 7 — Verify Metrics Are Reaching the Hub
 
On the Hub cluster, query Thanos to confirm the metrics have been forwarded:
 
```bash
oc exec -n open-cluster-management-observability \
  $(oc get pod -n open-cluster-management-observability -l app.kubernetes.io/name=thanos-query -o name | head -1) \
  -- wget -qO- 'http://localhost:9090/api/v1/query?query=jvm_memory_used_bytes' | jq '.data.result | length'
```
 
A non-zero result confirms the metric is arriving from the managed cluster.
 
---
 
### Step 8 — Visualize in RHACM Grafana
 
1. Open the **RHACM Hub Console**
2. Navigate to **All Clusters → Grafana** (or follow the Grafana route in the `open-cluster-management-observability` namespace)
3. Go to the **Explore** tab
4. Enter a query filtered to your cluster and namespace:
```
jvm_memory_used_bytes{cluster="managed-cluster-name", namespace="my-java-app-ns"}
```
 
Replace `managed-cluster-name` with the name shown in your RHACM cluster list.
 
---
 
## Troubleshooting Quick Reference
 
| Symptom | Likely Cause | Action |
|---|---|---|
| No targets in Observe → Targets | UWM not enabled or namespace not labelled | Re-check Steps 1 and 2 |
| Target shows DOWN | Java app not exposing `/metrics`, wrong port, or VM not running | Check `targetPort` in Service and verify app endpoint |
| Target is UP but metrics absent in Hub Grafana | Metrics not in allowlist | Re-check Step 5; confirm ConfigMap key is `metrics_list.yaml` |
| ServiceMonitor not picked up | Namespace label missing or RBAC gap | Re-check Step 2 and MCOA collector RBAC |
| `jvm_memory_used_bytes` missing | Micrometer Prometheus registry not on classpath | Add `micrometer-registry-prometheus` dependency to the Java app |
 
---
 
## Architecture Summary
 
```
┌─────────────────────────────────────────────┐
│              Managed Cluster                │
│                                             │
│  ┌─────────────┐      ┌──────────────────┐  │
│  │  Java VM    │◄─────│    Service       │  │
│  │  :8080      │      │  (monitor=true)  │  │
│  └─────────────┘      └────────┬─────────┘  │
│                                │             │
│                       ┌────────▼─────────┐  │
│                       │  ServiceMonitor  │  │
│                       └────────┬─────────┘  │
│                                │             │
│                    ┌───────────▼──────────┐  │
│                    │  UWM Prometheus      │  │
│                    │  (Agent mode / MCOA) │  │
│                    └───────────┬──────────┘  │
└────────────────────────────────┼────────────┘
                                 │ remote_write
┌────────────────────────────────▼────────────┐
│                  Hub Cluster                │
│                                             │
│   ┌────────────────────────────────────┐    │
│   │  Thanos Receive / Object Storage   │    │
│   └──────────────────┬─────────────────┘    │
│                      │                      │
│              ┌───────▼──────┐               │
│              │    Grafana   │               │
│              └──────────────┘               │
└─────────────────────────────────────────────┘
```
