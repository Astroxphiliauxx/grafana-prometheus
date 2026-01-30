# Custom Grafana Dashboard JSON Guide

This document explains how Grafana dashboard JSON files work and how to customize them.

---

## Dashboard JSON Structure

```
┌─────────────────────────────────────────────────────────────────┐
│                    smartpay-dashboard.json                       │
├─────────────────────────────────────────────────────────────────┤
│  1. Metadata (title, refresh, time range)                       │
│  2. Variables (dropdowns like instance selector)                │
│  3. Panels (individual charts/graphs/stats)                     │
│     └── Each panel has: position, query, visualization type     │
└─────────────────────────────────────────────────────────────────┘
```

---

## Key Sections

### 1. Dashboard Metadata

```json
{
  "title": "SmartPay Metrics Dashboard",   // Dashboard name
  "uid": "smartpay-metrics",               // Unique identifier
  "refresh": "5s",                         // Auto-refresh every 5 seconds
  "time": {
    "from": "now-1h",                      // Show last 1 hour of data
    "to": "now"
  }
}
```

| Property | Description |
|----------|-------------|
| `title` | Dashboard name shown in UI |
| `uid` | Unique ID (used in URLs) |
| `refresh` | Auto-refresh interval (5s, 10s, 1m, etc.) |
| `time.from` | Start of time range |
| `time.to` | End of time range |

---

### 2. Variables (Templating)

```json
"templating": {
  "list": [{
    "name": "instance",
    "type": "query",
    "datasource": "Prometheus",
    "query": "label_values(up{job=\"smartpay-metrics\"}, instance)"
  }]
}
```

**How it works:**
- Creates a **dropdown** at the top of the dashboard
- `name: "instance"` → Use as `$instance` in queries
- `query` → Fetches all `instance` label values from Prometheus
- User selects `host.docker.internal:8080`
- All panels filter data using `$instance`

---

### 3. Panel Structure

Each panel represents one visualization:

```json
{
  "title": "JVM Heap Memory Used",
  "type": "timeseries",
  "gridPos": { 
    "h": 8,
    "w": 12,
    "x": 0,
    "y": 0
  },
  "targets": [{
    "expr": "jvm_memory_used_bytes{area=\"heap\", instance=\"$instance\"}",
    "legendFormat": "{{id}}"
  }],
  "fieldConfig": {
    "defaults": { "unit": "bytes" }
  }
}
```

| Property | Description |
|----------|-------------|
| `title` | Panel heading |
| `type` | Visualization type |
| `gridPos` | Position and size on dashboard |
| `targets` | Data queries (PromQL) |
| `fieldConfig` | Display settings (units, colors) |

---

## Panel Grid Layout

The dashboard uses a **24-column grid system**:

```
 0   6   12   18   24
 │   │    │    │    │
 ├───┴────┼────┴────┤  y=0  (Row 1)
 │ Panel1 │ Panel2  │  w=12 each
 ├────────┼─────────┤  y=8  (Row 2)
 │ Panel3 │ Panel4  │
 └────────┴─────────┘
```

| Grid Property | Values | Meaning |
|---------------|--------|---------|
| `w` | 1-24 | Width (24 = full width, 12 = half) |
| `h` | 1+ | Height in grid units |
| `x` | 0-23 | Horizontal position (0 = left edge) |
| `y` | 0+ | Vertical position (row number) |

**Examples:**
- `w: 24, x: 0` → Full width panel
- `w: 12, x: 0` → Left half
- `w: 12, x: 12` → Right half
- `w: 8, x: 0` → One-third width

---

## Panel Types

| Type | Use Case | Visual |
|------|----------|--------|
| `timeseries` | Metrics over time | 📈 Line graph |
| `stat` | Single value display | 🔢 Big number |
| `gauge` | Percentage/ratio | ⭕ Dial meter |
| `bargauge` | Bar representation | 📊 Horizontal bars |
| `table` | Tabular data | 📋 Table |
| `piechart` | Distribution | 🥧 Pie chart |

---

## PromQL Query Examples

### Basic Metric Query
```promql
jvm_memory_used_bytes{area="heap", instance="$instance"}
│                      │             │
│                      │             └── Filter by selected instance
│                      └── Filter by label
└── Metric name
```

### Rate Calculation (for counters)
```promql
rate(http_server_requests_seconds_count{instance="$instance"}[1m])
│    │                                                        │
│    │                                                        └── Over last 1 minute
│    └── Counter metric
└── Calculate per-second rate
```

### Average Response Time
```promql
rate(http_server_requests_seconds_sum[1m]) / rate(http_server_requests_seconds_count[1m])
│                                          │
└── Total time spent                       └── ÷ request count = average
```

### Percentage Calculation
```promql
(metric_success / (metric_success + metric_failure)) * 100
```

---

## Common Units

Use in `fieldConfig.defaults.unit`:

| Unit | Description |
|------|-------------|
| `bytes` | Auto-formats to KB, MB, GB |
| `s` | Seconds |
| `ms` | Milliseconds |
| `percent` | Percentage (0-100) |
| `reqps` | Requests per second |
| `short` | Plain number |
| `none` | No unit |

---

## Auto-Provisioning

```
grafana/provisioning/
├── datasources/
│   └── prometheus.yml           ← Configures Prometheus connection
└── dashboards/
    ├── dashboards.yml           ← Points to JSON files location
    └── smartpay-dashboard.json  ← Dashboard definition
```

**dashboards.yml:**
```yaml
apiVersion: 1
providers:
  - name: 'default'
    folder: ''
    type: file
    options:
      path: /etc/grafana/provisioning/dashboards
```

When Grafana starts:
1. Reads `dashboards.yml`
2. Scans the specified path
3. Loads all `.json` files as dashboards

---

## How to Add a New Panel

1. Copy an existing panel object
2. Modify these properties:
   - `title` → New panel name
   - `gridPos` → New position
   - `targets[0].expr` → New PromQL query
   - `type` → Visualization type if different

**Example - Add CPU Panel:**
```json
{
  "title": "System CPU Usage",
  "type": "gauge",
  "gridPos": { "h": 4, "w": 6, "x": 0, "y": 48 },
  "targets": [{
    "expr": "system_cpu_usage{instance=\"$instance\"} * 100",
    "legendFormat": "CPU %"
  }],
  "fieldConfig": {
    "defaults": { 
      "unit": "percent",
      "max": 100,
      "min": 0
    }
  }
}
```

---

## Quick Reference

| Task | What to Modify |
|------|----------------|
| Change title | `"title": "..."` |
| Add new panel | Add object to `"panels": [...]` |
| Change query | Modify `"expr": "..."` |
| Resize panel | Adjust `gridPos.w` and `gridPos.h` |
| Move panel | Adjust `gridPos.x` and `gridPos.y` |
| Change chart type | Modify `"type": "..."` |
| Add variable | Add object to `"templating.list": [...]` |

---

## Useful Metrics for SmartPay

| Category | Metric | PromQL |
|----------|--------|--------|
| JVM Memory | Heap Used | `jvm_memory_used_bytes{area="heap"}` |
| JVM Threads | Live Count | `jvm_threads_live_threads` |
| HTTP | Request Rate | `rate(http_server_requests_seconds_count[1m])` |
| HTTP | Avg Response | `rate(http_server_requests_seconds_sum[1m]) / rate(http_server_requests_seconds_count[1m])` |
| Database | Active Connections | `hikaricp_connections_active` |
| Hibernate | Queries/sec | `rate(hibernate_query_executions_total[1m])` |
| Tomcat | Busy Threads | `tomcat_threads_busy_threads` |
