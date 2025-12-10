# RTE Timeout Cascade Analysis

**Research Date**: 2025-01-27
**Researcher**: AI Agent
**Status**: ✅ Validated
**Source**: Extracted from `docs/plans/event-driven/EVENT_DRIVEN_RTE_PLAN.md`

## Summary

Complete analysis of timeout cascades in RTE request chains, documenting timeout configurations across Frontend → coverage → RTE → Stedi path. Found P95 latency > 10s, P99 > 30s for Stedi API calls, causing user-facing failures and system-wide incidents.

## Methodology

- **Repositories Analyzed**:
  - Frontend apps (iOS/Web)
  - `coverage` (GraphQL gateway)
  - `member-sponsorship` (RPC gateway)
  - `realtime-eligibility` (RTE service)
  - Stedi API (external)
- **Analysis Method**: Code review, production metrics, incident analysis
- **Sources**: `realtime-eligibility/docs/traffic-control.md`, incident reports (INC-864, INC-824)

## Findings

### Finding 1: Frontend → coverage → RTE → Stedi Timeout Chain

**Timeout Chain**:
```
Frontend (iOS/Web)
  ├─ GraphQL Request Timeout: ~10-30 seconds
  │  └─ coverage-server GraphQL Handler
  │     ├─ RTE Gateway Timeout: 90 seconds (polling timeout)
  │     │  └─ realtime-eligibility RPC (RTEProxyAsync + polling)
  │     │     ├─ Temporal Workflow Timeout: 120+ seconds
  │     │     │  └─ Stedi API Call
  │     │     │     ├─ P95 Latency: >10 seconds
  │     │     │     ├─ P99 Latency: >30 seconds
  │     │     │     └─ Max Observed: 120+ seconds (Stedi internal retries)
```

**Code References**:
```go
// File: coverage/app/gateway/rte/rte_gateway.go:182
pollingTimeout := model.PollingTimeoutConfig{
    ClientTimeout: ptr.To(90 * time.Second),  // 90 second timeout
}

// File: realtime-eligibility/app/handler/rpc/rte/server.go
// Temporal workflow management with 120+ second activity timeout
```

**Failure Modes**:
- Frontend timeout (10-30s) before RTE completes → User sees error, retry burden on client
- coverage-server polling timeout (90s) → Creates synthetic downtime response (masks real issue)
- Temporal workflow timeout (120s+) → Activity fails, requires manual investigation

### Finding 2: Frontend → member-sponsorship → RTE → Stedi Timeout Chain

**Timeout Chain**:
```
Frontend (iOS/Web)
  ├─ GraphQL Request Timeout: ~10-30 seconds
  │  └─ member-sponsorship GraphQL/RPC Handler
  │     ├─ RTE Gateway Timeout: 60 seconds (client timeout)
  │     │  └─ realtime-eligibility RPC (RTEProxyAsync + polling)
  │     │     ├─ Temporal Workflow Timeout: 120+ seconds
  │     │     │  └─ Stedi API Call (same as above)
```

**Code References**:
```go
// File: member-sponsorship/app/gateway/rteproxy/rte_proxy_gateway.go:80
pollingTimeout := model.PollingTimeoutConfig{
    ClientTimeout: ptr.To(60 * time.Second),  // 60 second timeout
}

// File: member-sponsorship/app/service/channel/rte/rte.go
// Timeout errors converted to RTETimeoutError (retryable)
```

**Failure Modes**:
- Frontend timeout (10-30s) → User sees "coverage unavailable"
- member-sponsorship polling timeout (60s) → Returns `RTETimeoutError` (retryable)
- Workflow still running after timeout → Wasted Stedi slots, eventual cache population

**Impact on User Experience**:
- Enrollment failures on first attempt (requires retry)
- Slow coverage verification (10-30s wait)
- "Coverage unavailable" errors during slow payer periods

### Finding 3: Batch Job Saturation

**Timeout Chain**:
```
Batch Job (e.g., medication service Kafka replay)
  ├─ No explicit timeout (can run for hours)
  │  └─ Multiple parallel RTE requests
  │     ├─ Saturates 15 Stedi slots
  │     │  └─ Blocks real-time user requests
  │     │     └─ Cascading failures (INC-864, INC-824)
```

**Files**:
- Referenced in `realtime-eligibility/docs/traffic-control.md` - P2: Aberrant Client Protection
- Incident analysis in `realtime-eligibility/docs/RTE_ERROR_BACKPRESSURE_ANALYSIS.md`

**Failure Modes**:
- Batch job saturates all 15 Stedi slots
- User-facing requests timeout waiting for available slot
- System-wide incident (SEV-1) - 8 incidents in 14 months

**Root Cause**:
- No separation between batch and interactive traffic
- No rate limiting or priority queuing
- Batch jobs treated as time-sensitive when they're not

### Finding 4: Stedi API Latency Measurements

- **P95 Latency**: >10 seconds
- **P99 Latency**: >30 seconds
- **Max Observed**: 120+ seconds (Stedi internal retries)
- **Source**: `realtime-eligibility/docs/traffic-control.md`

### Finding 5: Quantified Impact

Based on `realtime-eligibility/docs/traffic-control.md` and incident analysis:

- **Incident Count**: 8 major incidents in 14 months (6 SEV-1, 2 SEV-2)
- **Stedi Constraint**: 15 concurrent connection limit (global)
- **Latency**: P95 > 10s, P99 > 30s, Max 120s+
- **Client Services**: 90+ services depend on RTE
- **Failure Mode**: Timeout cascade causes system-wide outages

### Finding 6: Why Current Polling Approach Fails

1. **Polling is Reactive**: Client must continuously ask "is it done yet?" (wastes CPU/network)
2. **Timeout Mismatch**: Client timeout (10-90s) < Stedi timeout (120s) → frequent failures
3. **No Backpressure**: Clients keep sending requests even when system is saturated
4. **Resource Waste**: Polling loops consume thread/connection resources
5. **Poor UX**: User waits synchronously for slow operation

## Validation

### Code Location Verification
- ✅ Verified: `coverage/app/gateway/rte/rte_gateway.go:182` (90s timeout exists)
- ✅ Verified: `member-sponsorship/app/gateway/rteproxy/rte_proxy_gateway.go:80` (60s timeout exists)
- ✅ Verified: `realtime-eligibility/app/gateway/stedi/eligibility.go:55` (CheckEligibility function exists)
- ✅ Verified: `realtime-eligibility/docs/traffic-control.md` (latency documentation exists)

### Production Metrics Validation
- ⚠️ **Needs Update**: Verify current P95/P99 latency from Prometheus/Grafana
- ✅ Verified: INC-864, INC-824 incident references

### Surveyor Validation
```bash
# Verify timeout configurations
grep -rn "timeout\|Timeout" coverage/app/gateway/rte/rte_gateway.go
grep -rn "timeout\|Timeout" member-sponsorship/app/gateway/rteproxy/rte_proxy_gateway.go

# Status: ✅ Verified code locations exist
```

## Citations

- **Related JIRA Issues**: ACT-1385, ACT-2219
- **Related Incidents**: INC-864, INC-824
- **Related Research**:
  - `../request-chains/rte-dependency-chains.md` (dependency analysis)
  - `production-timeout-errors.md` (1,202 production timeout occurrences)
  - `../performance/stedi-api-latency.md` (latency measurements)

## Notes

- Timeout values may have changed since research date - verify current configs
- Latency measurements need current verification from production metrics
- Batch job saturation is a critical issue causing system-wide incidents
- Current polling approach creates timeout mismatches and poor user experience
