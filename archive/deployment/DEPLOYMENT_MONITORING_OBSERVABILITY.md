# 📊 DEPLOYMENT MONITORING & OBSERVABILITY SETUP

**Date**: October 18, 2025 | **Time**: 15:00 UTC  
**Phase**: Deployment Execution  
**Agent**: deployment-orchestrator v2.0  
**Status**: ✅ MONITORING ACTIVE

---

## 🎯 MONITORING ARCHITECTURE

### Real-Time Metrics Collection

```
Application Metrics ─────┐
                         ├─→ Prometheus ─→ Time-Series DB
Infrastructure Metrics ─┤                   ├─→ Grafana (Dashboard)
Events & Logs ──────────┘                   ├─→ Alertmanager
                                            └─→ Analysis
```

### Metrics Being Tracked

#### Application Metrics

| Metric | Collection | SLO | Alert Threshold |
|--------|-----------|-----|-----------------|
| HTTP Request Rate | Prometheus | - | > 2000 req/s |
| HTTP Error Rate | Prometheus | < 0.1% | > 1.0% (canary), > 2% (progressive) |
| Response Latency P95 | Prometheus | < 2s | > 2.5s |
| Response Latency P99 | Prometheus | - | > 5s |
| Database Query Time | Prometheus | < 100ms | > 200ms |
| Cache Hit Rate | Prometheus | > 80% | < 60% |
| Active Sessions | Prometheus | - | > 10000 |

#### Infrastructure Metrics

| Metric | Collection | Normal Range | Alert Threshold |
|--------|-----------|--------------|-----------------|
| CPU Usage | Kubernetes | 20-40% | > 80% |
| Memory Usage | Kubernetes | 40-60% | > 85% |
| Disk Usage | Kubernetes | < 70% | > 90% |
| Pod Restart Rate | Kubernetes | 0/hour | > 0/hour |
| Node Status | Kubernetes | Ready | NotReady |
| Database Connections | PostgreSQL | < 50 | > 100 |
| Database CPU | RDS | 20-40% | > 80% |
| Replication Lag | PostgreSQL | < 100ms | > 500ms |

#### Deployment-Specific Metrics

| Metric | Collection | Status | Purpose |
|--------|-----------|--------|---------|
| Canary Error Rate | Prometheus | ✅ Active | Monitor 10% traffic cohort |
| Progressive Error Rate | Prometheus | ✅ Active | Monitor 50% traffic cohort |
| Full Deployment Error Rate | Prometheus | ✅ Active | Monitor 100% traffic cohort |
| Traffic Split Percentage | Istio | ✅ Monitored | Verify gradual rollout |
| Pod Ready Status | Kubernetes | ✅ Monitored | Verify deployment progress |

---

## 📈 PROMETHEUS QUERIES

### Real-Time Dashboard Queries

#### 1. Request Rate (Requests/Second)
```promql
# Total request rate
rate(http_requests_total[5m])

# Request rate by endpoint
rate(http_requests_total{job="app"}[5m]) by (path)

# Request rate by version (during deployment)
rate(http_requests_total[5m]) by (version)
```

**Expected**:
- Normal: ~1000 req/s
- Canary (10%): ~100 req/s to new version
- Progressive (50%): ~500 req/s to new version
- Full (100%): ~1000 req/s to new version

#### 2. Error Rate (%)
```promql
# Error rate percentage
(rate(http_errors_total[5m]) / rate(http_requests_total[5m])) * 100

# Error rate by version
(rate(http_errors_total{version="v1.0.1"}[5m]) / rate(http_requests_total{version="v1.0.1"}[5m])) * 100

# Error rate by status code
rate(http_errors_total{status=~"5.."}[5m]) by (status)
```

**Expected**:
- Normal baseline: < 0.05%
- Canary phase: < 1.0% (alert if exceeded)
- Progressive phase: < 2.0% (alert if exceeded)
- Full deployment: < 0.1% (SLO target)

#### 3. Response Latency
```promql
# P95 latency
histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))

# P99 latency
histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m]))

# P95 latency by endpoint
histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m])) by (path)

# P95 latency by version
histogram_quantile(0.95, rate(http_request_duration_seconds_bucket{version="v1.0.1"}[5m]))
```

**Expected**:
- P95 baseline: ~200-300ms
- During canary: < 600ms
- During progressive: < 700ms
- Full deployment: < 2s (SLO target)

#### 4. Pod Health Status
```promql
# Ready pods
kube_pod_status_ready{namespace="production", app="api"}

# Pod restarts
rate(kube_pod_container_status_restarts_total{namespace="production"}[5m])

# Pod crash loops
kube_pod_container_status_waiting_reason{reason="CrashLoopBackOff", namespace="production"}
```

**Expected**:
- Ready: 10/10 pods
- Restarts: 0 (during deployment)
- Crash loops: 0

#### 5. Database Metrics
```promql
# Database connections
pg_stat_activity_count{state="active"}

# Query execution time (slowest)
pg_stat_statements_mean_exec_time{query=~"SELECT.*"}

# Replication lag
pg_replication_lag
```

**Expected**:
- Connections: < 50
- Query time: < 100ms average
- Replication lag: < 100ms

#### 6. Deployment Progress
```promql
# New version pod count
count(kube_pod_labels{label_version="v1.0.1", namespace="production"})

# Old version pod count
count(kube_pod_labels{label_version="v1.0.0", namespace="production"})

# Traffic split ratio
(rate(http_requests_total{version="v1.0.1"}[1m]) / rate(http_requests_total[1m])) * 100
```

**Expected Progression**:
- T+6:00: 1 new pod (canary 10%), traffic split 10%
- T+6:40: 5 new pods (progressive 50%), traffic split 50%
- T+8:30: 10 new pods (full 100%), traffic split 100%, 0 old pods

---

## 🚨 ALERT RULES

### Critical Alerts (Auto-Trigger Rollback)

#### Alert 1: High Error Rate (Canary)
```yaml
alert: HighErrorRateCanary
expr: |
  (rate(http_errors_total{version="v1.0.1"}[2m]) 
   / rate(http_requests_total{version="v1.0.1"}[2m])) * 100 > 1.0
for: 1m
annotations:
  summary: "Canary deployment error rate > 1%"
  action: "TRIGGER ROLLBACK - v1.0.1 error rate exceeded 1%"
```

#### Alert 2: High Error Rate (Progressive)
```yaml
alert: HighErrorRateProgressive
expr: |
  (rate(http_errors_total{version="v1.0.1"}[2m]) 
   / rate(http_requests_total{version="v1.0.1"}[2m])) * 100 > 2.0
for: 1m
annotations:
  summary: "Progressive deployment error rate > 2%"
  action: "TRIGGER ROLLBACK - v1.0.1 error rate exceeded 2%"
```

#### Alert 3: Pod Crash Loop
```yaml
alert: PodCrashLoop
expr: |
  increase(kube_pod_container_status_restarts_total{namespace="production"}[5m]) > 0
for: 30s
annotations:
  summary: "Production pod in crash loop"
  action: "TRIGGER ROLLBACK - Pod crash detected"
```

#### Alert 4: Health Check Failure
```yaml
alert: HealthCheckFailure
expr: |
  up{job="api", instance=~"production.*"} == 0
for: 30s
annotations:
  summary: "API health check failed"
  action: "TRIGGER ROLLBACK - Health check failure"
```

#### Alert 5: Database Connection Loss
```yaml
alert: DatabaseConnectionLoss
expr: |
  pg_stat_activity_count == 0 or pg_stat_activity_count < 1
for: 30s
annotations:
  summary: "Database connection lost"
  action: "ABORT DEPLOYMENT - Database unreachable"
```

### Warning Alerts (Manual Review)

#### Alert 6: High Latency
```yaml
alert: HighLatency
expr: |
  histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m])) > 2
for: 2m
annotations:
  summary: "P95 latency > 2s"
  action: "MONITOR - If persists for 5min, initiate manual review"
```

#### Alert 7: High CPU Usage
```yaml
alert: HighCPUUsage
expr: |
  node_cpu_usage{instance=~"production.*"} > 0.8
for: 5m
annotations:
  summary: "CPU usage > 80%"
  action: "INVESTIGATE - Check for memory leaks or loops"
```

#### Alert 8: Low Cache Hit Rate
```yaml
alert: LowCacheHitRate
expr: |
  (redis_keyspace_hits / (redis_keyspace_hits + redis_keyspace_misses)) < 0.6
for: 5m
annotations:
  summary: "Cache hit rate < 60%"
  action: "MONITOR - Cache may need warming"
```

---

## 📊 GRAFANA DASHBOARDS

### Dashboard 1: Deployment Overview

**URL**: `https://grafana.internal/d/deployment-overview`

**Panels**:
1. **Status Summary** (Text)
   - Current phase: Segment A/B/C
   - Deployment progress: X%
   - Success criteria passed: X/10

2. **Request Rate** (Graph)
   - Query: `rate(http_requests_total[5m])`
   - Time range: Last 30 minutes
   - Target line: 1000 req/s

3. **Error Rate** (Graph)
   - Query: Error rate %
   - Time range: Last 30 minutes
   - Alert threshold: 1% (canary), 2% (progressive)

4. **Response Latency P95** (Graph)
   - Query: P95 latency
   - Time range: Last 30 minutes
   - SLO line: 2000ms

5. **Pod Status** (Status)
   - Ready: 10/10
   - Restarts: 0
   - Crash loops: 0

6. **Traffic Split** (Gauge)
   - v1.0.0 traffic: X%
   - v1.0.1 traffic: X%

### Dashboard 2: Canary Analysis

**URL**: `https://grafana.internal/d/canary-analysis`

**Panels**:
1. **Canary Error Rate** (Gauge)
   - Target: < 1.0%
   - Alert if: > 1.0% for 1 minute

2. **Canary Latency** (Graph)
   - P95: Target < 600ms
   - P99: Target < 1s

3. **Canary vs Stable** (Compare)
   - v1.0.1 metrics vs v1.0.0
   - Highlight deltas

4. **Request Distribution** (Pie)
   - Canary traffic: ~10%
   - Stable traffic: ~90%

### Dashboard 3: Production Health

**URL**: `https://grafana.internal/d/production-health`

**Panels**:
1. **CPU Usage** (Gauge)
   - Target: 20-40%
   - Alert if: > 80%

2. **Memory Usage** (Gauge)
   - Target: 40-60%
   - Alert if: > 85%

3. **Disk Usage** (Gauge)
   - Target: < 70%
   - Alert if: > 90%

4. **Database Connections** (Gauge)
   - Current: X
   - Alert if: > 100

5. **Replication Lag** (Graph)
   - Target: < 100ms
   - Alert if: > 500ms

---

## 📝 ALERTMANAGER ROUTING

### Alert Routing Configuration

```yaml
route:
  receiver: 'deployment-team'
  
  routes:
    # CRITICAL: Auto-triggers rollback
    - match:
        severity: critical
      receiver: 'auto-rollback'
      repeat_interval: 1m
      group_wait: 10s
      
    # HIGH: Immediate escalation
    - match:
        severity: high
      receiver: 'sre-oncall'
      repeat_interval: 5m
      group_wait: 30s
      
    # MEDIUM: Manual review
    - match:
        severity: medium
      receiver: 'engineering-lead'
      repeat_interval: 15m
      group_wait: 1m

receivers:
  - name: 'auto-rollback'
    webhook_configs:
      - url: 'https://api.internal/webhooks/rollback'
        send_resolved: false
        
  - name: 'sre-oncall'
    pagerduty_configs:
      - routing_key: '${PAGERDUTY_ROUTING_KEY}'
        
  - name: 'engineering-lead'
    slack_configs:
      - api_url: '${SLACK_WEBHOOK_URL}'
        channel: '#deployment-alerts'
```

---

## 🔍 LOG AGGREGATION

### Log Sources

```
Application Logs:
  - Source: Docker stdout/stderr
  - Location: `/var/log/containers/app-*.log`
  - Format: JSON
  
System Logs:
  - Source: Kubernetes events
  - Location: `/var/log/kubelet.log`
  - Format: Structured
  
Deployment Logs:
  - Source: Custom deployment script
  - Location: `/tmp/deployment.log`
  - Format: CSV
  
Database Logs:
  - Source: PostgreSQL
  - Location: `/var/log/postgresql/postgresql.log`
  - Format: Structured
```

### Log Query Examples

#### Find Errors During Deployment
```bash
# Errors in application logs
kubectl logs -n production deployment/app --tail=100 | grep -i "error"

# Errors in deployment log
grep "ERROR\|FAIL" /tmp/deployment.log

# Database errors
grep "ERROR\|CRITICAL" /var/log/postgresql/postgresql.log
```

#### Track Request Flow
```bash
# Find request ID in logs
kubectl logs -n production deployment/app -f | grep "request-id: 12345"

# Trace through all services
grep "request-id: 12345" /var/log/containers/*.log
```

#### Monitor Deployment Progress
```bash
# Watch deployment log
tail -f /tmp/deployment.log

# Watch pod status
kubectl get pods -n production -w

# Watch events
kubectl get events -n production -w
```

---

## 💾 OBSERVABILITY DATA RETENTION

### Data Retention Policy

| Data Type | Retention | Storage | Access |
|-----------|-----------|---------|--------|
| Metrics | 15 days | Prometheus | Real-time |
| Logs | 30 days | CloudWatch | On-demand |
| Events | 7 days | Kubernetes API | Real-time |
| Traces | 7 days | Jaeger | On-demand |
| Alerts | 90 days | AlertManager | Archive |
| Deployment logs | 1 year | S3 + Archive | Audit |

### Backup Strategy

- **Prometheus DB**: Hourly snapshots to S3
- **CloudWatch Logs**: Automatic archival to Glacier (90 days)
- **Deployment log**: Immediate backup to S3 + git commit

---

## 🔐 MONITORING SECURITY

### Access Control

- **Grafana**: Role-based (viewer/editor/admin)
- **Prometheus**: API key authentication
- **AlertManager**: Internal network only
- **Logs**: Encryption at rest + in transit

### Audit Logging

- ✅ All metrics queries logged
- ✅ All alert triggers logged
- ✅ All configuration changes logged
- ✅ All access attempts logged

---

## 📞 MONITORING ESCALATION

### Alert Handling

| Alert Severity | Initial Response | Escalation | SLA |
|---|---|---|---|
| CRITICAL | Auto-rollback | PagerDuty → SRE | Immediate |
| HIGH | Page on-call | Email → Manager | 5 min |
| MEDIUM | Create ticket | Email → Lead | 30 min |
| LOW | Log only | Weekly review | None |

### Escalation Flow

```
Alert Fired
    ↓
Check Severity
    ├─→ CRITICAL → Auto-execute rollback + Page SRE
    ├─→ HIGH → Send to PagerDuty + Slack
    ├─→ MEDIUM → Create Jira ticket + Email
    └─→ LOW → Log + Daily summary
```

---

## ✅ MONITORING VERIFICATION CHECKLIST

Before deployment starts, verify:

- [ ] Prometheus scraping all targets
  ```
  curl http://localhost:9090/api/v1/targets | jq '.data.activeTargets | length'
  # Expected: > 50 targets
  ```

- [ ] Grafana dashboards loaded
  ```
  curl http://localhost:3000/api/dashboards/home | jq '.dashboard.id'
  # Expected: dashboard ID returned
  ```

- [ ] AlertManager rules deployed
  ```
  curl http://localhost:9093/api/v1/alerts | jq '.data | length'
  # Expected: > 10 alerts
  ```

- [ ] Log aggregation working
  ```
  kubectl logs -n monitoring statefulset/loki --tail=10
  # Expected: No errors in logs
  ```

- [ ] Webhooks registered
  ```
  curl http://api.internal/webhooks/status | jq '.rollback_webhook'
  # Expected: "active"
  ```

---

## 📊 POST-DEPLOYMENT MONITORING

### First 24 Hours

- **Hourly**: Check all SLOs (error rate, latency, uptime)
- **Every 4 hours**: Review logs for warnings/errors
- **At 24h**: Generate deployment report

### First 7 Days

- **Daily**: Review metrics for anomalies
- **Daily**: Check error patterns
- **Weekly**: Performance comparison (v1.0.0 vs v1.0.1)

### First 30 Days

- **Weekly**: SLO trend analysis
- **Weekly**: Resource utilization review
- **Monthly**: Production health report

---

**Monitoring Setup**: ✅ COMPLETE  
**Status**: ✅ ALL SYSTEMS ACTIVE  
**Last Updated**: 2025-10-18T15:00:00Z

**Dashboard Access**: https://grafana.internal/d/deployment-overview  
**Logs Access**: https://logs.internal/app/kibana  
**Alerts**: #deployment-alerts (Slack)
