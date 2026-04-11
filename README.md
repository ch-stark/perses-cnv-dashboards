# Perses CNV Dashboards for RHACM

Perses-based dashboards for Red Hat Advanced Cluster Management (RHACM) OpenShift Virtualization monitoring.

## Goal

Migrate 8 existing Grafana dashboards from the [multicluster-observability-operator](https://github.com/stolostron/multicluster-observability-operator) to Perses format, and create 1 new dashboard for Cross-Cluster Live Migration (CCLM).

## Dashboards

### Existing Dashboards to Migrate (Grafana -> Perses)

| # | Dashboard | Source File | Status |
|---|---|---|---|
| 1 | OpenShift Virtualization Overview | `dash-acm-openshift-virtualization-overview.yaml` | Planned |
| 2 | Single Cluster View | `dash-acm-openshift-virtualization-single-cluster-view.yaml` | Planned |
| 3 | Single VM View | `dash-acm-openshift-virtualization-single-vm-view.yaml` | Planned |
| 4 | Virtual Machines Inventory | `dash-acm-virtual-machines-inventory.yaml` | Planned |
| 5 | Virtual Machines Utilization | `dash-acm-virtual-machines-utilization.yaml` | Planned |
| 6 | Virtual Machines Service Level | `dash-acm-virtual-machines-service-level.yaml` | Planned |
| 7 | Virtual Machines by Time in Status | `dash-acm-virtual-machines-by-time-in-status.yaml` | Planned |
| 8 | Load Aware Rebalancing | `dash-acm-openshift-virtualization-load-aware-rebalancing.yaml` | Planned |

### New Dashboard

| # | Dashboard | Status |
|---|---|---|
| 9 | Cross-Cluster Live Migration (CCLM) | Planned |

Note: 4 of the 8 dashboards (Inventory, Utilization, Service Level, Time-in-Status) have initial Perses conversions in [stolostron/multicluster-observability-operator PR #2389](https://github.com/stolostron/multicluster-observability-operator/pull/2389).

## Related Work

- [Analysis: poor-mans-drs vs Right-Sizing for Virt](docs/analysis-poor-mans-drs-vs-rightsizing.md)
- [CCLM Dashboard Design](docs/cclm-dashboard-design.md)
- [Migration Metrics Reference](docs/migration-metrics-reference.md)
- [Perses Dashboard Format Reference](docs/perses-format-reference.md)
- [Existing Grafana Dashboards Inventory](docs/existing-grafana-dashboards.md)

## Source Repository

All existing Grafana dashboards live in:
```
stolostron/multicluster-observability-operator
  operators/multiclusterobservability/manifests/base/grafana/virtualization/
  operators/multiclusterobservability/manifests/base/grafana/analytics/
```

Perses conversions target:
```
stolostron/multicluster-observability-operator
  operators/multiclusterobservability/manifests/base/perses/virtualization/
```

## Deployment

Dashboards are deployed as `PersesDashboard` Custom Resources in the `observability-virtualization` namespace, using `rbac-query-proxy` (HTTP:8080) as the Prometheus datasource.
