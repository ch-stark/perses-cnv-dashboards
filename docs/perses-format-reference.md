# Perses Dashboard Format Reference

## What is Perses?

Perses is a CNCF Sandbox project providing an open-source dashboard framework. It is the planned replacement for Grafana dashboards in RHACM's observability stack.

- GitHub: [perses/perses](https://github.com/perses/perses)
- Website: [perses.dev](https://perses.dev)

## Kubernetes CRD: PersesDashboard

In RHACM, Perses dashboards are deployed as `PersesDashboard` Kubernetes Custom Resources.

### Basic Structure

```yaml
apiVersion: perses.dev/v1alpha1
kind: PersesDashboard
metadata:
  name: my-dashboard
  namespace: observability-virtualization
  labels:
    app.kubernetes.io/name: my-dashboard
    app.kubernetes.io/part-of: acm-observability
spec:
  datasources:
    prometheus:
      default: true
      plugin:
        kind: PrometheusDatasource
        spec:
          proxy:
            kind: HTTPProxy
            spec:
              url: http://rbac-query-proxy:8080
  variables: []
  panels: {}
  layouts: []
  duration: 1h
  refreshInterval: 30s
```

## Key Concepts

### Datasources

```yaml
datasources:
  prometheus:
    default: true
    plugin:
      kind: PrometheusDatasource
      spec:
        proxy:
          kind: HTTPProxy
          spec:
            url: http://rbac-query-proxy:8080
```

In RHACM context, the datasource is always `rbac-query-proxy` on port 8080 which provides RBAC-filtered access to Thanos Query.

### Variables

```yaml
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
            name: prometheus
          labelName: cluster
          matchers:
            - kubevirt_vmi_info
```

### Panels

Panels are defined in a `panels` map (keyed by panel ID) and then referenced from `layouts`.

#### TimeSeries Panel

```yaml
panels:
  cpuUtilization:
    spec:
      display:
        name: CPU Utilization
        description: CPU usage over time
      plugin:
        kind: TimeSeriesChart
        spec:
          legend:
            position: bottom
            mode: list
          yAxis:
            format:
              unit: percent
          thresholds:
            steps:
              - value: 80
                color: "#FF0000"
      queries:
        - kind: TimeSeriesQuery
          spec:
            plugin:
              kind: PrometheusTimeSeriesQuery
              spec:
                datasource:
                  kind: PrometheusDatasource
                  name: prometheus
                query: >-
                  rate(kubevirt_vmi_cpu_usage_seconds_total{cluster="$cluster"}[5m]) * 100
                seriesNameFormat: "{{namespace}}/{{name}}"
```

#### Stat Panel

```yaml
panels:
  totalVMs:
    spec:
      display:
        name: Total VMs
      plugin:
        kind: StatChart
        spec:
          calculation: last
          format:
            unit: decimal
          sparkline: {}
      queries:
        - kind: TimeSeriesQuery
          spec:
            plugin:
              kind: PrometheusTimeSeriesQuery
              spec:
                datasource:
                  kind: PrometheusDatasource
                  name: prometheus
                query: >-
                  count(kubevirt_vm_info)
```

#### Table Panel

```yaml
panels:
  vmInventory:
    spec:
      display:
        name: VM Inventory
      plugin:
        kind: Table
        spec:
          density: compact
          columnSettings:
            - name: cluster
              header: Cluster
            - name: namespace
              header: Namespace
            - name: name
              header: VM Name
      queries:
        - kind: TimeSeriesQuery
          spec:
            plugin:
              kind: PrometheusTimeSeriesQuery
              spec:
                datasource:
                  kind: PrometheusDatasource
                  name: prometheus
                query: >-
                  kubevirt_vm_info
```

#### Gauge Panel

```yaml
panels:
  memoryUsage:
    spec:
      display:
        name: Memory Usage
      plugin:
        kind: GaugeChart
        spec:
          calculation: last
          format:
            unit: percent-decimal
          thresholds:
            defaultColor: "#00FF00"
            steps:
              - value: 70
                color: "#FFA500"
              - value: 90
                color: "#FF0000"
      queries:
        - kind: TimeSeriesQuery
          spec:
            plugin:
              kind: PrometheusTimeSeriesQuery
              spec:
                query: >-
                  (1 - kubevirt_vmi_memory_unused_bytes / kubevirt_vmi_memory_available_bytes) * 100
```

### Layouts

Layouts define the spatial arrangement of panels using a grid system.

```yaml
layouts:
  - kind: Grid
    spec:
      display:
        title: Overview
        collapse:
          open: true
      items:
        - x: 0
          y: 0
          width: 6
          height: 4
          content:
            $ref: "#/spec/panels/totalVMs"
        - x: 6
          y: 0
          width: 6
          height: 4
          content:
            $ref: "#/spec/panels/cpuUtilization"
        - x: 0
          y: 4
          width: 12
          height: 8
          content:
            $ref: "#/spec/panels/vmInventory"
```

Grid uses a 24-column system. Items are positioned with `x`, `y`, `width`, `height`.

### Transforms

Used in PR #2389 for correlating metrics:

```yaml
queries:
  - kind: TimeSeriesQuery
    spec:
      plugin:
        kind: PrometheusTimeSeriesQuery
        spec:
          query: "kubevirt_vm_info"
  - kind: TimeSeriesQuery
    spec:
      plugin:
        kind: PrometheusTimeSeriesQuery
        spec:
          query: "kubevirt_vmi_cpu_usage_seconds_total"
transforms:
  - kind: MergeSeries
    spec: {}
  - kind: JoinByColumnValue
    spec:
      columns:
        - name
        - namespace
        - cluster
```

## Differences from Grafana

| Aspect | Grafana | Perses |
|---|---|---|
| Format | JSON ConfigMaps | YAML CRDs (`PersesDashboard`) |
| Deployment | ConfigMap + sidecar loader | Native Kubernetes CRD |
| Datasource | Grafana datasource config | `PrometheusDatasource` plugin |
| Variables | Template variables | `ListVariable` with plugins |
| Layout | Row/Panel model | Grid layout with `$ref` to panels |
| Transforms | Grafana transformations | `MergeSeries`, `JoinByColumnValue` |
| Panel types | Panel plugins | Chart kind plugins (`TimeSeriesChart`, `StatChart`, `Table`, `GaugeChart`) |
| Ecosystem | Mature, large plugin ecosystem | CNCF Sandbox, growing |

## Migration Tools

- **percli** -- Perses CLI tool, can assist with dashboard creation
- **Grafana-to-Perses migration** -- Manual process currently; PR #2389 is an example of manual conversion
- Panel type mapping: Grafana `timeseries` -> Perses `TimeSeriesChart`, Grafana `stat` -> Perses `StatChart`, Grafana `table` -> Perses `Table`, Grafana `gauge` -> Perses `GaugeChart`

## RHACM-Specific Notes

- Target namespace: `observability-virtualization`
- Datasource: `rbac-query-proxy` at `http://rbac-query-proxy:8080`
- All cross-cluster metrics available via `cluster` label (Thanos aggregation)
- Dashboards deployed via MCOA (multicluster-observability-addon)
- Reference: [multicluster-observability-addon](https://github.com/stolostron/multicluster-observability-addon)
