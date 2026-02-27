# Book Management App - Observability

This document describes the observability stack implemented for the Book Management Application, integrated with Grafana Cloud.

## Overview

The application has comprehensive observability including:
- **Logs** - Application logs shipped to Grafana Loki
- **Traces** - Distributed tracing via OpenTelemetry
- **Profiles** - Continuous profiling via Pyroscope
- **Metrics** - Application and infrastructure metrics
- **Frontend Observability** - Core Web Vitals and RUM
- **Database Observability** - PostgreSQL query analysis

## Architecture

```
┌─────────────────┐     ┌──────────────────┐     ┌─────────────────┐
│   Frontend      │     │   Backend        │     │   PostgreSQL    │
│   (Next.js)     │     │   (NestJS)       │     │   Database      │
└────────┬────────┘     └────────┬─────────┘     └────────┬────────┘
         │                        │                        │
         │  OpenTelemetry         │  OpenTelemetry         │
         │  Faro (RUM)           │  Pyroscope             │  postgres_exporter
         │                        │                        │
         └────────────────────────┼────────────────────────┘
                                  │
                                  ▼
                    ┌─────────────────────────┐
                    │    Grafana Alloy        │
                    │    (Collector)         │
                    │  - OTLP Receiver       │
                    │  - Loki Source         │
                    │  - Pyroscope Receiver  │
                    │  - Prometheus Scraper  │
                    └───────────┬─────────────┘
                                │
                                ▼
              ┌─────────────────────────────────────────┐
              │           Grafana Cloud                │
              │  - Loki (Logs)                         │
              │  - Tempo (Traces)                      │
              │  - Pyroscope (Profiles)               │
              │  - Prometheus (Metrics)              │
              │  - Frontend Cloud (RUM)               │
              └─────────────────────────────────────────┘
```

## Components

### 1. Backend Observability (NestJS)

**Location:** `backend/src/lib/tracing.ts`

**Features:**
- OpenTelemetry tracing with auto-instrumentation
- Pyroscope continuous profiling
- Custom Prometheus metrics

**Metrics Exposed:**
- `books_created_total` - Counter for books created
- `books_updated_total` - Counter for books updated
- `books_deleted_total` - Counter for books deleted
- `http_request_duration_seconds` - HTTP request duration histogram
- `database_query_duration_seconds` - Database query duration histogram

**Environment Variables:**
```yaml
OTEL_EXPORTER_OTLP_ENDPOINT: "http://grafana-alloy:4317"
OTEL_SERVICE_NAME: "bookmanagement-backend"
OTEL_SERVICE_VERSION: "1.0.0"
PYROSCOPE_URL: "http://grafana-alloy:9999"
PYROSCOPE_APP_NAME: "bookmanagement-backend"
```

### 2. Frontend Observability (Next.js)

**Location:** `frontend/src/components/frontend-observability.tsx`

**Features:**
- Grafana Faro SDK integration
- Core Web Vitals tracking
- Frontend error tracking
- Page performance monitoring
- Distributed tracing (e2e)

**Environment Variables:**
```yaml
NEXT_PUBLIC_FARO_URL: "https://faro-collector-prod-ap-southeast-1.grafana.net/collect/xxx"
NEXT_PUBLIC_FARO_APP_NAME: "bookmanagement-frontend"
```

### 3. Grafana Alloy Collector

**Location:** `k8s/grafana-alloy-config.yaml`

**Features:**
- OTLP receiver for traces and metrics (gRPC: 4317, HTTP: 4318)
- Loki source for Kubernetes log collection
- Pyroscope receiver for profiles (port 9999)
- Prometheus scraping for pod metrics
- PostgreSQL exporter scraping

**Ports:**
| Port | Protocol | Purpose |
|------|----------|---------|
| 12345 | HTTP | Alloy metrics |
| 4317 | gRPC | OTLP traces/metrics |
| 4318 | HTTP | OTLP traces/metrics |
| 9999 | HTTP | Pyroscope profiles |

### 4. Database Observability

**Components:**
- `postgres_exporter` - Metrics collection
- Database Observability (db-oiy) - Query-level telemetry

**Collected:**
- Query execution times
- Query samples
- Schema details
- Explain plans
- Database locks and stats

## Kubernetes Resources

| Resource | Purpose |
|----------|---------|
| `namespace.yaml` | bookmanagement namespace |
| `configmap.yaml` | Application configuration |
| `grafana-alloy-deployment.yaml` | Alloy collector |
| `grafana-alloy-config.yaml` | Alloy configuration |
| `backend-deployment.yaml` | Backend with observability |
| `frontend-deployment.yaml` | Frontend with Faro |
| `postgres-deployment.yaml` | PostgreSQL with exporter |

## Setup

### Prerequisites
- Kubernetes cluster (minikube, kind, or cloud)
- Grafana Cloud account with:
  - Loki
  - Tempo
  - Prometheus
  - Pyroscope
  - Frontend Cloud

### Deploy

1. Apply Kubernetes manifests:
```bash
kubectl apply -f k8s/namespace.yaml
kubectl apply -f k8s/configmap.yaml
kubectl apply -f k8s/grafana-alloy-deployment.yaml
kubectl apply -f k8s/grafana-alloy-config.yaml
kubectl apply -f k8s/backend-deployment.yaml
kubectl apply -f k8s/frontend-deployment.yaml
kubectl apply -f k8s/postgres-deployment.yaml
```

2. Create secret (or use secret.example.yaml):
```bash
kubectl apply -f k8s/secret.example.yaml
```

### Verify Deployment

```bash
# Check pods
kubectl get pods -n bookmanagement

# Check Alloy logs
kubectl logs -n bookmanagement deploy/grafana-alloy | head -20

# Check backend observability
kubectl logs -n bookmanagement deploy/backend | grep -i "otel\|pyroscope"
```

## Grafana Cloud Dashboards

### Accessing Observability Data

| Data Type | Grafana Cloud Service |
|-----------|---------------------|
| Logs | Loki - `/{stack-name}/explore` |
| Traces | Tempo - `/{stack-name}/explore` |
| Profiles | Pyroscope - `/{stack-name}/profiles` |
| Metrics | Prometheus - `/{stack-name}/explore` |
| Frontend | Frontend Cloud - `/{stack-name}/frontend` |

### Key Queries

**Backend Logs:**
```logql
{app="bookmanagement", container="backend"}
```

**Backend Traces:**
```promql
service_name="bookmanagement-backend"
```

**CPU Profiles:**
```pyroscope
bookmanagement-backend{cpu=true}
```

**Database Query Latency:**
```promql
rate(pg_stat_database_tup_fetched_total[5m])
```

## Troubleshooting

### No Logs in Grafana
1. Check pod labels include `app.k8s.io/name: bookmanagement`
2. Verify Alloy is running: `kubectl get pods -n bookmanagement`
3. Check Alloy logs: `kubectl logs -n bookmanagement deploy/grafana-alloy`

### No Traces
1. Verify OTLP endpoint: `http://grafana-alloy:4317`
2. Check backend logs for "OpenTelemetry initialized"

### No Profiles
1. Verify Pyroscope URL: `http://grafana-alloy:9999`
2. Check Alloy logs for pyroscope errors

### Authentication Errors (401)
1. Verify Grafana Cloud tokens in secret
2. Check PYROSCOPE_AUTH_TOKEN is valid
3. Ensure GRAFANA_CLOUD_TOKEN is set

## Security

- Secrets are stored in `k8s/secret.example.yaml` (do not commit)
- `.gitignore` excludes sensitive files
- Use Grafana Cloud tokens with minimal required permissions

## Additional Resources

- [Grafana Alloy Documentation](https://grafana.com/docs/alloy/)
- [OpenTelemetry JavaScript](https://opentelemetry.io/docs/js/)
- [Pyroscope Documentation](https://grafana.com/docs/pyroscope/)
- [Grafana Faro](https://grafana.com/docs/grafana-cloud/frontend-observation/)
