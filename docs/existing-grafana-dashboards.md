# Existing RHACM Grafana Dashboards for CNV

All dashboards are defined in **stolostron/multicluster-observability-operator**:
- Virtualization: `operators/multiclusterobservability/manifests/base/grafana/virtualization/`
- Analytics/Right-Sizing: `operators/multiclusterobservability/manifests/base/grafana/analytics/`
- Platform: `operators/multiclusterobservability/manifests/base/grafana/platform/`

Dashboards are deployed as ConfigMaps. A `grafana-dashboard-loader` sidecar watches ConfigMaps in `open-cluster-management-observability` and loads them into Grafana. Metrics are collected via federation scrape with an allowlist of ~51 KubeVirt/node/infra metrics.

---

## 8 Virtualization Dashboards (`grafana/virtualization/`)

### 1. OpenShift Virtualization Overview

- **File:** `dash-acm-openshift-virtualization-overview.yaml`
- **Purpose:** Executive-level multi-cluster view of OpenShift Virtualization health
- **Panels (18):** Total Clusters, Clusters in Critical/Warning Health, Total VMs, VMs by Status, Clusters by Operator Version, Clusters by OpenShift Version, Running VMs by OS (pie chart), Running VMs by Cluster Top 20, CPU/Memory/Network/Storage Utilization timeseries
- **Key Metrics:**
  - `kubevirt_hyperconverged_operator_health_status`
  - `kubevirt_hco_system_health_status`
  - `kubevirt_vm_info`
  - `kubevirt_vmi_info`
  - `kubevirt_vmi_phase_count`
  - `kubevirt_vmi_cpu_usage_seconds_total`
  - `kubevirt_vmi_memory_available_bytes`
  - `kubevirt_vmi_memory_unused_bytes`
  - `kubevirt_vmi_memory_cached_bytes`
  - `kubevirt_vmi_network_receive_bytes_total`
  - `kubevirt_vmi_network_transmit_bytes_total`
  - `kubevirt_vmi_storage_read_traffic_bytes_total`
  - `kubevirt_vmi_storage_write_traffic_bytes_total`
  - `kubevirt_vmi_storage_iops_read_total`
  - `kubevirt_vmi_storage_iops_write_total`
  - `kubevirt_vm_running_status_last_transition_timestamp_seconds`
  - `cnv:vmi_status_running:count`
  - `kube_node_status_allocatable`
  - `node_cpu_seconds_total`
  - `node_memory_MemTotal_bytes`
  - `csv_succeeded`, `csv_abnormal`
  - `ALERTS`
  - `acm_managed_cluster_labels`

### 2. Single Cluster View

- **File:** `dash-acm-openshift-virtualization-single-cluster-view.yaml`
- **Purpose:** Deep-dive into a single cluster's virtualization posture
- **Panels:** Cluster Name, OpenShift Virt Version, Total Nodes, Total VMs, VMs by Status, Provider, Operator Status/Conditions, Running VMs by OS (pie), Recent VMs Started (table), CPU/Memory/Network Utilization Top 20 (timeseries), Storage IOPS
- **Key Metrics:**
  - `kubevirt_hyperconverged_operator_health_status`
  - `kubevirt_hco_system_health_status`
  - `kubevirt_vm_error_status_last_transition_timestamp_seconds`
  - `kubevirt_vm_running_status_last_transition_timestamp_seconds`
  - `kubevirt_vm_non_running_status_last_transition_timestamp_seconds`
  - `kubevirt_vm_starting_status_last_transition_timestamp_seconds`
  - `kubevirt_vm_migrating_status_last_transition_timestamp_seconds`
  - `kubevirt_vmi_info`, `kubevirt_vmi_phase_count`
  - `kubevirt_vmi_cpu_usage_seconds_total`
  - `kubevirt_vmi_memory_available_bytes`, `kubevirt_vmi_memory_unused_bytes`, `kubevirt_vmi_memory_cached_bytes`, `kubevirt_vmi_memory_used_bytes`
  - `kubevirt_vmi_vcpu_delay_seconds_total`
  - `kubevirt_vmi_network_*`, `kubevirt_vmi_storage_*`
  - `kubevirt_vm_resource_requests`
  - `kube_node_status_allocatable`, `node_cpu_seconds_total`, `node_memory_MemTotal_bytes`
  - `instance:node_cpu_utilisation:rate1m`, `instance:node_memory_utilisation:ratio`, `instance:node_network_*`
  - `csv_succeeded`, `csv_abnormal`
  - `acm_managed_cluster_labels`

### 3. Single VM View

- **File:** `dash-acm-openshift-virtualization-single-vm-view.yaml`
- **Purpose:** Detailed view of a single VM's configuration, resource consumption, active alerts, health
- **Panels:** Status indicators, resource gauges, CPU/memory/network/storage timeseries, filesystem usage, alert summary, snapshot info
- **Key Metrics:**
  - All VM status transition metrics
  - `kubevirt_vm_resource_requests`
  - `kubevirt_vm_cpu_usage_seconds_total`
  - `kubevirt_vm_disk_allocated_size_bytes`
  - `kubevirt_vm_info`, `kubevirt_vmi_info`
  - `kubevirt_vmi_memory_*`, `kubevirt_vmi_cpu_usage_seconds_total`
  - `kubevirt_vmi_vcpu_delay_seconds_total`
  - `kubevirt_vmi_status_addresses`
  - `kubevirt_vmi_filesystem_used_bytes`, `kubevirt_vmi_filesystem_capacity_bytes`
  - `kubevirt_vmi_network_*_packets_dropped_total`
  - `kubevirt_vmi_storage_*`
  - `kubevirt_vmsnapshot_succeeded_timestamp_seconds`
  - `ALERTS`

### 4. Virtual Machines Inventory

- **File:** `dash-acm-virtual-machines-inventory.yaml`
- **Purpose:** Comprehensive VM listing across clusters (24-column table)
- **Panels:** Single large table with cluster, namespace, name, status, creation date, CPU/memory requests, disk size, OS info, IP addresses
- **Key Metrics:**
  - `kubevirt_vm_running_status_last_transition_timestamp_seconds`
  - `kubevirt_vm_non_running_status_last_transition_timestamp_seconds`
  - `kubevirt_vm_error_status_last_transition_timestamp_seconds`
  - `kubevirt_vm_starting_status_last_transition_timestamp_seconds`
  - `kubevirt_vm_migrating_status_last_transition_timestamp_seconds`
  - `kubevirt_vm_resource_requests`
  - `kubevirt_vmi_memory_available_bytes`
  - `kubevirt_vm_create_date_timestamp_seconds`
  - `kubevirt_vm_disk_allocated_size_bytes`
  - `kubevirt_vm_info`, `kubevirt_vmi_info`

### 5. Virtual Machines Resources Utilization

- **File:** `dash-acm-virtual-machines-utilization.yaml`
- **Purpose:** Table-based utilization view showing per-VM CPU, memory, network, storage, filesystem metrics
- **Key Metrics:**
  - All VM status metrics
  - `kubevirt_vmi_cpu_usage_seconds_total`
  - `kubevirt_vmi_vcpu_delay_seconds_total`, `kubevirt_vmi_vcpu_wait_seconds_total`
  - `kubevirt_vmi_memory_available_bytes`, `kubevirt_vmi_memory_unused_bytes`, `kubevirt_vmi_memory_cached_bytes`
  - `kubevirt_vmi_memory_swap_in_traffic_bytes`, `kubevirt_vmi_memory_swap_out_traffic_bytes`
  - `kubevirt_vmi_network_transmit_bytes_total`, `kubevirt_vmi_network_receive_bytes_total`
  - `kubevirt_vmi_storage_read_traffic_bytes_total`, `kubevirt_vmi_storage_write_traffic_bytes_total`
  - `kubevirt_vmi_storage_iops_read_total`, `kubevirt_vmi_storage_iops_write_total`
  - `kubevirt_vmi_filesystem_capacity_bytes`, `kubevirt_vmi_filesystem_used_bytes`
  - `kubevirt_vm_resource_requests`, `kubevirt_vm_info`

### 6. Virtual Machines Service Level

- **File:** `dash-acm-virtual-machines-service-level.yaml`
- **Purpose:** VM uptime/downtime SLA tracking
- **Panels:** Total Uptime %, Planned Downtime %, Unplanned Downtime %, Hours for each, VM list table
- **Key Metrics:**
  - `kubevirt_vm_info`
  - `kubevirt_vm_running_status_last_transition_timestamp_seconds`
  - `kubevirt_vmi_info`
  - `kubevirt_vm_create_date_timestamp_seconds`

### 7. Virtual Machines by Time in Status

- **File:** `dash-acm-virtual-machines-by-time-in-status.yaml`
- **Purpose:** Identifies VMs in unhealthy states, stuck VMs (cleanup candidates)
- **Panels:** Total Allocated CPU/Memory/Disk (stat), VM List by Time In Status (table)
- **Key Metrics:**
  - `kubevirt_vm_resource_requests`
  - `kubevirt_vm_starting_status_last_transition_timestamp_seconds`
  - `kubevirt_vm_running_status_last_transition_timestamp_seconds`
  - `kubevirt_vm_non_running_status_last_transition_timestamp_seconds`
  - `kubevirt_vm_error_status_last_transition_timestamp_seconds`
  - `kubevirt_vm_migrating_status_last_transition_timestamp_seconds`
  - `kubevirt_vm_disk_allocated_size_bytes`
  - `kubevirt_vmi_migration_end_time_seconds`
  - `kubevirt_vmi_info`, `kubevirt_vm_info`

### 8. Load Aware Rebalancing

- **File:** `dash-acm-openshift-virtualization-load-aware-rebalancing.yaml`
- **Purpose:** Monitors Descheduler behavior for rebalancing by real CPU load
- **Panels:** Cluster average CPU utilization, CPU std deviation, Combined CPU utilization and PSI pressure by node, Allocatable/Over-utilized nodes, Node classification, CPU utilization VMs vs non-VM pods, VM Running count, Migration rate, VMI Running by Nodes, Top consumers
- **Key Metrics:**
  - `descheduler:averageworkersutilization:cpu:avg1m`
  - `descheduler:nodeutilization:cpu:avg1m`
  - `descheduler:nodepressure:cpu:avg1m`
  - `kube_node_role`
  - `node_namespace_pod_container:container_cpu_usage_seconds_total:sum_rate5m`
  - `kube_pod_info`
  - `machine_cpu_cores`
  - `kube_node_labels`
  - `kubevirt_vmi_phase_count`
  - `kubevirt_vmi_migration_succeeded`

---

## 4 Right-Sizing / Analytics Dashboards (`grafana/analytics/`)

### 9. ACM Right-Sizing OpenShift Virtualization

- **File:** `dash-acm-right-sizing-virtualization.yaml`
- **Purpose:** Summary of CPU/memory overestimation and underestimation across all VMs
- **Panels (8):** Total CPU/Memory Overestimation (stat), Total CPU/Memory Underestimation (stat), CPU/Memory Over/Underestimation Tables
- **Variables:** cluster, profile, days (1d-90d), namespace
- **Key Metrics:**
  - `acm_rs_vm:namespace:cpu_request`, `acm_rs_vm:namespace:cpu_recommendation`, `acm_rs_vm:namespace:cpu_usage`
  - `acm_rs_vm:namespace:memory_request`, `acm_rs_vm:namespace:memory_recommendation`, `acm_rs_vm:namespace:memory_usage`
  - `kubevirt_vm_running_status_last_transition_timestamp_seconds`

### 10. ACM Right-Sizing VM Overestimation (drill-down)

- **File:** `dash-acm-right-sizing-virtualization-overestimation.yaml`
- **Purpose:** Drill-down into a specific overestimated VM
- **Panels:** CPU/Memory Overestimation stat, Utilization timeseries, Usage/Request/Utilization stats
- **Key Metrics:** `acm_rs_vm:namespace:cpu_*`, `acm_rs_vm:namespace:memory_*`

### 11. ACM Right-Sizing VM Underestimation (drill-down)

- **File:** `dash-acm-right-sizing-virtualization-underestimation.yaml`
- **Purpose:** Drill-down into a specific underestimated VM
- **Key Metrics:** Same `acm_rs_vm:namespace:*` metrics

### 12. ACM Right-Sizing Namespace

- **File:** `dash-acm-right-sizing-namespace.yaml`
- **Purpose:** Namespace-level right-sizing for general workloads (not VM-specific)
- **Key Metrics:**
  - `acm_rs:cluster:cpu_recommendation`, `acm_rs:cluster:cpu_usage`, `acm_rs:cluster:cpu_request`
  - `acm_rs:namespace:cpu_usage`, `acm_rs:namespace:cpu_request`, `acm_rs:namespace:cpu_recommendation`, `acm_rs:namespace:cpu_request_hard`
  - Equivalent `memory_*` variants

---

## Consolidated Metric Families

| Metric Family | Description | Used In |
|---|---|---|
| `kubevirt_vm_info` | VM metadata (OS, flavor, instance type) | Overview, Inventory, Single VM, Utilization, Service Level, Time-in-Status |
| `kubevirt_vmi_info` | VMI metadata (node, IPs, phase) | Overview, Single Cluster, Single VM, Inventory, Service Level, Time-in-Status |
| `kubevirt_vm_*_status_last_transition_timestamp_seconds` | VM status transitions | All 8 virtualization dashboards |
| `kubevirt_vmi_cpu_usage_seconds_total` | VMI CPU usage | Overview, Single Cluster, Single VM, Utilization |
| `kubevirt_vmi_memory_*_bytes` | Memory (available, unused, cached, used, swap) | Overview, Single Cluster, Single VM, Utilization |
| `kubevirt_vmi_network_*_bytes_total` | Network I/O | Overview, Single Cluster, Single VM, Utilization |
| `kubevirt_vmi_storage_*` | Storage I/O and IOPS | Overview, Single Cluster, Single VM, Utilization |
| `kubevirt_vmi_filesystem_*_bytes` | Filesystem capacity and usage | Single VM, Utilization |
| `kubevirt_vm_resource_requests` | CPU/memory resource requests | Inventory, Utilization, Time-in-Status, Single Cluster, Single VM |
| `kubevirt_vm_disk_allocated_size_bytes` | Disk allocation | Inventory, Time-in-Status, Single VM |
| `kubevirt_hyperconverged_operator_health_status` | HCO operator health | Overview, Single Cluster |
| `kubevirt_vmi_phase_count` | VMI phase counts | Overview, Single Cluster, Load Rebalancing |
| `kubevirt_vmi_migration_succeeded` | Migration success | Load Rebalancing |
| `acm_rs_vm:namespace:cpu/memory_*` | Right-sizing recording rules | Right-Sizing dashboards (3) |
| `acm_rs:cluster/namespace:*` | Namespace-level right-sizing | Right-Sizing Namespace |
| `descheduler:*` | Descheduler CPU utilization/pressure | Load Rebalancing |

## Perses Conversion Status (PR #2389)

4 dashboards have initial Perses conversions in [stolostron/multicluster-observability-operator PR #2389](https://github.com/stolostron/multicluster-observability-operator/pull/2389):

| Dashboard | Perses File | Notes |
|---|---|---|
| Virtual Machines Inventory | `acm-virtual-machines-inventory` | Uses MergeSeries + JoinByColumnValue transforms |
| Virtual Machines Service Level | `acm-virtual-machines-service-level` | Review flagged missing cluster filters in variable matchers |
| Virtual Machines by Time in Status | `acm-virtual-machines-by-time-in-status` | Review flagged query syntax inconsistencies and duplicate query |
| Virtual Machines Utilization | `acm-virtual-machines-utilization` | Uses MergeSeries + JoinByColumnValue transforms |

All 4 target namespace `observability-virtualization` with datasource `rbac-query-proxy` (HTTP:8080).
