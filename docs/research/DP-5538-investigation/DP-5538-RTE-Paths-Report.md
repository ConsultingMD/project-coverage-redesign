# DP-5538: RTE Traffic Spike - Comprehensive Path Analysis

**Generated:** November 20, 2025
**Issue:** [DP-5538](https://includedhealth.atlassian.net/browse/DP-5538) - Internal DOS: requests to RTE service jumped 10x
**Analysis Tool:** Surveyor (TNG Descriptor & Dependency Analysis)

---

## Executive Summary

**Problem:** The realtime-eligibility (RTE) service experienced a 10x traffic spike, indicating an internal DOS scenario.

**Key Findings:**
- **35 unique dependency chains** lead to RTE
- **22+ client services** can trigger RTE requests through coverage-server
- **Traffic pattern:** Spikes observed at 10 minutes before the hour (cron job pattern)
- **Primary suspect:** cost-share service (based on Slack thread analysis)
- **Similar incident:** INC-882 (Nov 2, 2025) - medication service Kafka consumer caused RTE DOS

---

## All Paths to RTE Service

### Overview
- **Total unique chains to RTE:** 35
- **Maximum chain length:** 7 hops
- **Primary gateway:** coverage-server (GraphQL subgraph)
- **Final destination:** insurance service (via ConnectRPC)

### Path Breakdown by Hop Count

#### Direct RTE Callers (2 hops)
```
realtime-eligibility → insurance
```

#### 3-Hop Paths (1 path)
```
coverage-server → realtime-eligibility → insurance
```

#### 4-Hop Paths (19 paths)
**Client → coverage-server → realtime-eligibility → insurance**

Direct coverage-server clients:
1. `athena-bridge` → coverage-server → RTE
2. `cost-share` → coverage-server → RTE ⚠️ **SUSPECT**
3. `dedicated-care-team-server` → coverage-server → RTE
4. `digitalagent-server` → coverage-server → RTE
5. `family` → coverage-server → RTE
6. `medication` → coverage-server → RTE
7. `member-android-app` → coverage-server → RTE
8. `member-ios-app` → coverage-server → RTE
9. `memberfeedback-server` → coverage-server → RTE
10. `mx-ui-workflows` → coverage-server → RTE
11. `newcoproxy-server` → coverage-server → RTE
12. `oneapp-migration` → coverage-server → RTE
13. `practitioner-availability` → coverage-server → RTE
14. `practitioner-server` → coverage-server → RTE
15. `provider-match-server` → coverage-server → RTE
16. `px-careflow` → coverage-server → RTE
17. `quactl` → coverage-server → RTE
18. `revenue-cycle-management` → coverage-server → RTE
19. `service-request` → coverage-server → RTE
20. `wizard` → coverage-server → RTE

#### 5-Hop Paths (9 paths)
```
1. authzilla → mx-ui-workflows → coverage-server → realtime-eligibility → insurance
2. careapi-server → service-request → coverage-server → realtime-eligibility → insurance
3. dedicated-care-team-server → practitioner-server → coverage-server → realtime-eligibility → insurance
4. intake → newcoproxy-server → coverage-server → realtime-eligibility → insurance
5. member-ios-app → family → coverage-server → realtime-eligibility → insurance
6. memberfeedback-server → service-request → coverage-server → realtime-eligibility → insurance
7. mx-ui-workflows → service-request → coverage-server → realtime-eligibility → insurance
8. oneapp-migration → family → coverage-server → realtime-eligibility → insurance
9. wizard → service-request → coverage-server → realtime-eligibility → insurance
```

#### 6-Hop Paths (3 paths)
```
1. authzilla → mx-ui-workflows → service-request → coverage-server → realtime-eligibility → insurance
2. member-android-app → authzilla → mx-ui-workflows → coverage-server → realtime-eligibility → insurance
3. member-android-app → careapi-server → service-request → coverage-server → realtime-eligibility → insurance
```

#### 7-Hop Paths (1 path)
```
member-android-app → authzilla → mx-ui-workflows → service-request → coverage-server → realtime-eligibility → insurance
```

---

## GraphQL Operations to RTE

Coverage-server exposes 2 GraphQL operations that call RTE:

### 1. RteProxy (Query)
```graphql
query RteProxy($request: RteProxyInput!) {
    rteProxy(request: $request) {
        rawResponseData
        responseStatus
        responseStatusCode
        error
        wasCached
    }
}
```
**File:** `graphql_operations/coverage-server/RTEProxy.graphql`
**Signature:** `024bb7383608e85a23ee2ad7e3ce0a26bcb1fc86072e412e3df1e6603bb342cf`

### 2. WarmRTECache (Mutation)
```graphql
mutation WarmRTECache($request: RteProxyInput!) {
    warmRteCache(request: $request)
}
```
**File:** `graphql_operations/coverage-server/WarmRTECache.graphql`
**Signature:** `35cff6f5873a09834be39d62776f31b7f5421a7d2a9623cf500d0b8ca42df11b`

---

## Client Service Categories

### Frontend/Member-Facing Services
- `member-android-app` (7 paths)
- `member-ios-app` (2 paths)
- `mx-ui-workflows` (4 paths)
- `authzilla` (3 paths)

### Backend Services
- `coverage-server` ⭐ **PRIMARY GATEWAY** (543 total chains)
- `service-request` (5 paths)
- `family` (2 paths)
- `careapi-server` (2 paths)

### Practitioner/Provider Services
- `practitioner-server` (2 paths)
- `practitioner-availability` (1 path)
- `provider-match-server` (1 path)
- `dedicated-care-team-server` (2 paths)

### Financial/Claims Services
- `cost-share` ⚠️ **SUSPECT** (1 path)
- `revenue-cycle-management` (1 path)

### Integration/Bridge Services
- `athena-bridge` (1 path)
- `newcoproxy-server` (2 paths)
- `intake` (1 path)

### Medication Services
- `medication` (1 path) - ⚠️ Previous incident: INC-882

### Other Services
- `digitalagent-server` (1 path)
- `memberfeedback-server` (2 paths)
- `oneapp-migration` (2 paths)
- `px-careflow` (1 path)
- `quactl` (1 path)
- `wizard` (2 paths)

---

## Related Incidents

### INC-882: RTE DOS (Nov 2, 2025)
**Root Cause:** Medication service Kafka consumer defaulted to earliest offset, causing massive request backlog to RTE.

**Symptoms:**
- High volume of RTE requests
- Timeout issues on coverage-server
- Memory pressure on RTE server

**Resolution:**
- Paused offending Kafka consumer
- Restarted RTE server
- Fixed consumer offset configuration

### ACT-2795 / TSE-4137: Sudden RTE Downtime Spike (Nov 10, 2025)
**Symptoms:**
- Large sudden spike of coverage failures for RTE-based users
- Error messages: "issues retrieving eligibility details"
- Increased request volume

### Current Incident: DP-5538 (Nov 19, 2025)
**Symptoms:**
- 10x traffic spike to RTE
- Traffic spikes at 10 minutes before the hour (cron pattern)

**Suspected Client:** cost-share service
- Based on Slack thread analysis
- Pattern suggests scheduled job (cron)

---

## Monitoring & Observability

### Grafana Dashboards
1. **Service Dashboards:** Located in service dashboard folder
2. **GraphQL Subgraph by Client Dashboard:** Helps identify which client is causing volume spikes
3. **RTE-specific metrics:**
   - `rte_invalid_member_id_rejected_total`
   - `rte_invalid_placeholder_value_total`

### Prometheus Counters
- RTE request counters with client labels
- Request timing histograms
- Error rate metrics

### Event-Driven RTE Metrics
Document: `EVENT_DRIVEN_RTE_SUMMARY.md`
- Request submission metrics
- Queue addition tracking
- Slot assignment monitoring
- Completion metrics

### Identifying Offending Clients
**Methods:**
1. Client profile analysis - break down request rate by internal client
2. GraphQL subgraph dashboard - monitor request rates per client
3. Observability tools - identify volume spikes
4. Rate limiting/traffic control - prevent aberrant client behavior

---

## Recommendations

### Immediate Actions
1. **Identify offending client:**
   - Check Grafana GraphQL Subgraph by Client dashboard
   - Filter for traffic spikes at :50 past the hour
   - Correlate with cost-share service deployment/cron schedule

2. **Review cost-share service:**
   - Check for new cron jobs or scheduled tasks
   - Review recent deployments
   - Examine Kafka consumers and offset configurations
   - Check for batch processing jobs

3. **Implement rate limiting:**
   - Add per-client rate limits on coverage-server GraphQL gateway
   - Implement admission control for RTE requests
   - Add backpressure mechanisms

### Long-Term Improvements
1. **Traffic Control Architecture:**
   - Implement "Three Gates" pattern (from Coverage Redesign spec)
   - Add request collapsing at coverage-server level
   - Implement per-payer fairness scheduling
   - Add priority metadata for frontend vs. batch traffic

2. **Monitoring Enhancements:**
   - Add per-client request rate alerts
   - Implement anomaly detection for RTE traffic patterns
   - Create runbook for RTE DOS incidents
   - Add dashboards showing request amplification ratios

3. **Client Requirements:**
   - Document RTE usage best practices
   - Require rate limiting on all RTE clients
   - Mandate caching strategies for batch jobs
   - Add circuit breakers for RTE dependencies

---

## Investigation Checklist

- [ ] Check Grafana dashboards for client identification
- [ ] Review cost-share service logs at incident time
- [ ] Check for cost-share cron jobs running at :50 past hour
- [ ] Review recent cost-share deployments
- [ ] Check Kafka consumer offsets for cost-share
- [ ] Review RTE cache hit rates during spike
- [ ] Analyze request patterns (member IDs, payers)
- [ ] Check for duplicate/redundant requests
- [ ] Review coverage-server request amplification
- [ ] Examine if WarmRTECache was misused

---

## Data Sources

- **Surveyor Analysis:** TNG descriptor dependency graphs
- **RTE Chains File:** `tng_descriptors/rte_chains_with_queries.json`
- **Coverage Chains:** `coverage_callers.json` (543 total chains)
- **Glean Search:** JIRA tickets, incident reports, Slack threads
- **Related Documents:**
  - `RTE_UPSTREAM_DEPENDENCIES.md`
  - `Coverage Redesign - Traffic Control & Scheduling`
  - `EVENT_DRIVEN_RTE_SUMMARY.md`
  - INC-882 incident report

---

## Next Steps

1. **Immediate:** Identify and throttle offending client
2. **Short-term:** Implement basic rate limiting
3. **Medium-term:** Deploy traffic control improvements
4. **Long-term:** Implement full "Three Gates" architecture

---

**Report Generated by:** Surveyor CLI
**Analysis Date:** November 20, 2025
**For:** DP-5538 Investigation
