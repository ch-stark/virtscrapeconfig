# Technical Overview: Elevating Fleet Observability with MCOA, Health Alerting, and Virtualization Right-Sizing

The transition to the MultiCluster Observability Addon (MCOA) in Red Hat Advanced Cluster Management (RHACM) restructures how metrics are collected, buffered, and monitored across a fleet. By replacing custom legacy components with standard upstream projects, MCOA introduces architectural changes that improve network partition tolerance, pipeline monitoring, and resource capacity planning for both containerized workloads and OpenShift Virtualization virtual machines.

---

## MCOA Architecture and Virtualization Metrics

In the previous architecture, metrics were gathered by a custom metrics-collector deployment. MCOA replaces these components with the upstream Prometheus Operator and a resource-optimized **PrometheusAgent**.

For virtualized environments, MCOA automatically generates and deploys a specific `platform-metrics-virtualization` ScrapeConfig to all managed clusters. This ensures that core KubeVirt performance data is seamlessly federated to the centralized hub without manual configuration.

A primary technical advantage of the PrometheusAgent is its standard remote-write implementation, which utilizes a local **Write Ahead Log (WAL)** on the managed cluster's disk to securely buffer federated metric samples. If a network partition occurs between the managed cluster and the hub cluster, the agent retains the data and will transmit the buffered samples once connectivity is restored, allowing the observability pipeline to tolerate network partitions for **up to two hours** without data loss.

---

## Comprehensive Pipeline Health Detection

Relying solely on the Open Cluster Management (OCM) Addon lease to determine observability health leaves a potential blind spot: the agent pod may be running while silently failing to forward metrics due to network routing or authentication errors.

To provide deterministic health tracking and completely eliminate these silent failures, MCOA deploys specific Prometheus alerting rules directly to the managed clusters to monitor the remote-write pipeline:

- **`MetricsCollectorNotIngestingSamples`** — Fires if the Prometheus agent fails to federate metrics locally from the in-cluster Prometheus.
- **`MetricsCollectorRemoteWriteFailures`** — Fires when the agent experiences a high HTTP failure rate on remote-write requests to the hub.
- **`MetricsCollectorRemoteWriteBehind`** — Fires when the remote-write process is too slow, indicating network latency or hub-side ingestion bottlenecks.

Additionally, a fail-safe alert named **`ManagedClusterMetricsMissing`** is evaluated on the hub cluster. This alert fires if the hub has not received any metrics from a managed cluster for **15 minutes**, despite the cluster's OCM lease reporting it as available.

---

## Strategic VM Resource Optimization (Right-Sizing)

**Right-Sizing Recommendation**, which is Generally Available (GA) in ACM 2.16, is an analytical feature that evaluates historical CPU and memory usage metrics to identify over-provisioned or under-utilized resources. For OpenShift Virtualization, it relies on core KubeVirt metrics like `kubevirt_vmi_cpu_usage_seconds_total` and `kubevirt_vm_resource_requests` to calculate these recommendations.

The feature operates by deploying targeted Prometheus recording rules to the managed clusters. The resulting low-cardinality metrics are federated to the hub and visualized in centralized Grafana dashboards.

### Why Right-Sizing Matters for Virtualization

The Vertical Pod Autoscaler (VPA) is a common tool for optimizing resource requests in Kubernetes, but it **is not supported for OpenShift Virtualization**. VPA targets standard Kubernetes workload controllers (Deployments, StatefulSets, DaemonSets, etc.) and does not work with KubeVirt `VirtualMachine` custom resources. This leaves a gap: without Right-Sizing, there is no automated way to evaluate whether VMs are correctly sized across a fleet.

RHACM Right-Sizing fills this gap as a **strategic advisory layer**. It provides human-validated recommendations based on long-term historical data (days or weeks) and does not automatically mutate workloads. This allows platform administrators and FinOps teams to review fleet-wide efficiency reporting and schedule graceful VM resizing during approved maintenance windows.

For live vertical resizing of individual VMs, KubeVirt provides its own native mechanisms such as **memory hotplug** and **CPU hotplug**, which can adjust resources without rebooting the guest. Right-Sizing complements these by answering the fleet-wide question: *which VMs should be resized, and by how much?*

---

## Enabling MCOA and Right-Sizing

To deploy MCOA and the Right-Sizing recording rules for both namespaces and virtual machines, the following capabilities must be enabled within the `MultiClusterObservability` custom resource (CR) on the hub cluster:

```yaml
apiVersion: observability.open-cluster-management.io/v1beta2
kind: MultiClusterObservability
metadata:
  name: observability
spec:
  capabilities:
    platform:
      analytics:
        namespaceRightSizingRecommendation:
          enabled: true # <-- Enables namespace right-sizing
        virtualizationRightSizingRecommendation:
          enabled: true # <-- Enables VM right-sizing
      metrics:
        default:
          enabled: true # <-- Required to enable MCOA for platform metrics
    userWorkloads:
      metrics:
        default:
          enabled: true # <-- Optional to enable MCOA for user workload metrics
```

When this CR is applied, the `multicluster-observability-addon-manager` generates the required `PrometheusAgent`, `ScrapeConfig`, and `PrometheusRule` resources. It then binds these configurations to the OCM `ClusterManagementAddOn` and `Placement` APIs to distribute the collection architecture down to the managed clusters.
