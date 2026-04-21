# Monitoring ODF Across Your Fleet with RHACM: From Metrics Collection to Perses Dashboard

OpenShift Data Foundation (ODF) is the storage backbone for many OpenShift Virtualization deployments, providing Ceph-based persistent storage for VM disks. When you manage multiple clusters with Red Hat Advanced Cluster Management (RHACM), you need fleet-wide visibility into ODF health and capacity across all your spoke clusters — not just one at a time.

This guide walks through the end-to-end process of building a multicluster ODF monitoring dashboard in RHACM 2.16+. There are four steps:

1. **Select** the ODF metrics you want to collect
2. **Create** a ScrapeConfig to federate them from each spoke cluster
3. **Reference** the ScrapeConfig in the ClusterManagementAddOn so it gets deployed fleet-wide
4. **Create** a Perses dashboard to visualize the data on the Hub

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│  Hub Cluster                                                    │
│  ┌───────────────┐    ┌──────────────┐    ┌──────────────────┐  │
│  │ Perses         │◄───│ Thanos Query │◄───│ Thanos Receive   │  │
│  │ Dashboard      │    │              │    │                  │  │
│  └───────────────┘    └──────────────┘    └────────▲─────────┘  │
│                                                    │            │
│  ClusterManagementAddOn ──► pushes ScrapeConfig    │            │
│                              to all spoke clusters │            │
└────────────────────────────────────────────────────┼────────────┘
                                                     │ remote-write
┌────────────────────────────────────────────────────┼────────────┐
│  Spoke Cluster                                     │            │
│  ┌──────────────┐    ┌──────────────────┐    ┌─────┴──────────┐ │
│  │ ODF / Ceph    │───►│ Prometheus       │───►│ PrometheusAgent│ │
│  │ exporters     │    │ (platform)       │    │ (MCOA)         │ │
│  └──────────────┘    └──────────────────┘    └────────────────┘ │
│                        ▲                                        │
│                        │ /federate                              │
│                      ScrapeConfig                               │
│                      (odf-custom-metrics)                       │
└─────────────────────────────────────────────────────────────────┘
```

The Multicluster Observability Addon (MCOA) deploys a PrometheusAgent on each spoke cluster. A `ScrapeConfig` tells that agent which ODF metrics to federate from the spoke's local Prometheus. The agent remote-writes them to Thanos on the Hub, where a Perses dashboard queries them across all clusters.

## Step 1: Select Your ODF Metrics

ODF exposes dozens of Ceph metrics, but for a fleet overview dashboard you typically need just a handful. Start with these core metrics:

| Metric | Type | Description |
|---|---|---|
| `odf_system_health_status` | Gauge | Overall ODF system health (0=OK, 1=Warning, 2=Error) |
| `odf_system_map` | Gauge | Mapping of ODF system components and their relationships |
| `odf_system_raw_capacity_total_bytes` | Gauge | Total physical storage capacity |
| `odf_system_raw_capacity_used_bytes` | Gauge | Physical storage currently in use |
| `csv_succeeded` | Gauge | Whether the ODF Operator CSV is installed and healthy |

The `csv_succeeded` metric is a good practice addition — it lets you quickly see which clusters actually have ODF installed and running by filtering for the `odf-operator` CSV in the `openshift-storage` namespace.

**Tip:** Keep the metric list small. Every metric you add increases the remote-write volume across all spoke clusters. Start with what you need for the dashboard, and add more later.

## Step 2: Create the ScrapeConfig

The `ScrapeConfig` tells the MCOA PrometheusAgent on each spoke cluster to federate the selected metrics from the spoke's local Prometheus. Create this resource on the Hub — it will be distributed to spokes automatically in the next step.

```yaml
apiVersion: monitoring.rhobs/v1alpha1
kind: ScrapeConfig
metadata:
  name: odf-custom-metrics
  namespace: open-cluster-management-observability
  labels:
    app: metrics
    app.kubernetes.io/component: platform-metrics-collector
    app.kubernetes.io/managed-by: multicluster-observability-addon-manager
    app.kubernetes.io/part-of: multicluster-observability-addon
    app.kubernetes.io/version: 1.0.0
    chart: metrics-1.0.0
    release: multicluster-observability-addon
  annotations:
    operator.prometheus.io/controller-id: acm-observability
spec:
  jobName: odf-metrics-federation
  metricsPath: /federate
  params:
    match[]:
      - '{__name__="odf_system_health_status"}'
      - '{__name__="odf_system_map"}'
      - '{__name__="odf_system_raw_capacity_total_bytes"}'
      - '{__name__="odf_system_raw_capacity_used_bytes"}'
      - '{__name__="csv_succeeded",exported_namespace="openshift-storage",name=~"odf-operator.*"}'
  scheme: HTTPS
  scrapeClass: non-ocp-monitoring
  staticConfigs:
    - targets:
        - acm-prometheus-k8s.open-cluster-management-agent-addon.svc:9091
  tlsConfig:
    ca: {}
    cert: {}
    insecureSkipVerify: true
```

Key points about this configuration:

- **`metricsPath: /federate`** — This uses Prometheus federation to pull metrics from the spoke's local Prometheus rather than scraping ODF exporters directly. Federation is the standard pattern in MCOA.
- **`params.match[]`** — Each entry is a PromQL matcher selecting which metrics to federate. Use label filters like `exported_namespace="openshift-storage"` to avoid pulling unrelated metrics with the same name.
- **`staticConfigs.targets`** — Points to the MCOA Prometheus proxy running on the spoke at `acm-prometheus-k8s.open-cluster-management-agent-addon.svc:9091`.
- **`scrapeClass: non-ocp-monitoring`** — Indicates this target is not part of the standard OpenShift platform monitoring stack.
- **Labels** — The `app.kubernetes.io/*` labels are required for the MCOA addon manager to recognize and manage this ScrapeConfig.

## Step 3: Reference the ScrapeConfig in the ClusterManagementAddOn

The ScrapeConfig from Step 2 exists on the Hub, but it will not be pushed to spoke clusters until you reference it in the `ClusterManagementAddOn` resource. This is the central configuration that controls what gets deployed across your fleet.

Edit the `multicluster-observability-addon` ClusterManagementAddOn and add your ScrapeConfig to the `installStrategy.placements[0].configs` list:

```bash
oc edit clustermanagementaddon multicluster-observability-addon
```

Add the following entry to the `configs` array under the `global` placement:

```yaml
spec:
  installStrategy:
    placements:
    - configs:
      # ... existing configs like platform-metrics, platform-metrics-kubevirt, etc.
      - group: monitoring.rhobs
        name: odf-custom-metrics
        namespace: open-cluster-management-observability
        resource: scrapeconfigs
      name: global
      namespace: open-cluster-management-global-set
      rolloutStrategy:
        type: All
    type: Placements
```

After saving, the addon manager will roll out the ScrapeConfig to all managed clusters in the global placement. You can verify the rollout status:

```bash
# Check the addon status — look for odf-custom-metrics in configReferences
oc get clustermanagementaddon multicluster-observability-addon -o yaml | grep -A5 odf-custom-metrics
```

Once the rollout completes, you should see the ScrapeConfig on each spoke cluster:

```bash
# On a spoke cluster
oc get scrapeconfigs -n open-cluster-management-agent-addon odf-custom-metrics
```

And verify that ODF metrics are flowing to the Hub:

```bash
# On the Hub, query Thanos via the rbac-query-proxy
curl -s "http://rbac-query-proxy.open-cluster-management-observability.svc:8080/api/v1/query?query=odf_system_health_status" | jq '.data.result[].metric.cluster'
```

## Step 4: Create the Perses Dashboard

With ODF metrics now flowing from all spoke clusters to Thanos on the Hub, you can create a Perses dashboard to visualize them. The dashboard is deployed as a `PersesDashboard` Custom Resource.

```yaml
apiVersion: perses.dev/v1alpha2
kind: PersesDashboard
metadata:
  name: acm-odf-multicluster-overview
  namespace: observability-virtualization
  labels:
    app.kubernetes.io/name: acm-odf-multicluster-overview
    app.kubernetes.io/part-of: acm-observability
    app.kubernetes.io/component: storage
spec:
  config:
    display:
      name: ACM ODF Multi-Cluster Overview
      duration: 1h
      refreshInterval: 30s
    variables:
      - kind: ListVariable
        spec:
          name: cluster
          display:
            name: Cluster
            hidden: false
            allowAllValue: true
            allowMultiple: true
          plugin:
            kind: PrometheusLabelValuesVariable
            spec:
              datasource:
                kind: PrometheusDatasource
                name: rbac-query-proxy-datasource
              labelName: cluster
              matchers:
                - odf_system_health_status
    panels:
      odfHealth:
        kind: Panel
        spec:
          display:
            name: ODF System Health
            description: "0 = OK, 1 = Warning, 2 = Error"
          plugin:
            kind: StatChart
            spec:
              calculation: last
              format:
                unit: decimal
              thresholds:
                defaultColor: "#00FF00"
                steps:
                  - value: 1
                    color: "#FFA500"
                  - value: 2
                    color: "#FF0000"
          queries:
            - kind: TimeSeriesQuery
              spec:
                plugin:
                  kind: PrometheusTimeSeriesQuery
                  spec:
                    query: odf_system_health_status{cluster=~"$cluster"}
                    seriesNameFormat: "{{cluster}}"
      cephHealth:
        kind: Panel
        spec:
          display:
            name: Ceph Health
            description: "0 = OK, 1 = Warning, 2 = Error"
          plugin:
            kind: StatChart
            spec:
              calculation: last
              format:
                unit: decimal
              thresholds:
                defaultColor: "#00FF00"
                steps:
                  - value: 1
                    color: "#FFA500"
                  - value: 2
                    color: "#FF0000"
          queries:
            - kind: TimeSeriesQuery
              spec:
                plugin:
                  kind: PrometheusTimeSeriesQuery
                  spec:
                    query: ceph_health_status{cluster=~"$cluster"}
                    seriesNameFormat: "{{cluster}}"
      totalCapacity:
        kind: Panel
        spec:
          display:
            name: Total Raw Capacity
            description: Total physical capacity across the fleet
          plugin:
            kind: TimeSeriesChart
            spec:
              legend:
                position: bottom
              yAxis:
                format:
                  unit: bytes
          queries:
            - kind: TimeSeriesQuery
              spec:
                plugin:
                  kind: PrometheusTimeSeriesQuery
                  spec:
                    query: odf_system_raw_capacity_total_bytes{cluster=~"$cluster"}
                    seriesNameFormat: "{{cluster}}"
      usedCapacity:
        kind: Panel
        spec:
          display:
            name: Used Raw Capacity
            description: Physical storage currently in use
          plugin:
            kind: TimeSeriesChart
            spec:
              legend:
                position: bottom
              yAxis:
                format:
                  unit: bytes
          queries:
            - kind: TimeSeriesQuery
              spec:
                plugin:
                  kind: PrometheusTimeSeriesQuery
                  spec:
                    query: odf_system_raw_capacity_used_bytes{cluster=~"$cluster"}
                    seriesNameFormat: "{{cluster}}"
      capacityUtilization:
        kind: Panel
        spec:
          display:
            name: Capacity Utilization (%)
            description: Percentage of raw capacity in use
          plugin:
            kind: TimeSeriesChart
            spec:
              legend:
                position: bottom
              yAxis:
                format:
                  unit: percent
              thresholds:
                defaultColor: "#00FF00"
                steps:
                  - value: 75
                    color: "#FFA500"
                  - value: 85
                    color: "#FF0000"
          queries:
            - kind: TimeSeriesQuery
              spec:
                plugin:
                  kind: PrometheusTimeSeriesQuery
                  spec:
                    query: >-
                      (odf_system_raw_capacity_used_bytes{cluster=~"$cluster"}
                      / odf_system_raw_capacity_total_bytes{cluster=~"$cluster"}) * 100
                    seriesNameFormat: "{{cluster}}"
      odfFleetTable:
        kind: Panel
        spec:
          display:
            name: ODF Fleet Status Summary
            description: Health and utilization across all clusters
          plugin:
            kind: TimeSeriesTable
            spec: {}
          queries:
            - kind: TimeSeriesQuery
              spec:
                plugin:
                  kind: PrometheusTimeSeriesQuery
                  spec:
                    query: odf_system_health_status{cluster=~"$cluster"}
                    seriesNameFormat: "{{cluster}} - Health"
            - kind: TimeSeriesQuery
              spec:
                plugin:
                  kind: PrometheusTimeSeriesQuery
                  spec:
                    query: odf_system_raw_capacity_total_bytes{cluster=~"$cluster"}
                    seriesNameFormat: "{{cluster}} - Total Capacity"
            - kind: TimeSeriesQuery
              spec:
                plugin:
                  kind: PrometheusTimeSeriesQuery
                  spec:
                    query: >-
                      (odf_system_raw_capacity_used_bytes{cluster=~"$cluster"}
                      / odf_system_raw_capacity_total_bytes{cluster=~"$cluster"}) * 100
                    seriesNameFormat: "{{cluster}} - Util %"
    layouts:
      - kind: Grid
        spec:
          display:
            title: Fleet Health
          collapse:
            open: true
          items:
            - x: 0
              'y': 0
              width: 12
              height: 4
              content:
                $ref: "#/spec/panels/odfHealth"
            - x: 12
              'y': 0
              width: 12
              height: 4
              content:
                $ref: "#/spec/panels/cephHealth"
      - kind: Grid
        spec:
          display:
            title: Fleet Storage Capacity
          collapse:
            open: true
          items:
            - x: 0
              'y': 0
              width: 8
              height: 6
              content:
                $ref: "#/spec/panels/totalCapacity"
            - x: 8
              'y': 0
              width: 8
              height: 6
              content:
                $ref: "#/spec/panels/usedCapacity"
            - x: 16
              'y': 0
              width: 8
              height: 6
              content:
                $ref: "#/spec/panels/capacityUtilization"
      - kind: Grid
        spec:
          display:
            title: Fleet Details
          collapse:
            open: true
          items:
            - x: 0
              'y': 0
              width: 24
              height: 6
              content:
                $ref: "#/spec/panels/odfFleetTable"
```

Apply the dashboard:

```bash
oc apply -f odfdashboard.yaml
```

## End-to-End Summary

The complete flow looks like this:

| Step | What | Where | Resource |
|---|---|---|---|
| 1 | Choose ODF metrics | Planning | `odf_system_health_status`, `odf_system_raw_capacity_*`, `csv_succeeded` |
| 2 | Create ScrapeConfig | Hub (`open-cluster-management-observability` namespace) | `ScrapeConfig/odf-custom-metrics` |
| 3 | Reference in addon | Hub | `ClusterManagementAddOn/multicluster-observability-addon` |
| 4 | Create dashboard | Hub (`observability-virtualization` namespace) | `PersesDashboard/acm-odf-multicluster-overview` |

Once all four steps are in place, the addon manager pushes the ScrapeConfig to every managed cluster in the global placement. Each spoke's PrometheusAgent federates the ODF metrics from local Prometheus and remote-writes them to Thanos on the Hub. The Perses dashboard queries Thanos and displays fleet-wide ODF health and capacity — all without needing to log into individual clusters.

## Extending the Dashboard

This starter dashboard covers fleet health and raw capacity. To go deeper, add more ODF/Ceph metrics to your ScrapeConfig and build additional panels:

- **IOPS and throughput** — `ceph_pool_rd`, `ceph_pool_wr`, `ceph_pool_rd_bytes`, `ceph_pool_wr_bytes`
- **OSD status** — `ceph_osd_up`, `ceph_osd_in`, `ceph_osd_apply_latency_ms`, `ceph_osd_commit_latency_ms`
- **PG health** — `ceph_pg_active`, `ceph_pg_degraded`, `ceph_pg_undersized`
- **Pool-level capacity** — `ceph_pool_bytes_used`, `ceph_pool_max_avail`
- **PVC usage per VM** — Correlate `kubevirt_vmi_storage_*` metrics with `kube_persistentvolumeclaim_info` to link VM disk I/O back to ODF pools

Remember: every metric you add to the ScrapeConfig increases remote-write volume across your fleet. Add incrementally, and monitor cardinality.
