# Stedi API Latency Measurements

**Research Date**: 2025-01-27
**Researcher**: AI Agent
**Status**: ⚠️ Needs Current Metrics
**Source**: Extracted from `docs/plans/event-driven/EVENT_DRIVEN_RTE_PLAN.md` and `realtime-eligibility/docs/traffic-control.md`

## Summary

Latency measurements for Stedi API calls showing P95 > 10s, P99 > 30s, with maximum observed latency of 120+ seconds. These measurements are critical for understanding timeout cascades and system performance issues.

## Methodology

- **Source**: `realtime-eligibility/docs/traffic-control.md`
- **Analysis Method**: Production metrics analysis
- **Time Period**: Historical measurements (needs current verification)

## Findings

### Finding 1: Latency Percentiles

- **P95 Latency**: >10 seconds
- **P99 Latency**: >30 seconds
- **Max Observed**: 120+ seconds (Stedi internal retries)

**Source**: `realtime-eligibility/docs/traffic-control.md`

### Finding 2: Impact on System

**Timeout Mismatch**:
- Client timeout (10-90s) < Stedi timeout (120s) → frequent failures
- Frontend timeout (10-30s) before RTE completes
- Coverage-server polling timeout (90s) vs. Stedi max (120s+)

**System Constraints**:
- **Stedi Constraint**: 15 concurrent connection limit (global)
- **Client Services**: 90+ services depend on RTE
- **Failure Mode**: Timeout cascade causes system-wide outages

### Finding 3: Stedi Internal Retries

**Max Observed**: 120+ seconds
**Cause**: Stedi's own internal retry window
**Impact**: Requires long timeout values (>120 seconds) for Temporal workflow activities

**Documentation Reference**:
```
"The timeout for the specific HTTP call to Stedi must be long
(e.g., > 120 seconds). This is to accommodate Stedi's own
internal retry window"
```

## Validation

### Production Metrics Validation
- ⚠️ **CRITICAL**: Verify current P95/P99 latency from Prometheus/Grafana
- ⚠️ **CRITICAL**: Verify current max observed latency
- ⚠️ **Needs Update**: Check if latency has improved or worsened since research date

### Documentation Verification
- ✅ Verified: `realtime-eligibility/docs/traffic-control.md` exists
- ⚠️ Needs Check: Verify current documentation matches measurements

## Citations

- **Related Documentation**: `realtime-eligibility/docs/traffic-control.md`
- **Related Research**:
  - `../timeouts/rte-timeout-cascade.md` (timeout chain analysis)
  - `../request-chains/rte-dependency-chains.md` (dependency analysis)

## Notes

- **CRITICAL**: These measurements need current verification from production metrics
- Latency measurements are historical and may have changed
- P95/P99 measurements are critical for timeout configuration
- Max observed latency (120s+) impacts Temporal workflow timeout settings
