# KubeVirt Migration Prometheus Metrics Reference

Complete reference of all Prometheus metrics available for monitoring KubeVirt VM live migrations, including cross-cluster live migration (CCLM).

---

## Live Migration Data/Progress Metrics (virt-handler)

Real-time migration progress metrics sourced from libvirt domain job statistics, exposed by `virt-handler`. They use the Prometheus Collector interface pattern (computed at scrape time).

| Metric Name | Type | Labels | Description |
|---|---|---|---|
| `kubevirt_vmi_migration_data_bytes_total` | Counter | `namespace`, `vmi`, `node` | Total Guest OS data to be migrated. Replaces deprecated `kubevirt_vmi_migration_data_total_bytes`. |
| `kubevirt_vmi_migration_data_processed_bytes` | Gauge | `namespace`, `vmi`, `node` | Total Guest OS data processed and migrated so far. |
| `kubevirt_vmi_migration_data_remaining_bytes` | Gauge | `namespace`, `vmi`, `node` | Remaining guest OS data to be migrated. |
| `kubevirt_vmi_migration_dirty_memory_rate_bytes` | Gauge | `namespace`, `vmi`, `node` | Rate of memory being dirtied in the Guest OS (bytes/sec). |
| `kubevirt_vmi_migration_memory_transfer_rate_bytes` | Gauge | `namespace`, `vmi`, `node` | Rate at which memory is being transferred (bytes/sec). Replaces incorrectly named `kubevirt_vmi_migration_disk_transfer_rate_bytes`. |

---

## Migration Lifecycle/Status Metrics (virt-controller)

Set by virt-controller based on VMI Migration custom resource state.

| Metric Name | Type | Labels | Description |
|---|---|---|---|
| `kubevirt_vmi_migration_succeeded` | Gauge | `namespace`, `vmi`, `node` | Indicates if the VMI migration succeeded (1 = yes). |
| `kubevirt_vmi_migration_failed` | Gauge | `namespace`, `vmi`, `node` | Indicates if the VMI migration failed (1 = yes). |
| `kubevirt_vmi_migration_start_time_seconds` | Gauge | `namespace`, `vmi`, `node` | Unix timestamp at which the migration started. |
| `kubevirt_vmi_migration_end_time_seconds` | Gauge | `namespace`, `vmi`, `node` | Unix timestamp at which the migration ended. |

---

## Migration Phase Transition Metrics

| Metric Name | Type | Labels | Description |
|---|---|---|---|
| `kubevirt_vmi_migration_phase_transition_time_from_creation_seconds` | Histogram | `namespace`, `vmi`, `phase` | Histogram of VM migration phase transition duration from creation time. Useful for understanding how long migrations spend in each phase. |

---

## Migration Queue/State Metrics (virt-controller)

Cluster-level gauges tracking current number of migrations in each state. No per-VMI labels.

| Metric Name | Type | Description |
|---|---|---|
| `kubevirt_vmi_migrations_in_pending_phase` | Gauge | Number of current pending migrations. |
| `kubevirt_vmi_migrations_in_running_phase` | Gauge | Number of current running migrations. |
| `kubevirt_vmi_migrations_in_scheduling_phase` | Gauge | Number of current scheduling migrations. |
| `kubevirt_vmi_migrations_in_unset_phase` | Gauge | Number of current migrations in unset phase (pending items not yet processed by virt-controller). |

---

## VM-Level Migration Status Metrics

| Metric Name | Type | Labels | Description |
|---|---|---|---|
| `kubevirt_vm_migrating_status_last_transition_timestamp_seconds` | Counter | `name`, `namespace` | VM last transition timestamp to migrating status. |
| `kubevirt_vmi_non_evictable` | Gauge | `namespace`, `vmi`, `node` | Indicates a VM with eviction strategy set to Live Migration but not migratable (e.g., uses local storage, hostdev). |

---

## Dirty Rate Metric (Pre-Migration Assessment)

| Metric Name | Type | Labels | Description |
|---|---|---|---|
| `kubevirt_vmi_dirty_rate_bytes_per_second` | Gauge | `namespace`, `vmi`, `node` | Guest dirty-rate in bytes/sec. Reported via `GetDomainDirtyRateStats()`. Use before migration to estimate convergence feasibility. |

---

## Related VMI Phase Transition Metrics

Not migration-specific but useful when analyzing migration lifecycle.

| Metric Name | Type | Labels | Description |
|---|---|---|---|
| `kubevirt_vmi_phase_transition_time_from_creation_seconds` | Histogram | `namespace`, `vmi`, `phase` | VMI phase transition duration from creation. |
| `kubevirt_vmi_phase_transition_time_from_deletion_seconds` | Histogram | `namespace`, `vmi`, `phase` | VMI phase transition duration from deletion. |
| `kubevirt_vm_non_running_status_last_transition_timestamp_seconds` | Counter | `name`, `namespace` | VM last transition to paused/stopped. |
| `kubevirt_vm_running_status_last_transition_timestamp_seconds` | Counter | `name`, `namespace` | VM last transition to running. |
| `kubevirt_vm_starting_status_last_transition_timestamp_seconds` | Counter | `name`, `namespace` | VM last transition to starting. |

---

## Migration-Related Prometheus Alerts

| Alert Name | Condition | Severity | Description |
|---|---|---|---|
| `KubeVirtVMIExcessiveMigrations` | VMI migrates >12 times in 24h | Warning | Indicates infrastructure problems (network disruptions, resource shortage). |
| `KubeVirtVMStuckInMigratingState` | VM in migrating state >5 min | Warning | **DEPRECATED** -- potential infrastructure problems. |
| `OutdatedVirtualMachineInstanceWorkloads` | Outdated VMIs 24h after update | Warning | VMIs in old virt-launcher pods; live-migrate to update. |
| `VMCannotBeEvicted` | VM has eviction strategy but cannot migrate | Warning | Related to `kubevirt_vmi_non_evictable`. |

---

## Deprecated Migration Metrics

| Old Name | Replaced By | Notes |
|---|---|---|
| `kubevirt_vmi_migration_data_total_bytes` | `kubevirt_vmi_migration_data_bytes_total` | Renamed for Prometheus naming conventions. |
| `kubevirt_vmi_migration_disk_transfer_rate_bytes` | `kubevirt_vmi_migration_memory_transfer_rate_bytes` | Name was incorrect; tracks memory, not disk. |
| `kubevirt_migrate_vmi_data_processed_bytes` | `kubevirt_vmi_migration_data_processed_bytes` | Older naming convention. |
| `kubevirt_migrate_vmi_data_remaining_bytes` | `kubevirt_vmi_migration_data_remaining_bytes` | Older naming convention. |
| `kubevirt_migrate_vmi_dirty_memory_rate_bytes` | `kubevirt_vmi_migration_dirty_memory_rate_bytes` | Older naming convention. |

---

## Useful PromQL Queries for Migration Monitoring

```promql
# Migration duration (seconds)
kubevirt_vmi_migration_end_time_seconds - kubevirt_vmi_migration_start_time_seconds

# Current migration bandwidth (bytes/sec)
kubevirt_vmi_migration_memory_transfer_rate_bytes

# Migration convergence ratio (dirty rate vs transfer rate)
# Values > 1 mean migration may not converge
kubevirt_vmi_migration_dirty_memory_rate_bytes / kubevirt_vmi_migration_memory_transfer_rate_bytes

# Migration progress percentage
kubevirt_vmi_migration_data_processed_bytes / kubevirt_vmi_migration_data_bytes_total

# Excessive migrations (>12 in 24h)
changes(kubevirt_vmi_migration_succeeded[24h]) > 12

# Non-evictable VMs (migration risk)
kubevirt_vmi_non_evictable == 1

# Total pending migration backlog
kubevirt_vmi_migrations_in_pending_phase + kubevirt_vmi_migrations_in_scheduling_phase + kubevirt_vmi_migrations_in_unset_phase

# VMs currently migrating
kubevirt_vm_migrating_status_last_transition_timestamp_seconds > kubevirt_vm_running_status_last_transition_timestamp_seconds

# Migration rate (migrations completed per hour)
increase(kubevirt_vmi_migration_succeeded[1h])

# Average migration duration over last hour
avg(kubevirt_vmi_migration_end_time_seconds - kubevirt_vmi_migration_start_time_seconds)
```

---

## Cross-Cluster Live Migration Notes

KubeVirt cross-cluster live migration (CCLM) uses **decentralized live migration**:

- Two separate VMI resources and two VMIM resources (one per cluster)
- A `virt-synchronization-controller` pod on each cluster coordinates state
- The same `kubevirt_vmi_migration_*` metrics apply on both source and target clusters
- To get a unified view, aggregate metrics across clusters via Prometheus Federation or Thanos (as RHACM does)
- MTV/Forklift exposes metrics on `/metrics` endpoint (port 2112) of forklift-controller, but specific `forklift_*` metric names are not publicly documented yet

### CCLM Migration Phases (VMIM status)

1. Pending
2. Scheduling
3. Scheduled
4. PreparingTarget
5. TargetReady
6. Running
7. Succeeded / Failed

### CCLM Requirements

- OpenShift Virtualization 4.20+ / MTV 2.10+
- Same L2 network segment between clusters
- CA certificate exchange between clusters
- Feature gates: `decentralizedLiveMigration` (HyperConverged CR), `feature_ocp_live_migration` (ForkliftController CR)
- Default limits: 5 parallel migrations, 2 outbound per node, 64 MiB/s bandwidth

---

## Authoritative Sources

- [kubevirt.io/monitoring/metrics.html](http://kubevirt.io/monitoring/metrics.html)
- [kubevirt/kubevirt/docs/observability/metrics.md](https://github.com/kubevirt/kubevirt/blob/main/docs/observability/metrics.md)
- [kubevirt/monitoring/docs/metrics.md](https://github.com/kubevirt/monitoring/blob/main/docs/metrics.md)
- [KubeVirt Component Monitoring](https://kubevirt.io/user-guide/user_workloads/component_monitoring/)
- [KubeVirtVMIExcessiveMigrations Runbook](http://kubevirt.io/monitoring/runbooks/KubeVirtVMIExcessiveMigrations.html)
- [PR #7474 -- Original migration metrics](https://github.com/kubevirt/kubevirt/pull/7474)
- [PR #13325 -- Node label on migration metrics](https://github.com/kubevirt/kubevirt/pull/13325)
- [PR #13423 -- kubevirt_vmi_migration_data_bytes_total](https://github.com/kubevirt/kubevirt/pull/13423)
- [PR #13428 -- start/end time metrics](https://github.com/kubevirt/kubevirt/pull/13428)
- [OKD 4.20: Cross-cluster live migration](https://docs.okd.io/4.20/virt/live_migration/virt-enabling-cclm-for-vms.html)
- [KubeVirt Decentralized Live Migration](https://kubevirt.io/user-guide/compute/decentralized_live_migration/)
