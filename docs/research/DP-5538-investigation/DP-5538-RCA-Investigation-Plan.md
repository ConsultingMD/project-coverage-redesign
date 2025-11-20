# DP-5538: RCA Investigation Plan
## Internal DOS - RTE Service 10x Traffic Spike

**JIRA:** [DP-5538](https://includedhealth.atlassian.net/browse/DP-5538)
**Status:** Backlog - Unprioritized
**Reporter:** Created from Slack message
**Date:** November 19, 2025
**Investigator:** TJ Singleton

---

## Plan: Deep investigation, RCA, and fix options

This document outlines the investigation plan for DP-5538, following the structured RCA process.

---

## 1. Clarify the Problem and Impact

### Problem Statement
The realtime-eligibility (RTE) service experienced a **10x traffic spike**, indicating an internal Denial of Service (DOS) scenario.

### Symptoms Observed
- **10x increase** in requests to RTE service
- **Traffic pattern:** Spikes occurring at **10 minutes before the hour** (e.g., :50 past each hour)
- Observed on RTE ConnectRPC calls
- Pattern suggests scheduled job (cron) behavior

### Environments Affected
- **Production** (confirmed via Grafana dashboards)
- Potentially UAT (to be verified)

### Impact Assessment
- **Severity:** Potential P1 if sustained
  - RTE service has only **15 concurrent connections** to Stedi
  - P95 latency > 10 seconds
  - Can cause cascading failures to 90+ dependent services

- **User Impact:**
  - Delayed eligibility checks
  - Potential coverage lookup failures
  - Timeout errors on frontend applications

- **Business Impact:**
  - Degraded member experience
  - Potential inability to verify insurance coverage
  - Risk of SEV-1 incident (based on historical data: 6 SEV-1, 2 SEV-2 incidents in 14 months)

### Acceptance Criteria
- [ ] Identify offending client causing 10x traffic spike
- [ ] Implement mitigation to prevent recurrence
- [ ] Add monitoring/alerting to detect similar patterns early
- [ ] Document findings in JIRA ticket
- [ ] Create runbook for similar incidents

---

## 2. Expand Context (Logs, Rollbar, Grafana)

### Rollbar Investigation

**Actions:**
- [ ] Search for RTE-related errors during incident timeframe (Nov 19, 2025)
- [ ] Filter by:
  - Service: `realtime-eligibility`
  - Error patterns: timeouts, connection pool exhaustion, 429 errors
  - Time window: Focus on :50 minute marks
- [ ] Check for:
  - Timeout errors on RTE service
  - Coverage-server timeout errors (downstream impact)
  - Errors from suspected clients (cost-share, medication)

**Expected findings:**
- Increased timeout errors during spike windows
- Connection pool exhaustion warnings
- Cache miss patterns

### Grafana Dashboard Analysis

**Primary Dashboards:**
1. **RTE Service Dashboard**
   - Request rate metrics (check for 10x spike)
   - P95/P99 latency during incident
   - Connection pool utilization
   - Cache hit/miss ratios

2. **GraphQL Subgraph by Client Dashboard** ⭐ **KEY DASHBOARD**
   - Request breakdown by internal client
   - Identify which client shows 10x increase
   - Filter for coverage-server → realtime-eligibility calls

3. **Coverage-Server Dashboard**
   - Total request volume to RTE
   - Request amplification ratio
   - Timeout rate to RTE

**Metrics to examine:**
- `rte_request_total` (labeled by client)
- `rte_request_duration_seconds` (P95, P99)
- `stedi_connection_pool_active`
- `rte_cache_hit_ratio`
- `rte_invalid_member_id_rejected_total`
- `coverage_server_rte_proxy_requests_total` (by client)

**Actions:**
- [ ] Export metrics for incident timeframe (Nov 19, hourly :50 marks)
- [ ] Create time-series graph showing request rate by client
- [ ] Correlate spike times with client request patterns
- [ ] Check for cache hit rate drops during spikes

### Logs Analysis (Loki)

**Query patterns:**
```
{service="realtime-eligibility"} |= "high volume" | json
{service="coverage-server"} |= "rteProxy" | json | client_name != ""
{service="cost-share"} |= "eligibility" or "coverage"
```

**Actions:**
- [ ] Search RTE logs for rate limiting warnings
- [ ] Check coverage-server logs for client identity during spikes
- [ ] Review cost-share service logs at :50 timestamps
- [ ] Look for error patterns: connection refused, timeout, circuit breaker opened

### Recent Changes & Deployments

**Actions:**
- [ ] Check deploy markers around Nov 19 for:
  - cost-share service
  - medication service
  - coverage-server
  - realtime-eligibility
- [ ] Review feature flag changes (LaunchDarkly)
- [ ] Check for new cron jobs or scheduled tasks
- [ ] Review Kafka consumer offset changes
- [ ] Check configuration changes (timeout values, rate limits)

### Related Incidents Cross-Reference

**INC-882 (Nov 2, 2025) - Similar RTE DOS:**
- **Root cause:** Medication service Kafka consumer defaulted to earliest offset
- **Pattern:** Massive backlog processing caused RTE flood
- **Resolution:** Paused consumer, fixed offset configuration
- **Lessons:** Check Kafka consumer configurations

**ACT-2795 (Nov 10, 2025) - RTE Downtime Spike:**
- Coverage failures for RTE-based users
- Increased request volume
- Pattern: Sudden spike, not gradual

**INC-821 (July 16, 2025) - Stone Service Timeout:**
- Traffic surge overwhelmed service
- Auto-scaling couldn't keep up
- Cascading failures

---

## 3. Code and Dependency Review

### Primary Services Involved

#### Suspect #1: cost-share Service
**Why suspect:**
- Mentioned in Slack thread as potential offender
- Calls coverage-server → RTE
- Cron pattern matches (:50 past hour)

**Review checklist:**
- [ ] Check for scheduled jobs (cron, Kubernetes CronJobs)
- [ ] Review recent PRs to cost-share
- [ ] Examine eligibility check logic
- [ ] Look for batch processing code
- [ ] Check Kafka consumer configurations
- [ ] Review retry logic and backoff strategies
- [ ] Check for request caching

**File locations:**
- `/Users/tj.singleton/src/github.com/ConsultingMD/cost-share/`
- Look for: cron configs, scheduled task definitions, coverage/eligibility clients

#### Suspect #2: medication Service
**Why suspect:**
- Previous incident (INC-882) with similar pattern
- Known to make RTE calls
- Kafka consumer history of issues

**Review checklist:**
- [ ] Verify Kafka consumer offset settings (should not be "earliest")
- [ ] Check for consumer lag/backlog
- [ ] Review eligibility check triggers
- [ ] Examine retry/error handling

#### coverage-server (Gateway)
**Review checklist:**
- [ ] Check request amplification logic
- [ ] Review caching strategy (is cache being bypassed?)
- [ ] Examine rate limiting configuration (likely absent)
- [ ] Check request collapsing (singleflight) implementation
- [ ] Review GraphQL resolver logic for RteProxy and WarmRTECache

**Key files:**
- GraphQL resolvers for RTE operations
- Gateway/proxy logic to RTE service
- Cache implementation

#### realtime-eligibility Service
**Review checklist:**
- [ ] Connection pool configuration (should limit to 15)
- [ ] Circuit breaker settings
- [ ] Temporal workflow configuration
- [ ] Cache TTL settings
- [ ] Request queuing/scheduling logic

### Recent Code Changes

**Search for relevant PRs:**
```bash
# cost-share recent changes
gh pr list --repo ConsultingMD/cost-share --state merged --limit 20

# medication recent changes
gh pr list --repo ConsultingMD/medication --state merged --limit 20

# Check for cron/scheduled job additions
git log --all --grep="cron" --grep="schedule" --since="2025-11-01"
```

### Common Patterns to Look For

1. **Unhandled Edge Cases:**
   - Missing cache checks
   - Null member ID handling
   - Invalid payer ID handling

2. **Race Conditions:**
   - Concurrent requests for same member
   - Cache stampede scenarios
   - Lock contention

3. **Timeout/Retry Issues:**
   - Aggressive retry logic
   - Missing exponential backoff
   - No circuit breaker

4. **Data Shape Changes:**
   - New required fields triggering re-fetch
   - Cache invalidation logic changes
   - Schema migration impacts

---

## 4. Form Hypotheses

### Hypothesis 1: cost-share Scheduled Job Flooding RTE
**Evidence supporting:**
- Traffic spikes at :50 past the hour (cron pattern)
- cost-share mentioned in Slack thread
- cost-share calls coverage → RTE

**Evidence against:**
- No direct log evidence yet (pending Grafana analysis)
- Could be coincidental timing

**How to test:**
- Check cost-share cron job definitions
- Correlate spike timing with cost-share job execution
- Review Grafana client breakdown

**Likelihood:** ⭐⭐⭐⭐ (4/5) - HIGH

---

### Hypothesis 2: Kafka Consumer Replay (Similar to INC-882)
**Evidence supporting:**
- Similar pattern to INC-882 (medication service)
- Kafka consumer issues can cause massive backlogs
- Sudden 10x spike matches replay scenario

**Evidence against:**
- Would likely be more sustained, not hourly spikes
- Pattern is too regular (hourly)

**How to test:**
- Check Kafka consumer lag for all RTE clients
- Review consumer group offsets
- Look for consumer rebalancing events

**Likelihood:** ⭐⭐ (2/5) - LOW to MEDIUM

---

### Hypothesis 3: WarmRTECache Misuse/Abuse
**Evidence supporting:**
- WarmRTECache mutation could be called in batches
- If called without proper rate limiting, could flood RTE
- Could be part of a warming strategy gone wrong

**Evidence against:**
- Would need to identify which service is calling it
- Timing pattern suggests automated trigger

**How to test:**
- Check GraphQL operation metrics for WarmRTECache usage
- Identify which clients call WarmRTECache
- Review cache warming strategies

**Likelihood:** ⭐⭐⭐ (3/5) - MEDIUM

---

### Hypothesis 4: Missing Request Collapsing
**Evidence supporting:**
- Coverage-server has in-memory singleflight (25x amplification potential)
- Multiple pods means duplicate requests to RTE
- Could amplify a small upstream spike into 10x downstream

**Evidence against:**
- This would be an ongoing issue, not hourly
- Would show up as high amplification ratio

**How to test:**
- Check request amplification ratio (upstream → downstream)
- Review singleflight effectiveness metrics
- Compare request counts at coverage-server vs RTE

**Likelihood:** ⭐⭐ (2/5) - Contributing factor, not root cause

---

### Hypothesis 5: New Client or Feature Launch
**Evidence supporting:**
- Sudden traffic increase could indicate new feature
- New client might not have rate limiting
- Could be A/B test or rollout

**Evidence against:**
- Hourly pattern doesn't match user-driven traffic
- Would expect more gradual ramp

**How to test:**
- Review recent feature launches
- Check LaunchDarkly flag changes
- Review new service deployments

**Likelihood:** ⭐⭐ (2/5) - LOW

---

## 5. Root Cause Analysis (5 Whys)

**Most likely hypothesis:** cost-share Scheduled Job Flooding RTE

### 5 Whys Analysis

**1. Why did RTE receive a 10x traffic spike?**
- Because a client service sent 10x more requests than normal.

**2. Why did the client service send 10x more requests?**
- Because a scheduled job (cron) processed a large batch of records and made an eligibility check for each one without rate limiting.

**3. Why didn't the scheduled job have rate limiting?**
- Because there are no enforced rate limits on coverage-server's GraphQL gateway, and clients are expected to self-regulate (which is not reliable).

**4. Why are there no rate limits on coverage-server?**
- Because the current architecture assumes all internal clients are well-behaved, and rate limiting was not implemented during the initial design (ACT-2366 tracking this improvement).

**5. Why was the batch job size not validated before processing?**
- Because there's no monitoring or alerting on request rates from individual clients, so the team didn't know the batch size would cause issues.

### Contributing Root Causes

#### Code/Design Issues
1. **No rate limiting** on coverage-server GraphQL gateway
2. **No request collapsing** across pods (in-memory singleflight only)
3. **No backpressure mechanism** - system relies on timeouts
4. **No priority metadata** - can't distinguish batch jobs from frontend traffic
5. **Missing circuit breakers** on client services

#### Data/Configuration Issues
1. **Batch job size** not validated against RTE capacity
2. **Caching strategy** may not be optimized (check if cache was bypassed)
3. **Kafka consumer offsets** (if applicable) may be misconfigured

#### Operational Practices
1. **No client request rate monitoring** - can't identify aberrant clients proactively
2. **No runbook** for RTE DOS scenarios
3. **Insufficient alerting** - spike not caught until manual observation
4. **Missing capacity planning** for batch jobs hitting RTE

#### Process/Requirements
1. **No documented SLA** for RTE usage by internal clients
2. **No required review** of batch job impact on shared services
3. **Missing traffic control architecture** (ACT-2366 proposal pending)

---

## 6. Propose Fix Options

### Option 1: Immediate Mitigation - Throttle Offending Client ⭐ RECOMMENDED FOR IMMEDIATE ACTION

**Description:**
Identify the offending client (likely cost-share) and temporarily throttle or pause the scheduled job.

**Implementation steps:**
1. Confirm client identity via Grafana dashboard
2. Contact cost-share service owners
3. Options:
   - Pause cron job temporarily
   - Reduce batch size
   - Add delay between requests
   - Spread job execution across hour instead of single burst

**Risks:**
- May delay cost-share business functionality
- Doesn't prevent future occurrences

**Tradeoffs:**
- ✅ Fast to implement (hours)
- ✅ Minimal code changes
- ✅ Immediately reduces RTE load
- ❌ Doesn't address root cause
- ❌ Requires manual coordination

**Effort:** S (Small) - Few hours

---

### Option 2: Add Client Rate Limiting on coverage-server ⭐ RECOMMENDED FOR SHORT-TERM

**Description:**
Implement per-client rate limiting on coverage-server's GraphQL gateway for RTE operations.

**Implementation steps:**
1. Add rate limiting middleware to coverage-server GraphQL handler
2. Configure limits per client (based on client identifier)
3. Return 429 (Too Many Requests) when limit exceeded
4. Add metrics for rate limit hits
5. Add alert for rate limit violations

**Code changes:**
```go
// coverage-server GraphQL middleware
type RateLimiter struct {
    limiters map[string]*rate.Limiter
}

func (r *RateLimiter) CheckLimit(ctx context.Context, clientID string) error {
    limiter := r.getLimiter(clientID)
    if !limiter.Allow() {
        return fmt.Errorf("rate limit exceeded for client %s", clientID)
    }
    return nil
}
```

**Configuration:**
```yaml
rate_limits:
  rte_operations:
    default: 10/second
    batch_clients:  # Lower limit for known batch services
      cost-share: 2/second
      medication: 2/second
```

**Risks:**
- May cause 429 errors for legitimate traffic
- Requires tuning to find right limits
- Could break existing batch jobs

**Tradeoffs:**
- ✅ Prevents future incidents
- ✅ Protects RTE from abuse
- ✅ Relatively simple to implement
- ❌ Requires testing and tuning
- ❌ May need per-client customization

**Effort:** M (Medium) - 1-2 weeks

---

### Option 3: Implement Request Collapsing Across Pods

**Description:**
Replace in-memory singleflight with distributed request collapsing using Redis.

**Implementation steps:**
1. Add Redis-backed request collapsing in coverage-server
2. Use distributed lock with fingerprint key
3. Share cache keys across all coverage-server pods
4. Reduce amplification from 25x to near 1x

**Code changes:**
- Replace `singleflight.Group` with Redis-based coordination
- Use request fingerprint (member ID + payer + date) as lock key
- TTL-based locks (30s) to prevent deadlocks

**Risks:**
- Redis dependency
- Potential increased latency (Redis round-trip)
- Lock contention issues

**Tradeoffs:**
- ✅ Significantly reduces RTE load
- ✅ Benefits all clients, not just offender
- ❌ More complex implementation
- ❌ Adds Redis dependency
- ❌ Requires thorough testing

**Effort:** L (Large) - 3-4 weeks

---

### Option 4: Implement Full Traffic Control Architecture

**Description:**
Implement the "Three Gates" architecture from Coverage Redesign spec (ACT-2366).

**Components:**
1. **Gate 1:** Per-member request collapsing (coverage-server, member-sponsorship)
2. **Gate 2:** RTE response cache (realtime-eligibility)
3. **Gate 3:** Stedi concurrency scheduler (15-slot resource manager)

**Implementation steps:**
1. Deploy distributed request collapsing (Gate 1)
2. Enhance RTE cache with DynamoDB backing (Gate 2)
3. Implement Stedi scheduler with fairness and priority (Gate 3)
4. Add priority metadata propagation
5. Implement per-payer fairness queuing

**Risks:**
- Large architectural change
- Requires significant testing
- Potential performance impact
- Complex rollout

**Tradeoffs:**
- ✅ Comprehensive solution
- ✅ Addresses all known issues (P1-P5)
- ✅ Prevents future incidents
- ❌ Very large effort
- ❌ Months-long timeline
- ❌ High complexity

**Effort:** XL (Extra Large) - 3-6 months

**Note:** This is already planned under ACT-2366. DP-5538 adds urgency.

---

## 7. Mitigations and Rollout Plan

### Short-Term Mitigations (Implement Immediately)

#### 1. Identify and Throttle Offending Client
**Timeline:** Today (Nov 20, 2025)

**Steps:**
- [ ] Check Grafana GraphQL Subgraph by Client dashboard
- [ ] Filter for Nov 19 hourly :50 spikes
- [ ] Confirm client identity (expected: cost-share)
- [ ] Contact service owners via Slack
- [ ] Request immediate mitigation:
  - Pause cron job, OR
  - Reduce batch size by 10x, OR
  - Add 100ms delay between requests

#### 2. Add Monitoring and Alerting
**Timeline:** This week

**Steps:**
- [ ] Create alert: "RTE request rate > 100/sec"
- [ ] Create alert: "Per-client RTE rate > 20/sec"
- [ ] Add dashboard: "RTE Request Rate by Client (Last 24h)"
- [ ] Add dashboard panel: "RTE Request Rate Pattern (Hourly)"

#### 3. Document in Runbook
**Timeline:** This week

**Steps:**
- [ ] Create/update runbook: "RTE DOS Investigation"
- [ ] Add steps to identify offending client
- [ ] Add common mitigation strategies
- [ ] Link to relevant Grafana dashboards
- [ ] Add escalation contacts

### Medium-Term Fix (Option 2: Rate Limiting)
**Timeline:** Next sprint (1-2 weeks)

#### Implementation Plan

**Phase 1: Design (Days 1-2)**
- [ ] Define rate limit values per client type
- [ ] Design rate limiter implementation (token bucket vs leaky bucket)
- [ ] Choose storage backend (Redis vs in-memory)
- [ ] Define 429 response format
- [ ] Create metrics and alerts

**Phase 2: Implementation (Days 3-7)**
- [ ] Implement rate limiter middleware
- [ ] Add configuration system
- [ ] Add metrics collection
- [ ] Add unit tests
- [ ] Add integration tests

**Phase 3: Testing (Days 8-10)**
- [ ] Test with synthetic traffic
- [ ] Verify 429 responses
- [ ] Test with real batch client (cost-share in UAT)
- [ ] Load test to ensure no performance regression
- [ ] Verify metrics and alerts

**Phase 4: Rollout (Days 11-14)**
- [ ] Deploy to UAT with conservative limits
- [ ] Monitor for 2 days
- [ ] Gradually tighten limits
- [ ] Deploy to production (one pod first)
- [ ] Roll out to all pods
- [ ] Monitor for 1 week

#### Tests to Add/Update

**Unit Tests:**
- [ ] Test rate limiter allows requests under limit
- [ ] Test rate limiter blocks requests over limit
- [ ] Test rate limiter resets after time window
- [ ] Test per-client limit independence

**Integration Tests:**
- [ ] Test GraphQL RteProxy with rate limit
- [ ] Test WarmRTECache with rate limit
- [ ] Test 429 response format
- [ ] Test metrics emission

**E2E Tests:**
- [ ] Test batch client stays within limits
- [ ] Test frontend client not affected
- [ ] Test multiple clients don't interfere

#### Rollbar Expectations
- [ ] Zero rate limit errors for frontend clients
- [ ] Expected 429 errors for batch clients hitting limits
- [ ] No increase in timeout errors
- [ ] No increase in RTE connection pool errors

#### Grafana Expectations
- [ ] RTE request rate stays under 50/sec (down from 10x spike)
- [ ] Per-client request rates within configured limits
- [ ] No increase in P95/P99 latency
- [ ] Rate limit hit counter shows batch client throttling
- [ ] Cache hit rate improves (less pressure on RTE)

### Long-Term Architecture (Option 4)
**Timeline:** Next quarter (Q1 2026)

**Reference:** ACT-2366 - Traffic Control Architecture

**Phases:**
1. Q1 2026: Gate 1 (distributed request collapsing)
2. Q1 2026: Gate 2 (enhanced caching)
3. Q2 2026: Gate 3 (scheduler with fairness)
4. Q2 2026: Priority metadata propagation

---

## 8. Document and Share

### Final RCA Summary
**To be completed after investigation**

Will include:
- Root cause: [Pending confirmation - likely cost-share cron job]
- Contributing factors: [No rate limiting, no request collapsing, etc.]
- Timeline of events
- Impact assessment
- Resolution steps
- Preventive measures

### Recommended Fix
**Immediate:** Option 1 (Throttle offending client)
**Short-term:** Option 2 (Rate limiting)
**Long-term:** Option 4 (Full traffic control - already planned)

### Verification Steps
1. Monitor RTE request rate returns to baseline (<50/sec)
2. No new spikes at hourly :50 marks
3. Rate limiting prevents similar incidents
4. Alerts catch future anomalies

### Update Locations
- [ ] This JIRA ticket (DP-5538)
- [ ] Related JIRA (ACT-2366 - Traffic Control)
- [ ] Runbook: "RTE DOS Investigation"
- [ ] Team docs: "RTE Usage Best Practices"
- [ ] Slack: Update #eng-data-platform with findings

---

## Next Steps Checklist

### Today (Nov 20, 2025)
- [x] Gather context from JIRA, Glean, Slack
- [x] Run surveyor analysis for RTE paths
- [x] Document RCA investigation plan
- [ ] Check Grafana to identify offending client
- [ ] Contact client service owners
- [ ] Implement immediate throttling

### This Week
- [ ] Complete Rollbar/Grafana analysis
- [ ] Confirm root cause
- [ ] Post RCA summary to JIRA
- [ ] Add monitoring and alerts
- [ ] Update runbook
- [ ] Kick off Option 2 (rate limiting) implementation

### Next Sprint
- [ ] Implement and test rate limiting
- [ ] Roll out to production
- [ ] Monitor effectiveness
- [ ] Update ACT-2366 priority if needed

---

## Investigation Progress Tracking

**Started:** November 20, 2025
**Status:** Investigation phase
**Next Update:** After Grafana analysis

**Key Questions Still Outstanding:**
1. Which client is causing the spike? (Hypothesis: cost-share)
2. What is the batch job doing?
3. Why did it start on Nov 19?
4. Is there a data volume change driving this?

---

## References

- **JIRA:** [DP-5538](https://includedhealth.atlassian.net/browse/DP-5538)
- **Slack Thread:** https://ih-epdd.slack.com/archives/C0908UCMFRQ/p1763588636399249
- **Related Incidents:**
  - [INC-882](https://docs.google.com/document/d/1Q9PleayTFJCoi66TBCg4L7NzOCrDPCHwncc_zTPPIBU) - RTE DOS (Nov 2, 2025)
  - [ACT-2795](https://includedhealth.atlassian.net/browse/ACT-2795) - RTE Downtime Spike (Nov 10, 2025)
- **Architecture Specs:**
  - Coverage Redesign - Traffic Control (ACT-2366)
  - RTE_UPSTREAM_DEPENDENCIES.md
  - EVENT_DRIVEN_RTE_SUMMARY.md
- **Analysis Reports:**
  - [DP-5538-RTE-Paths-Report.md](/tmp/DP-5538-RTE-Paths-Report.md) - Surveyor analysis

---

**Document Version:** 1.0
**Last Updated:** November 20, 2025
**Author:** TJ Singleton
