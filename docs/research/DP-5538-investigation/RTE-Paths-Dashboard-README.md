# RTE Paths Dashboard - RED Metrics

**Created:** November 20, 2025
**Purpose:** Monitor all paths to RTE service through coverage-server using RED metrics
**Related:** DP-5538 Investigation

---

## Dashboard Overview

This Grafana dashboard provides comprehensive RED metrics (Rate, Errors, Duration) for all client services that call coverage-server → RTE.

### Key Features

- **Rate Metrics:** Requests per second by client service
- **Error Metrics:** Error rate (%) and error counts by status code
- **Duration Metrics:** P50, P95, P99 latency by client service
- **RTE Service Metrics:** Direct RTE request rate and latency
- **Alerts:** High request rate (>5 req/sec) and high error rate (>5%)
- **Top Clients Table:** Ranked by request volume
- **cost-share Detail Panel:** Special focus on known offender

---

## Dashboard Panels

### 1. Overview: Total RTE Request Rate
- Single stat showing total requests/sec to RTE
- Color-coded thresholds: Green (<5), Yellow (5-10), Red (>10)

### 2. RATE: Requests per Second by Client Service
- Time series graph showing request rate for all 20+ client services
- Services tracked:
  - cost-share ⚠️ (known offender)
  - medication ⚠️ (previous incident)
  - athena-bridge, family, member-android-app, member-ios-app
  - memberfeedback-server, mx-ui-workflows, newcoproxy-server
  - oneapp-migration, practitioner-availability, practitioner-server
  - provider-match-server, px-careflow, quactl
  - revenue-cycle-management, service-request, wizard
  - dedicated-care-team-server, digitalagent-server

### 3. ERRORS: Error Rate by Client Service (%)
- Percentage of requests returning 4xx/5xx errors
- Thresholds: Green (<1%), Yellow (1-5%), Orange (5-10%), Red (>10%)

### 4. ERRORS: Error Count by Client Service
- Absolute error counts by service and status code
- Helps identify which services are failing

### 5-7. DURATION: P50, P95, P99 Latency
- Three panels showing latency percentiles
- P95/P99 have thresholds: Green (<1s), Yellow (1-5s), Orange (5-10s), Red (>10s)
- Critical for identifying slow clients

### 8. RTE Service: Request Rate
- Direct view of RTE service request rate
- Uses `stedi_request_duration_count` metric

### 9. RTE Service: P95 Latency
- RTE service latency (critical bottleneck)
- Thresholds aligned with RTE capacity (15 connections, P95 >10s)

### 10. Top 10 Clients by Request Volume
- Table showing highest volume clients in last hour
- Helps identify aberrant clients quickly

### 11. cost-share Service Detail ⚠️
- Special panel for known offender
- Shows request rate, error rate, and P95 latency
- Critical for monitoring DP-5538 mitigation

### 12-13. Alert Panels
- High Request Rate Alert: Shows services exceeding 5 req/sec
- High Error Rate Alert: Shows services with >5% error rate

---

## Metrics Used

### Prometheus Metrics

1. **traces_spanmetrics_calls_total**
   - Labels: `service`, `span_name`, `status_code`
   - Used for: Rate and error metrics

2. **traces_spanmetrics_latency_bucket**
   - Labels: `service`, `span_name`, `le`
   - Used for: Duration metrics (histogram quantiles)

3. **stedi_request_duration_count**
   - Labels: `service_name`
   - Used for: RTE service direct metrics

4. **stedi_request_duration_bucket**
   - Labels: `service_name`, `le`
   - Used for: RTE service latency

### Span Name Filters

- `HTTP POST.*coverage.*` - HTTP calls to coverage service
- `.*coverage.*` - Any coverage-related spans
- `.*rte.*|.*RTE.*` - RTE-related spans

---

## Import Instructions

### Option 1: Grafana Web UI (Recommended)

1. **Open Grafana:**
   ```
   https://grafana.includedhealth.com
   ```

2. **Navigate to Import:**
   - Click "+" → "Import"
   - Or: Dashboards → Import

3. **Upload Dashboard:**
   - Click "Upload JSON file"
   - Select: `/tmp/rte-paths-dashboard.json`

4. **Configure:**
   - **Folder:** Select "coverage-server" folder
   - **Name:** "RTE Paths - All Clients (RED Metrics)"
   - **UID:** Auto-generated (or set custom)

5. **Import:**
   - Click "Import"
   - Dashboard will be created and opened

### Option 2: Grafana API

```bash
# Set your Grafana API key
export GRAFANA_API_KEY="your-api-key-here"
export GRAFANA_URL="https://grafana.includedhealth.com"

# Create dashboard
curl -X POST \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $GRAFANA_API_KEY" \
  -d @/tmp/rte-paths-dashboard.json \
  "$GRAFANA_URL/api/dashboards/db"
```

### Option 3: Terraform/Infrastructure as Code

If your Grafana dashboards are managed via Terraform, add:

```hcl
resource "grafana_dashboard" "rte_paths" {
  folder    = grafana_folder.coverage_server.id
  config_json = file("${path.module}/rte-paths-dashboard.json")
}
```

---

## Dashboard Configuration

### Time Range
- **Default:** Last 6 hours
- **Refresh:** 30 seconds (auto-refresh)

### Variables
- **client_service:** Multi-select variable for filtering by service
  - Options: All client services calling coverage
  - Default: All services

### Annotations
- **Deployments:** Shows deployment markers
- Tags: `deploy`

### Links
- **DP-5538 Investigation:** Links to JIRA ticket

---

## Use Cases

### 1. Identify Offending Clients
- Check "Top 10 Clients" table
- Look for services with high request rates (>5 req/sec)
- Compare to baseline (normal: <1 req/sec)

### 2. Monitor DP-5538 Mitigation
- Watch "cost-share Service Detail" panel
- Verify request rate drops below 1 req/sec
- Confirm no spikes at :50 past hour

### 3. Detect New Incidents
- Set up alerts for:
  - High request rate (>5 req/sec)
  - High error rate (>5%)
  - High latency (P95 >10s)

### 4. Capacity Planning
- Track total RTE request rate
- Monitor RTE service latency
- Identify when approaching 15 connection limit

---

## Alerting Recommendations

### Critical Alerts

1. **RTE Request Rate Spike**
   ```promql
   sum(rate(stedi_request_duration_count[5m])) > 10
   ```
   - Threshold: >10 req/sec
   - Action: Page on-call

2. **Client Request Rate Spike**
   ```promql
   sum(rate(traces_spanmetrics_calls_total{
     service=~"cost-share|medication",
     span_name=~"HTTP POST.*coverage.*"
   }[5m])) > 5
   ```
   - Threshold: >5 req/sec for batch services
   - Action: Notify service owners

3. **High Error Rate**
   ```promql
   100 * sum(rate(traces_spanmetrics_calls_total{
     span_name=~"HTTP POST.*coverage.*",
     status_code=~"5.."
   }[5m])) / sum(rate(traces_spanmetrics_calls_total{
     span_name=~"HTTP POST.*coverage.*"
   }[5m])) > 5
   ```
   - Threshold: >5% error rate
   - Action: Investigate failures

---

## Troubleshooting

### No Data Showing

1. **Check metric availability:**
   ```promql
   count(traces_spanmetrics_calls_total{span_name=~".*coverage.*"})
   ```

2. **Verify service names match:**
   - Check actual service names in Prometheus
   - Update regex filters if needed

3. **Check time range:**
   - Ensure metrics exist for selected time range
   - Try "Last 1 hour" if "Last 6 hours" shows nothing

### Incorrect Service Names

If services aren't showing up:
1. Query Prometheus to find actual service names:
   ```promql
   label_values(traces_spanmetrics_calls_total{span_name=~".*coverage.*"}, service)
   ```
2. Update dashboard JSON with correct service names
3. Re-import dashboard

### Missing Metrics

If `traces_spanmetrics_*` metrics aren't available:
- Check if OpenTelemetry collector is configured
- Verify span metrics are being exported
- May need to use alternative metrics (e.g., `http_requests_total`)

---

## Dashboard Maintenance

### Regular Updates

- **Weekly:** Review top clients table for new services
- **Monthly:** Update service list if new clients added
- **After Incidents:** Add new services to monitoring

### Adding New Services

1. Identify new service calling coverage → RTE
2. Add to service regex in dashboard JSON:
   ```json
   "service=~\"cost-share|medication|NEW_SERVICE|...\""
   ```
3. Re-import dashboard

---

## Related Documentation

- **DP-5538 Investigation:** https://includedhealth.atlassian.net/browse/DP-5538
- **RTE Paths Report:** `/tmp/DP-5538-RTE-Paths-Report.md`
- **Grafana Evidence:** `/tmp/DP-5538-Grafana-Evidence.md`

---

## Files

- **Dashboard JSON:** `/tmp/rte-paths-dashboard.json`
- **Import Script:** `/tmp/create-rte-dashboard.sh`
- **This README:** `/tmp/RTE-Paths-Dashboard-README.md`

---

**Created by:** TJ Singleton
**Date:** November 20, 2025
**For:** DP-5538 Investigation & Ongoing Monitoring
