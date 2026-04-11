# CCLM Dashboard Design: Cross-Cluster Live Migration

## Context

Cross-Cluster Live Migration (CCLM) allows running VMs to be moved between OpenShift clusters without downtime. It uses KubeVirt's decentralized live migration with MTV/Forklift orchestration. This dashboard is needed to monitor and manage the migration process and progress.

Reference: [stolostron/multicluster-observability-operator PR #2389](https://github.com/stolostron/multicluster-observability-operator/pull/2389)

## User Story

> As a user performing a Cross-Cluster Live Migration, I want to see what VMs are going to be live-migrated, what VMs have errors, what VMs have already been successfully migrated, and whether there are any bottlenecks.

## Requirements

- All clusters (source and target), cluster names, and namespaces should be shown
- Monitor storage, network bandwidth, CPU, and memory on both source and target clusters
- Detect dangling/pending VMs
- Prevent live migration failures, especially with large VMs and significant storage
- Migration requires isolation, necessitating bandwidth reservation (1-10%)
- Estimation feature for migration duration
- Leverage existing ACM/CNV dashboards (Single VM, Single-Cluster view) where possible
- Dashboard must be in **Perses** format (not Grafana)

---

## Dashboard Layout

### Row 1: Migration Fleet Overview

**Purpose:** At-a-glance status of all CCLM operations across the fleet.

| Panel | Type | Metric / Query |
|---|---|---|
| Total Migrations Active | Stat | `kubevirt_vmi_migrations_in_running_phase` |
| Total Migrations Pending | Stat | `kubevirt_vmi_migrations_in_pending_phase + kubevirt_vmi_migrations_in_scheduling_phase` |
| Total Migrations Succeeded (24h) | Stat | `increase(kubevirt_vmi_migration_succeeded[24h])` |
| Total Migrations Failed (24h) | Stat | `increase(kubevirt_vmi_migration_failed[24h])` |
| Non-Evictable VMs | Stat | `count(kubevirt_vmi_non_evictable == 1)` |
| Dangling/Stuck Migrations | Stat | VMs in migrating state > 5min (see query below) |

**Dangling/Stuck query:**
```promql
(time() - kubevirt_vm_migrating_status_last_transition_timestamp_seconds) > 300
  AND kubevirt_vm_migrating_status_last_transition_timestamp_seconds > 0
```

### Row 2: Migration Queue & Phase Distribution

| Panel | Type | Metric / Query |
|---|---|---|
| Migrations by Phase | Bar Chart | `kubevirt_vmi_migrations_in_pending_phase`, `kubevirt_vmi_migrations_in_scheduling_phase`, `kubevirt_vmi_migrations_in_running_phase`, `kubevirt_vmi_migrations_in_unset_phase` |
| Migration Phase Timeline | TimeSeries | Same metrics over time |
| Migration Rate | TimeSeries | `rate(kubevirt_vmi_migration_succeeded[5m])` and `rate(kubevirt_vmi_migration_failed[5m])` |

### Row 3: Active Migrations Table

**Purpose:** Per-VM detail for all currently migrating VMs.

| Column | Source |
|---|---|
| Cluster (Source) | `cluster` label from federated metrics |
| Namespace | `namespace` label |
| VM Name | `vmi` label |
| Migration Phase | Derived from `kubevirt_vmi_migrations_in_*_phase` or VMIM status |
| Data Total | `kubevirt_vmi_migration_data_bytes_total` |
| Data Processed | `kubevirt_vmi_migration_data_processed_bytes` |
| Data Remaining | `kubevirt_vmi_migration_data_remaining_bytes` |
| Progress % | `kubevirt_vmi_migration_data_processed_bytes / kubevirt_vmi_migration_data_bytes_total * 100` |
| Transfer Rate | `kubevirt_vmi_migration_memory_transfer_rate_bytes` |
| Dirty Rate | `kubevirt_vmi_migration_dirty_memory_rate_bytes` |
| Convergence Ratio | `dirty_rate / transfer_rate` (>1 = risk of non-convergence) |
| Duration | `time() - kubevirt_vmi_migration_start_time_seconds` |
| Estimated Time Remaining | See estimation section below |

### Row 4: Migration Bandwidth & Convergence

| Panel | Type | Metric / Query |
|---|---|---|
| Transfer Rate by VM | TimeSeries | `kubevirt_vmi_migration_memory_transfer_rate_bytes` per VM |
| Dirty Rate by VM | TimeSeries | `kubevirt_vmi_migration_dirty_memory_rate_bytes` per VM |
| Convergence Ratio | TimeSeries | `kubevirt_vmi_migration_dirty_memory_rate_bytes / kubevirt_vmi_migration_memory_transfer_rate_bytes` with threshold line at 1.0 |
| Migration Progress | TimeSeries | `kubevirt_vmi_migration_data_processed_bytes / kubevirt_vmi_migration_data_bytes_total * 100` per VM |

### Row 5: Source Cluster Resources

**Purpose:** Monitor resource pressure on the source cluster to detect bottlenecks.

| Panel | Type | Metric / Query |
|---|---|---|
| Source CPU Utilization | TimeSeries | `instance:node_cpu_utilisation:rate1m{cluster="$source_cluster"}` |
| Source Memory Utilization | TimeSeries | `instance:node_memory_utilisation:ratio{cluster="$source_cluster"}` |
| Source Network Bandwidth | TimeSeries | `instance:node_network_transmit_bytes_excluding_lo:rate1m{cluster="$source_cluster"}` |
| Source Storage I/O | TimeSeries | `kubevirt_vmi_storage_read_traffic_bytes_total + kubevirt_vmi_storage_write_traffic_bytes_total` filtered by source cluster |

### Row 6: Target Cluster Resources

**Purpose:** Monitor resource availability on the target cluster.

| Panel | Type | Metric / Query |
|---|---|---|
| Target CPU Available | Gauge | `kube_node_status_allocatable{resource="cpu", cluster="$target_cluster"}` vs current usage |
| Target Memory Available | Gauge | `kube_node_status_allocatable{resource="memory", cluster="$target_cluster"}` vs current usage |
| Target Network Bandwidth | TimeSeries | `instance:node_network_receive_bytes_excluding_lo:rate1m{cluster="$target_cluster"}` |
| Target Storage Capacity | Gauge | Available PV capacity on target cluster |

### Row 7: Migration History

| Panel | Type | Metric / Query |
|---|---|---|
| Completed Migrations (table) | Table | `kubevirt_vmi_migration_end_time_seconds` joined with start time, data total, duration |
| Migration Duration Distribution | Histogram | `kubevirt_vmi_migration_phase_transition_time_from_creation_seconds` |
| Failed Migrations (table) | Table | VMs where `kubevirt_vmi_migration_failed == 1`, with error context |

### Row 8: Dangling / Pending VMs

**Purpose:** Identify VMs that are stuck, dangling, or have issues preventing migration.

| Panel | Type | Metric / Query |
|---|---|---|
| VMs Stuck in Migrating State | Table | `kubevirt_vm_migrating_status_last_transition_timestamp_seconds` where duration > threshold |
| Non-Evictable VMs | Table | `kubevirt_vmi_non_evictable == 1` with VM details |
| VMs in Error State | Table | `kubevirt_vm_error_status_last_transition_timestamp_seconds > 0` |
| Pending Migrations Age | Table | Migrations in pending/scheduling/unset phase with time elapsed |

---

## Dashboard Variables

| Variable | Type | Query | Description |
|---|---|---|---|
| `cluster` | Query | `label_values(acm_managed_cluster_labels, name)` | All managed clusters |
| `source_cluster` | Query | `label_values(kubevirt_vmi_migration_data_bytes_total, cluster)` | Clusters with active outbound migrations |
| `target_cluster` | Query | `label_values(acm_managed_cluster_labels, name)` | Target cluster for migration |
| `namespace` | Query | `label_values(kubevirt_vmi_info{cluster="$cluster"}, namespace)` | Namespace filter |
| `vm_name` | Query | `label_values(kubevirt_vmi_info{cluster="$cluster", namespace="$namespace"}, name)` | VM filter |

---

## Estimation Feature

### Migration Duration Estimation

Estimate remaining migration time based on current progress rate:

```promql
# Estimated time remaining (seconds)
kubevirt_vmi_migration_data_remaining_bytes / kubevirt_vmi_migration_memory_transfer_rate_bytes
```

### Convergence Feasibility

A migration will converge only if the transfer rate exceeds the dirty rate:

```promql
# Convergence ratio: < 1.0 means migration will converge
kubevirt_vmi_migration_dirty_memory_rate_bytes / kubevirt_vmi_migration_memory_transfer_rate_bytes
```

Display as:
- **Green:** ratio < 0.5 (healthy convergence)
- **Yellow:** ratio 0.5-0.9 (converging but slow)
- **Red:** ratio >= 1.0 (will NOT converge -- intervention needed)

### Bandwidth Reservation Check

Compare migration bandwidth against total available network bandwidth:

```promql
# Migration bandwidth as % of total node bandwidth
kubevirt_vmi_migration_memory_transfer_rate_bytes
  / instance:node_network_transmit_bytes_excluding_lo:rate1m * 100
```

Target: 1-10% reservation. Alert if migration consumes more.

---

## Integration with Existing Dashboards

The CCLM dashboard should include **drill-down links** to existing ACM/CNV dashboards:

| Link Target | When to Link | Parameters |
|---|---|---|
| Single VM View | Click on VM name in migration table | `cluster`, `namespace`, `vm_name` |
| Single Cluster View | Click on source/target cluster name | `cluster` |
| Load Aware Rebalancing | From source/target cluster resource panels | `cluster` |
| VM Inventory | From fleet overview | `cluster`, `namespace` |
| Right-Sizing Virtualization | For VMs with resource issues | `cluster`, `namespace` |

---

## Metrics to Investigate / Request

Some metrics that would be valuable but may not exist yet:

1. **Forklift/MTV specific metrics** -- `forklift_*` metrics from the forklift-controller `/metrics` endpoint (port 2112). Need to document what's available.
2. **CCLM-specific state metrics** -- `virt-synchronization-controller` status/metrics on each cluster
3. **Storage migration progress** -- Separate disk copy progress from memory migration progress
4. **Network quality metrics** -- Latency/jitter between source and target clusters for migration network
5. **CA certificate expiry** -- For the trust relationship between clusters

---

## Perses Implementation Notes

- Deploy as `PersesDashboard` CRD in `observability-virtualization` namespace
- Datasource: `rbac-query-proxy` (HTTP:8080)
- Use `MergeSeries` and `JoinByColumnValue` transforms for metric correlation (consistent with PR #2389 approach)
- Variables use `PrometheusLabelValuesVariable` kind
- Cross-cluster queries rely on RHACM's Thanos aggregation (metrics from all managed clusters available via `cluster` label)

---

## Open Questions

1. What Forklift/MTV metrics are available from the `/metrics` endpoint? Need to inspect a running forklift-controller.
2. Can `virt-synchronization-controller` expose metrics about CCLM state synchronization?
3. Should the dashboard show MTV Migration Plan status, or only KubeVirt-level migration status?
4. What is the best way to correlate source and target cluster metrics for the same CCLM operation?
5. Are there recording rules needed to pre-aggregate cross-cluster migration data in Thanos?
