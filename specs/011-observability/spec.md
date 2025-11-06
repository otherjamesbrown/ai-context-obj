# Spec 011: Observability (LGTM Stack)

## Purpose

Deploy and configure the LGTM stack (Loki, Grafana, Tempo, Mimir/Prometheus) for logs, metrics, and traces.

## Dependencies

- **001-infrastructure**: LKE cluster running
- All services (005-007) deployed and instrumented

## Success Criteria

- [ ] Prometheus collecting metrics from all services
- [ ] Grafana dashboards for each service
- [ ] Loki collecting logs from all pods
- [ ] Tempo receiving traces
- [ ] Alerts configured for critical issues

## Components

### 1. Prometheus (Metrics Collection)
```bash
helm install prometheus prometheus-community/kube-prometheus-stack \
  --namespace ias-system \
  --set prometheus.prometheusSpec.serviceMonitorSelectorNilUsesHelmValues=false
```

### 2. Grafana (Visualization)
Included with kube-prometheus-stack.

Dashboards:
- System Overview
- User Service Metrics
- API Router Performance
- Analytics Service Health
- vLLM GPU Utilization

### 3. Loki (Log Aggregation)
```bash
helm install loki grafana/loki-stack \
  --namespace ias-system \
  --set promtail.enabled=true \
  --set loki.persistence.enabled=true
```

### 4. Tempo (Distributed Tracing)
```bash
helm install tempo grafana/tempo \
  --namespace ias-system \
  --set tempo.receivers.otlp.protocols.grpc.enabled=true
```

## Service Instrumentation

All services already export:
- `/metrics` - Prometheus metrics
- Structured JSON logs to stdout
- OpenTelemetry traces

## Grafana Dashboards

### 1. System Overview Dashboard
- Total requests/second
- Error rate (%)
- P95 latency by service
- Active organizations
- Total tokens processed

### 2. API Router Dashboard
- Requests by model
- Budget exceeded rate
- Latency histogram
- vLLM backend health

### 3. Database Dashboard
- Connection pool utilization
- Query latency
- Slow queries (>1s)
- Table sizes

## Alerts (Basic V1)

```yaml
# Prometheus AlertManager rules
groups:
  - name: ias_alerts
    rules:
      - alert: HighErrorRate
        expr: rate(http_requests_total{status=~"5.."}[5m]) > 0.05
        annotations:
          summary: "High error rate detected"
      
      - alert: HighLatency
        expr: histogram_quantile(0.95, http_request_duration_seconds) > 1
        annotations:
          summary: "P95 latency > 1s"
      
      - alert: DatabaseDown
        expr: up{job="postgres"} == 0
        annotations:
          summary: "PostgreSQL is down"
```

## Contracts

See `contracts/` for:
- `helm-values/` - Helm values for each component
- `dashboards/` - Grafana dashboard JSON
- `alerts.yaml` - Prometheus alert rules

## Access

```bash
# Port-forward to Grafana
kubectl port-forward -n ias-system svc/prometheus-grafana 3000:80

# Login: admin / <password-from-secret>
```

## Next Steps

After observability is set up:
- Monitor system health
- Debug issues with traces
- Optimize based on metrics

