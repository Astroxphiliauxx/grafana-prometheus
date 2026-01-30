# Grafana Implementation Guide

## Overview

This document explains the complete monitoring stack setup: how data flows from the SmartPay application to Grafana dashboards.

---

## Architecture

```
  ┌──────────────────┐       ┌──────────────────┐       ┌──────────────────┐
  │   1. SMARTPAY    │       │   2. PROMETHEUS  │       │    3. GRAFANA    │
  │   (Spring Boot)  │ ───▶  │   (Time-Series   │ ───▶  │  (Visualization) │
  │                  │ PULL  │     Database)    │ QUERY │                  │
  │   Port: 8080     │       │   Port: 9090     │       │   Port: 3000     │
  └──────────────────┘       └──────────────────┘       └──────────────────┘
         │                           │                          │
         ▼                           ▼                          ▼
    Exposes Metrics            Stores Metrics            Displays Metrics
```

---

## Step 1: SmartPay Generates Metrics

Spring Boot uses **Micrometer** + **Spring Boot Actuator** to collect metrics:

```properties
# application.properties
management.endpoint.prometheus.enabled=true
management.metrics.export.prometheus.enabled=true
management.endpoints.web.base-path=/manage
```

**Metrics collected automatically:**

| Metric Type | Examples |
|-------------|----------|
| JVM | Memory usage, GC pauses, thread counts |
| HTTP | Request count, response times, status codes |
| Hibernate | Queries executed, sessions opened, cache hits |
| HikariCP | Connection pool usage, timeouts |
| Tomcat | Thread pool, request processing time |

**Endpoint:** `http://localhost:8080/manage/prometheus`

---

## Step 2: Prometheus Scrapes Metrics

Prometheus **pulls** metrics every 5 seconds:

```yaml
# prometheus.yml
scrape_configs:
  - job_name: 'smartpay-metrics'
    metrics_path: '/manage/prometheus'
    scrape_interval: 5s
    static_configs:
    - targets: ['host.docker.internal:8080']
```

**Process:**
1. Calls `GET http://host.docker.internal:8080/manage/prometheus` every 5s
2. Parses text-based metrics
3. Stores data with timestamps in time-series database
4. Retains historical data (default: 15 days)

---

## Step 3: Grafana Queries Prometheus

Grafana connects to Prometheus as a data source:

```yaml
# grafana/provisioning/datasources/prometheus.yml
datasources:
  - name: Prometheus
    type: prometheus
    url: http://host.docker.internal:9090
    isDefault: true
```

**Query Flow:**
```
Dashboard Panel → PromQL Query → Prometheus → Data → Grafana Renders Graph
```

---

## Data Flow Diagram

```
   SMARTPAY APP                  PROMETHEUS                    GRAFANA
   ════════════                  ══════════                    ═══════
        │                             │                            │
   [Micrometer]                       │                            │
   collects stats              Every 5 seconds                     │
        │                      ┌──────┴──────┐                     │
        ▼                      │   SCRAPE    │                     │
   ┌─────────┐                 │  (HTTP GET) │                     │
   │ /manage │ ◀───────────────┴─────────────┘                     │
   │/promethe│                       │                             │
   │   us    │                       ▼                      User opens
   └─────────┘               ┌───────────────┐               dashboard
                             │ Time-Series   │                     │
                             │   Database    │              ┌──────┴──────┐
                             └───────────────┘              │   QUERY     │
                                     │                      │  (PromQL)   │
                                     │◀─────────────────────┴─────────────┘
                                     │
                             ┌───────┴───────┐
                             │ Returns data  │───────▶  📊 Graphs
                             └───────────────┘          📈 Charts
```

---

## Key Concepts

| Concept | Description |
|---------|-------------|
| **Pull-based** | Prometheus pulls metrics from app (not push) |
| **Time-series** | Every metric has timestamp for historical analysis |
| **Labels** | Key-value pairs for filtering (e.g., `area="heap"`) |
| **PromQL** | Query language for filtering and aggregating |
| **Scrape interval** | How often Prometheus fetches data (5s) |

---

## Quick Start

```powershell
# Start Prometheus
docker run -d --name prometheus -p 9090:9090 \
  -v "${PWD}/prometheus.yml:/etc/prometheus/prometheus.yml" \
  prom/prometheus:latest

# Start Grafana
cd d:\paynet-universe-nmb\smartpay\smartpay-web\src\main\resources\promethus-config
docker-compose -f docker-compose.grafana.yml up -d
```

## Access URLs

| Service | URL | Credentials |
|---------|-----|-------------|
| SmartPay Metrics | http://localhost:8080/manage/prometheus | - |
| Prometheus | http://localhost:9090 | - |
| Grafana | http://localhost:3000 | admin / admin123 |

## Recommended Dashboards

| Dashboard | ID | Description |
|-----------|-----|-------------|
| JVM Micrometer | 4701 | JVM metrics (memory, threads, GC) |
| Spring Boot Statistics | 6756 | HTTP requests, response times |
