``
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
    # Example: Ensure the 'container' label is removed to reduce cardinality if not needed
    - action: labeldrop
      regex: (container|endpoint|instance)
    # Example: Replace a long label name with something shorter for dashboards
    - sourceLabels: [kubernetes_namespace]
      targetLabel: vm_namespace
      action: replace

  staticConfigs:
    - targets:
        - prometheus-k8s.openshift-monitoring.svc:9091
``
Why these metrics matter

Wait Time (vcpu_wait_seconds_total): This is the most important "noisy neighbor" metric. It tells you if your VM is ready to run but the physical CPU is busy with other tasks.

Storage Latency (storage_read/write_times_ms_total): In virtualization, high disk latency (I/O wait) is the #1 cause of "frozen" guest operating systems. Comparing time totals against IOPS lets you calculate average latency per op.

Memory Ballooning (actual_balloon_bytes): If you are overcommitting memory, this tells you how much the hypervisor has "stolen" back from the guest to keep the node stable.

Network Errors: Unlike standard pods, VMs are sensitive to MTU mismatches and bridge errors. Tracking errors_total is vital for SDN troubleshooting.

Deployment Tip
When applying this in an ACM (Advanced Cluster Management) environment, ensure your ManagedClusterAddOn for observability is active. This ScrapeConfig will tell the local Prometheus agent on the spoke cluster what to send back to the central Thanostack.

Would you like me to add specific alerting rules (PrometheusRules) based on these metrics as well?
