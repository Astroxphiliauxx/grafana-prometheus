# SmartPay Alert Configuration Guide

This document explains how alerts are configured, processed, and delivered to Telegram for the SmartPay Infrastructure Monitoring system.

---

## Architecture Overview

```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│   SmartPay      │     │   Prometheus    │     │    Grafana      │     │   Telegram      │
│   Application   │────▶│   (Scraper)     │────▶│   (Alerting)    │────▶│   Bot API       │
│                 │     │                 │     │                 │     │                 │
│ /manage/        │     │ Stores metrics  │     │ Evaluates rules │     │ Sends message   │
│ prometheus      │     │ every 5 seconds │     │ every 1 minute  │     │ to group chat   │
└─────────────────┘     └─────────────────┘     └─────────────────┘     └─────────────────┘
```

---

## File Structure

```
promethus-config/
├── prometheus.yml                    # Prometheus scrape configuration
├── docker-compose.grafana.yml        # Grafana container setup
└── grafana/
    └── provisioning/
        ├── datasources/              # Prometheus data source
        ├── dashboards/               # Dashboard JSON files
        └── alerting/
            ├── alerts.yml            # Alert rules definitions
            └── contact-points.yml    # Telegram notification config
```

---

## Step 1: Metric Collection (prometheus.yml)

Prometheus scrapes metrics from SmartPay every 5 seconds:

```yaml
scrape_configs:
  - job_name: 'smartpay-metrics'
    metrics_path: '/manage/prometheus'
    scrape_interval: 5s
    static_configs:
      - targets: ['host.docker.internal:8080']
```

**Key Points:**
- `job_name`: Identifies the source in queries (used in alert rules)
- `metrics_path`: SmartPay exposes metrics at `/manage/prometheus`
- `scrape_interval`: How often metrics are collected
- `targets`: Application host and port

---

## Step 2: Alert Rules (alerts.yml)

Alert rules define WHEN to trigger notifications. Located at:
`grafana/provisioning/alerting/alerts.yml`

### Structure of an Alert Rule

```yaml
groups:
  - orgId: 1                          # Grafana organization ID
    name: SmartPay Infrastructure Alerts
    folder: SmartPay                  # Folder in Grafana UI
    interval: 1m                      # How often rules are evaluated
    rules:
      - uid: high-cpu-alert           # Unique identifier
        title: High CPU Usage         # Display name
        condition: C                  # Which query triggers the alert
        data:
          - refId: A                  # Query reference
            datasourceUid: PBFA97CFB590B2093
            model:
              expr: process_cpu_usage * 100    # PromQL query
          - refId: C                  # Condition evaluation
            datasourceUid: __expr__
            model:
              conditions:
                - evaluator:
                    params: [90]      # Threshold value
                    type: gt          # Greater than
        for: 3m                       # Duration before firing
        labels:
          severity: critical          # Used for routing
        annotations:
          summary: CPU usage above 90% for 3 minutes
```

### How Alert Evaluation Works

1. **Query Execution (refId: A)**: Grafana runs the PromQL query
2. **Condition Check (refId: C)**: Result compared against threshold
3. **Duration Check (for)**: Condition must persist for specified time
4. **State Transition**:
   - `Normal` → `Pending` → `Firing` → `Resolved`

### Current Alert Rules

| Alert | Metric | Threshold | Duration | Severity |
|-------|--------|-----------|----------|----------|
| Application Down | `up{job="smartpay-metrics"}` | < 1 | 30s | Critical |
| High Heap Memory | `jvm_memory_used_bytes{area="heap"}` | > 85% | 5m | Critical |
| High CPU Usage | `process_cpu_usage` | > 90% | 3m | Critical |
| DB Pool Exhausted | `hikaricp_connections_active/max` | > 95% | 1m | Critical |
| High 5xx Error Rate | `http_server_requests{outcome="SERVER_ERROR"}` | > 1% | 2m | Critical |
| High 4xx Error Rate | `http_server_requests{outcome="CLIENT_ERROR"}` | > 10% | 5m | Warning |
| Slow Response Time | `http_server_requests_seconds_sum/count` | > 2s | 3m | Warning |
| No HTTP Traffic | `rate(http_server_requests_seconds_count)` | < 0.001 | 5m | Warning |
| High Error Count | `http_server_requests{outcome=~".*ERROR"}` | > 50 | 1m | Warning |

---

## Step 3: Contact Points (contact-points.yml)

Contact points define WHERE and HOW to send notifications.

```yaml
contactPoints:
  - orgId: 1
    name: SmartPay-Telegram
    receivers:
      - uid: telegram-smartpay
        type: telegram
        settings:
          bottoken: "YOUR_BOT_TOKEN"    # From @BotFather
          chatid: "-123456789"          # Group chat ID (negative for groups)
          parse_mode: HTML              # Enable HTML formatting
          message: |                    # Custom message template
            <b>SMARTPAY ALERT NOTIFICATION</b>
            ════════════════════════════
            
            <b>Alert Status:</b> {{ .Status | toUpper }}
            <b>Alert Name:</b> {{ .Labels.alertname }}
            <b>Severity Level:</b> {{ .Labels.severity | toUpper }}
            
            <b>Description:</b>
            {{ .Annotations.summary }}
            
            <b>Time Detected:</b> {{ .StartsAt.Local.Format "02 Jan 2006, 03:04:05 PM" }}
            ════════════════════════════
```

### Template Variables

| Variable | Description |
|----------|-------------|
| `{{ .Status }}` | Alert status: firing or resolved |
| `{{ .Labels.alertname }}` | Name from alert rule |
| `{{ .Labels.severity }}` | Severity label (critical/warning) |
| `{{ .Annotations.summary }}` | Description from alert rule |
| `{{ .StartsAt }}` | When alert started |
| `{{ .EndsAt }}` | When alert resolved |

---

## Step 4: Notification Policies (contact-points.yml)

Policies control ROUTING based on alert labels:

```yaml
policies:
  - orgId: 1
    receiver: SmartPay-Telegram       # Default receiver
    group_by:
      - alertname                     # Group alerts by name
    group_wait: 30s                   # Wait before first notification
    group_interval: 5m                # Wait between grouped notifications
    repeat_interval: 4h               # Repeat if still firing
    routes:
      - receiver: SmartPay-Telegram
        matchers:
          - severity = critical       # Match critical alerts
        group_wait: 10s               # Faster for critical
        repeat_interval: 1h
      - receiver: SmartPay-Telegram
        matchers:
          - severity = warning        # Match warning alerts
        group_wait: 1m                # Slower for warnings
        repeat_interval: 4h
```

### Timing Explained

| Setting | Purpose |
|---------|---------|
| `group_wait` | Delay before sending first notification (allows grouping) |
| `group_interval` | Minimum time between notifications for same group |
| `repeat_interval` | How often to re-notify if alert still firing |

---

## Complete Alert Flow

```
1. METRIC COLLECTION
   └── SmartPay exposes metrics at /manage/prometheus
   └── Prometheus scrapes every 5 seconds
   └── Metrics stored in time-series database

2. ALERT EVALUATION (every 1 minute)
   └── Grafana queries Prometheus
   └── Evaluates PromQL expressions
   └── Compares against thresholds
   └── Checks duration requirements

3. STATE TRANSITION
   └── Normal: Metric within threshold
   └── Pending: Threshold breached, waiting for 'for' duration
   └── Firing: Duration exceeded, alert triggered
   └── Resolved: Metric returned to normal

4. NOTIFICATION ROUTING
   └── Alert matches policy based on labels (severity)
   └── Grouped with similar alerts
   └── Routed to contact point (SmartPay-Telegram)

5. TELEGRAM DELIVERY
   └── Message template rendered with alert data
   └── HTTP POST to Telegram Bot API
   └── Message appears in configured group chat
```

---

## Adding a New Alert

1. **Edit `alerts.yml`** - Add new rule:
```yaml
- uid: new-alert-id
  title: Your Alert Name
  condition: C
  data:
    - refId: A
      model:
        expr: your_metric_query
    - refId: C
      model:
        conditions:
          - evaluator:
              params: [threshold_value]
              type: gt  # gt, lt, within_range, outside_range
  for: 5m
  labels:
    severity: warning  # or critical
  annotations:
    summary: Description shown in Telegram
```

2. **Restart Grafana**:
```bash
docker-compose -f docker-compose.grafana.yml restart
```

---

## Telegram Bot Setup Reference

1. **Create Bot**: Message @BotFather → `/newbot`
2. **Get Token**: Copy the bot token provided
3. **Create Group**: Add bot to group, make admin
4. **Get Chat ID**: 
   - Visit: `https://api.telegram.org/bot<TOKEN>/getUpdates`
   - Find `"chat":{"id": -XXXXXXXXX}`
5. **Test**: `https://api.telegram.org/bot<TOKEN>/sendMessage?chat_id=<CHAT_ID>&text=Test`

---

## Troubleshooting

| Issue | Solution |
|-------|----------|
| Alerts not firing | Check Prometheus is receiving metrics (`/manage/prometheus`) |
| No Telegram message | Verify bot token and chat ID, test with browser URL |
| Grafana won't start | Check YAML syntax in alerting files |
| Wrong threshold | Edit `alerts.yml`, restart Grafana |

---

*Last Updated: February 2026*
