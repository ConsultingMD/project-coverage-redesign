# RTE Traffic Monitoring - Observability.yaml Dashboard

**Created:** November 20, 2025
**Issue:** [DP-5538](https://includedhealth.atlassian.net/browse/DP-5538) - Internal DOS: RTE traffic spike
**Location:** `coverage/platform/observability.yaml` (new group: `rte-traffic-monitoring`)

---

## Overview

This monitoring setup tracks all paths to the Real-Time Eligibility (RTE) service through the observability.yaml format, which is the standard way to define Grafana dashboards and alarms at IncludedHealth.

### Why Observability.yaml?

- **Auto-provisioned**: Alarms defined in observability.yaml automatically create Grafana panels in the "Custom Alarms" dashboard
- **Environment-aware**: Separate thresholds and configurations for production vs UAT
- **Integrated alerting**: Alarms can trigger PagerDuty alerts through defined contact points
- **Version-controlled**: All configuration is code-reviewed and deployed with the service
- **Standardized**: Follows IncludedHealth's platform conventions

## Dashboard Access

Once deployed, the RTE monitoring dashboards will appear in Grafana:

1. **Custom Alarms Dashboard**: All alarms auto-generate panels
   - Path: `Grafana > Dashboards > coverage-server > Custom Alarms`
   - Filter by group: "rte-traffic-monitoring"

2. **Direct Grafana Access**: https://grafana.includedhealth.com/

## Monitored Metrics (11 Alarms)

### Critical Alarms (Production)

1. **Total RTE Request Rate from coverage-server**
   - **Threshold**: > 10 req/s (production), > 15 req/s (UAT)
   - **Purpose**: Overall spike detection
   - **Severity**: Critical

2. **cost-share Service RTE Traffic Rate**
   - **Threshold**: > 2.0 req/s
   - **Purpose**: Monitor primary suspect from DP-5538
   - **Baseline**: ~0.5 req/s normal
   - **Severity**: Critical

3. **realtime-eligibility Service Request Rate**
   - **Threshold**: > 50 req/s (production), > 75 req/s (UAT)
   - **Purpose**: Direct RTE service load monitoring
   - **Baseline**: 5-10 req/s normal
   - **Severity**: Critical

### Warning Alarms (Production)

4. **medication Service RTE Traffic Rate**
   - **Threshold**: > 5.0 req/s
   - **Purpose**: Monitor medication service (involved in INC-882)
   - **Severity**: Warning

5. **RTE P95 Latency from coverage-server**
   - **Threshold**: > 15s (production), > 20s (UAT)
   - **Purpose**: Detect Stedi connection saturation (15 connection limit)
   - **Severity**: Warning

6. **RTE Error Rate by Client Service**
   - **Threshold**: > 1.0 errors/s
   - **Purpose**: Identify problematic clients
   - **Severity**: Warning

7. **Batch/Backend RTE Traffic (High-volume services)**
   - **Threshold**: > 10 req/s
   - **Services**: cost-share, medication, revenue-cycle-management, wizard
   - **Severity**: Warning

### Informational Panels (No Alerts)

8. **RTE Request Rate by Client Service**
   - Shows top 10 clients by request rate
   - Use for incident investigation

9. **Frontend RTE Traffic (Mobile Apps)**
   - Aggregates member-ios-app + member-android-app
   - Helps distinguish frontend vs batch traffic

10. **GraphQL Operation rteProxy Call Rate**
    - Monitors primary GraphQL operation for RTE

11. **GraphQL Operation warmRteCache Call Rate**
    - Monitors cache pre-warming/batch operations

## Incident Response Workflow

When an RTE traffic spike alarm fires:

### 1. Identify Offending Client
```promql
# Check "RTE Request Rate by Client Service" panel
topk(10, sum by (service) (rate(traces_spanmetrics_calls_total{
  span_name="HTTP POST coverage.production.grnds.com"
}[5m])))
```

### 2. Check Specific Suspects
- **cost-share**: Primary suspect from DP-5538
- **medication**: Involved in INC-882 (Kafka replay)
- Check hourly spike patterns at :50 past hour (cron job signature)

### 3. Investigate Client
```bash
# Check client logs
tng service logs <service-name> --since 1h

# Check Kafka lag (if applicable)
tng kafka consumer-lag <service-name>

# Check cron schedules
kubectl get cronjobs -n <namespace>
```

### 4. Mitigate
**Temporary:**
- Scale down offending client pods
- Disable batch jobs/cron schedules
- Pause Kafka consumers

**Investigation:**
- Review recent deployments
- Check for config changes
- Analyze error logs for root cause

### 5. Long-term Solution
- Implement traffic control architecture (ACT-2366)
- Add rate limiting per client
- Implement backpressure/admission control
- Add priority metadata for request classification

## Related Incidents

- **DP-5538** (Nov 20, 2025): cost-share 10x traffic spike
- **INC-882** (Nov 2, 2025): medication Kafka consumer DOS
- **INC-864** (Oct 2025): medication Kafka replay
- **INC-824, INC-821** (July 2025): coverage-server traffic surges

## Key Services to Monitor

### High-Risk Batch Services
- **cost-share**: Cost calculation batch jobs
- **medication**: Kafka consumers, batch processing
- **revenue-cycle-management**: Billing batch jobs
- **wizard**: Member flow orchestration

### Mobile Frontend (Low Risk)
- **member-ios-app**: iOS mobile app
- **member-android-app**: Android mobile app

## Deployment

The observability.yaml changes will be deployed with the next coverage-server deployment:

```bash
# Local testing with tng obs tool
cd coverage/
tng obs dev start
# Apply local changes
tng obs apply coverage-server
# Stop local Grafana
tng obs dev stop

# Production deployment
# Changes go live when coverage-server is deployed
# No manual Grafana configuration needed
```

## Queries Reference

### All Prometheus Queries Used

```yaml
# 1. Total RTE Request Rate
sum(rate(traces_spanmetrics_calls_total{
  service="coverage-server",
  span_name=~"HTTP POST.*coverage.*|.*rte.*|.*RTE.*"
}[5m]))

# 2. cost-share Service Rate
sum(rate(traces_spanmetrics_calls_total{
  service="cost-share",
  span_name="HTTP POST coverage.production.grnds.com"
}[5m]))

# 3. Top 10 Clients
topk(10, sum by (service) (rate(traces_spanmetrics_calls_total{
  span_name="HTTP POST coverage.production.grnds.com"
}[5m])))

# 4. medication Service Rate
sum(rate(traces_spanmetrics_calls_total{
  service="medication",
  span_name="HTTP POST coverage.production.grnds.com"
}[5m]))

# 5. RTE P95 Latency
histogram_quantile(0.95, sum by(le) (rate(traces_spanmetrics_latency_bucket{
  service="coverage-server",
  span_name=~".*rte.*|.*RTE.*"
}[5m])))

# 6. Error Rate by Client
sum by (service) (rate(traces_spanmetrics_calls_total{
  span_name="HTTP POST coverage.production.grnds.com",
  status_code=~"5.*"
}[5m]))

# 7. realtime-eligibility Service Rate
sum(rate(traces_spanmetrics_calls_total{
  service="realtime-eligibility"
}[5m]))

# 8. Frontend Traffic (Mobile)
sum(rate(traces_spanmetrics_calls_total{
  service=~"member-ios-app|member-android-app",
  span_name="HTTP POST coverage.production.grnds.com"
}[5m]))

# 9. Batch/Backend Traffic
sum(rate(traces_spanmetrics_calls_total{
  service=~"cost-share|medication|revenue-cycle-management|wizard",
  span_name="HTTP POST coverage.production.grnds.com"
}[5m]))

# 10. GraphQL rteProxy Operation
sum(rate(graphql_requests_total{
  app="coverage-server",
  operation_name="rteProxy"
}[5m]))

# 11. GraphQL warmRteCache Operation
sum(rate(graphql_requests_total{
  app="coverage-server",
  operation_name="warmRteCache"
}[5m]))
```

## Documentation Links

- **Observability.yaml Guide**: https://includedhealth.atlassian.net/wiki/spaces/ENG/pages/4291362860/Getting+Started+with+observability.yaml
- **Observability.yaml Overview**: https://includedhealth.atlassian.net/wiki/spaces/ENG/pages/4291297284/observability.yaml+Overview
- **DP-5538 JIRA Ticket**: https://includedhealth.atlassian.net/browse/DP-5538
- **RTE Paths Analysis**: See `/tmp/DP-5538-RTE-Paths-Report.md`

## Next Steps

1. **Deploy**: Coverage-server deployment will activate the monitoring
2. **Validate**: Check Grafana Custom Alarms dashboard after deployment
3. **Tune Thresholds**: Adjust based on actual production traffic patterns
4. **Add Runbook**: Create detailed runbook for RTE traffic spike response
5. **Implement Traffic Control**: Complete ACT-2366 for long-term solution
