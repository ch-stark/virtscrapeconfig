# Custom KubeVirt Metrics Collection with ScrapeConfig for ACM Observability

Collecting the right KubeVirt metrics is essential for understanding VM health across a multi-cluster fleet. This guide walks through a `ScrapeConfig` resource that federates key virtualization metrics from spoke cluster Prometheus into the ACM Observability Thanos stack, and provides alerting rules to act on them.

## The ScrapeConfig

This `ScrapeConfig` resource tells the local Prometheus agent on each spoke cluster which KubeVirt metrics to federate back to the hub.

```yaml
apiVersion: monitoring.rhobs/v1alpha1
kind: ScrapeConfig
metadata:
  name: custom-kubevirt-metrics
  namespace: open-cluster-management-agent-addon
  labels:
    app: metrics
    app.kubernetes.io/component: virtualization-collector
spec:
  jobName: kubevirt-enhanced
  scheme: HTTPS
  scrapeClass: ocp-monitoring
  metricsPath: /federate
  params:
    match[]:
      # --- VM Lifecycle & Status ---
      - '{__name__="kubevirt_vmi_phase_count"}'
      - '{__name__="kubevirt_vmi_info"}'
      - '{__name__="kubevirt_vm_status_last_transition_timestamp_seconds"}'

      # --- CPU & Execution ---
      - '{__name__="kubevirt_vmi_cpu_usage_seconds_total"}'
      - '{__name__="kubevirt_vmi_vcpu_wait_seconds_total"}'

      # --- Memory & Ballooning ---
      - '{__name__="kubevirt_vmi_memory_used_bytes"}'
      - '{__name__="kubevirt_vmi_memory_available_bytes"}'
      - '{__name__="kubevirt_vmi_memory_actual_balloon_bytes"}'

      # --- Storage & I/O Performance ---
      - '{__name__="kubevirt_vmi_storage_iops_read_total"}'
      - '{__name__="kubevirt_vmi_storage_iops_write_total"}'
      - '{__name__="kubevirt_vmi_storage_read_traffic_bytes_total"}'
      - '{__name__="kubevirt_vmi_storage_write_traffic_bytes_total"}'
      - '{__name__="kubevirt_vmi_storage_read_times_ms_total"}'
      - '{__name__="kubevirt_vmi_storage_write_times_ms_total"}'

      # --- Network Health ---
      - '{__name__="kubevirt_vmi_network_receive_bytes_total"}'
      - '{__name__="kubevirt_vmi_network_transmit_bytes_total"}'
      - '{__name__="kubevirt_vmi_network_receive_errors_total"}'
      - '{__name__="kubevirt_vmi_network_transmit_errors_total"}'

      # --- Infrastructure Context ---
      - '{__name__="kubevirt_node_schedulable"}'
      - '{__name__="node_cpu_seconds_total"}'

  metricRelabelings:
    # Remove high-cardinality labels not needed for fleet-level dashboards
    - action: labeldrop
      regex: (container|endpoint|instance)
    # Shorten namespace label for cleaner dashboard queries
    - sourceLabels: [kubernetes_namespace]
      targetLabel: vm_namespace
      action: replace

  staticConfigs:
    - targets:
        - prometheus-k8s.openshift-monitoring.svc:9091
```

## Why These Metrics Matter

### CPU Wait Time (`kubevirt_vmi_vcpu_wait_seconds_total`)

This is the most important "noisy neighbor" metric. It tells you if your VM is ready to run but the physical CPU is busy with other tasks. A rising vCPU wait time means the VM is being starved of compute -- even if the guest OS CPU usage looks low.

### Storage Latency (`kubevirt_vmi_storage_read_times_ms_total` / `kubevirt_vmi_storage_write_times_ms_total`)

In virtualization, high disk latency (I/O wait) is the #1 cause of "frozen" guest operating systems. Comparing time totals against IOPS lets you calculate average latency per operation:

```
avg_read_latency = rate(storage_read_times_ms_total) / rate(storage_iops_read_total)
```

### Memory Ballooning (`kubevirt_vmi_memory_actual_balloon_bytes`)

If you are overcommitting memory, this tells you how much the hypervisor has "stolen" back from the guest to keep the node stable. A large balloon value means the VM is under memory pressure from the host side, even if the guest hasn't hit its own OOM threshold yet.

### Network Errors (`kubevirt_vmi_network_receive_errors_total` / `kubevirt_vmi_network_transmit_errors_total`)

Unlike standard pods, VMs are sensitive to MTU mismatches and bridge errors. Tracking `errors_total` is vital for SDN troubleshooting, especially with Multus secondary networks.

## Alerting Rules

The following `PrometheusRule` provides alerts based on the metrics collected above. Deploy this on the spoke clusters alongside the `ScrapeConfig`.

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: kubevirt-vm-alerts
  namespace: openshift-cnv
  labels:
    prometheus: k8s
    role: alert-rules
spec:
  groups:
    - name: kubevirt-cpu
      rules:
        - alert: VMHighVCPUWait
          expr: |
            rate(kubevirt_vmi_vcpu_wait_seconds_total[5m]) > 0.05
          for: 15m
          labels:
            severity: warning
          annotations:
            summary: "VM {{ $labels.name }} has high vCPU wait time"
            description: >-
              VM {{ $labels.name }} in namespace {{ $labels.namespace }} has been
              experiencing vCPU wait > 50ms/s for 15 minutes. This indicates CPU
              contention on the host node -- the VM is ready to run but the
              physical CPU is busy.

        - alert: VMHighCPUUsage
          expr: |
            rate(kubevirt_vmi_cpu_usage_seconds_total[5m]) > 0.9
          for: 30m
          labels:
            severity: warning
          annotations:
            summary: "VM {{ $labels.name }} sustained CPU usage above 90%"
            description: >-
              VM {{ $labels.name }} in namespace {{ $labels.namespace }} has been
              running above 90% CPU utilization for 30 minutes.

    - name: kubevirt-memory
      rules:
        - alert: VMMemoryPressure
          expr: |
            (kubevirt_vmi_memory_used_bytes / kubevirt_vmi_memory_available_bytes) > 0.9
          for: 15m
          labels:
            severity: warning
          annotations:
            summary: "VM {{ $labels.name }} memory usage above 90%"
            description: >-
              VM {{ $labels.name }} in namespace {{ $labels.namespace }} is using
              more than 90% of available memory for 15 minutes.

        - alert: VMBalloonInflated
          expr: |
            kubevirt_vmi_memory_actual_balloon_bytes > 0
          for: 30m
          labels:
            severity: info
          annotations:
            summary: "VM {{ $labels.name }} memory balloon is active"
            description: >-
              VM {{ $labels.name }} in namespace {{ $labels.namespace }} has an
              active memory balloon of {{ $value | humanize1024 }}B. The
              hypervisor is reclaiming memory from this guest, which may degrade
              in-guest performance.

    - name: kubevirt-storage
      rules:
        - alert: VMHighStorageReadLatency
          expr: |
            (rate(kubevirt_vmi_storage_read_times_ms_total[5m])
              / rate(kubevirt_vmi_storage_iops_read_total[5m])) > 20
          for: 15m
          labels:
            severity: warning
          annotations:
            summary: "VM {{ $labels.name }} has high storage read latency"
            description: >-
              VM {{ $labels.name }} in namespace {{ $labels.namespace }} has
              average read latency above 20ms for 15 minutes. This can cause the
              guest OS to appear frozen.

        - alert: VMHighStorageWriteLatency
          expr: |
            (rate(kubevirt_vmi_storage_write_times_ms_total[5m])
              / rate(kubevirt_vmi_storage_iops_write_total[5m])) > 20
          for: 15m
          labels:
            severity: warning
          annotations:
            summary: "VM {{ $labels.name }} has high storage write latency"
            description: >-
              VM {{ $labels.name }} in namespace {{ $labels.namespace }} has
              average write latency above 20ms for 15 minutes.

    - name: kubevirt-network
      rules:
        - alert: VMNetworkErrors
          expr: |
            (rate(kubevirt_vmi_network_receive_errors_total[5m])
              + rate(kubevirt_vmi_network_transmit_errors_total[5m])) > 0
          for: 10m
          labels:
            severity: warning
          annotations:
            summary: "VM {{ $labels.name }} is experiencing network errors"
            description: >-
              VM {{ $labels.name }} in namespace {{ $labels.namespace }} has
              sustained network errors for 10 minutes. Check for MTU mismatches,
              bridge configuration issues, or Multus network problems.

    - name: kubevirt-lifecycle
      rules:
        - alert: VMFrequentPhaseTransitions
          expr: |
            increase(kubevirt_vmi_phase_count[1h]) > 5
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "VM {{ $labels.name }} is cycling through phases"
            description: >-
              VM {{ $labels.name }} in namespace {{ $labels.namespace }} has
              transitioned phases more than 5 times in the last hour. This may
              indicate a crash loop or scheduling instability.

        - alert: NoSchedulableKubeVirtNodes
          expr: |
            sum(kubevirt_node_schedulable) == 0
          for: 5m
          labels:
            severity: critical
          annotations:
            summary: "No schedulable KubeVirt nodes available"
            description: >-
              There are no nodes marked as schedulable for KubeVirt workloads.
              No new VMs can be started until this is resolved.
```

## Alert Summary

| Alert | Severity | Condition | What It Catches |
|---|---|---|---|
| `VMHighVCPUWait` | warning | vCPU wait > 50ms/s for 15m | CPU contention / noisy neighbors |
| `VMHighCPUUsage` | warning | CPU > 90% for 30m | Sustained high utilization |
| `VMMemoryPressure` | warning | Memory > 90% for 15m | Guest memory exhaustion |
| `VMBalloonInflated` | info | Balloon active for 30m | Host reclaiming memory from guest |
| `VMHighStorageReadLatency` | warning | Avg read latency > 20ms for 15m | Slow storage / frozen guest OS |
| `VMHighStorageWriteLatency` | warning | Avg write latency > 20ms for 15m | Slow storage |
| `VMNetworkErrors` | warning | Any RX/TX errors for 10m | MTU mismatch / bridge errors |
| `VMFrequentPhaseTransitions` | warning | >5 phase changes in 1h | Crash loops / scheduling issues |
| `NoSchedulableKubeVirtNodes` | critical | Zero schedulable nodes for 5m | No capacity for new VMs |

## Deployment

### Prerequisites

Ensure the `ManagedClusterAddOn` for observability is active on your spoke clusters. The `ScrapeConfig` tells the local Prometheus agent what to federate back to the central Thanos stack on the hub.

### Apply the resources

```bash
# On each spoke cluster (or distribute via ACM Policy)
oc apply -f scrapeconfig.yaml
oc apply -f prometheusrule.yaml
```

### Distribute via ACM Policy

For fleet-wide deployment, wrap both resources in a `ConfigurationPolicy` with `remediationAction: enforce` and use a `PlacementBinding` to target all clusters running OpenShift Virtualization.
