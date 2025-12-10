# Research Documentation Index

This directory contains **validated research findings** from system analysis. These documents are citable and can be shared with team members working on related issues.

**⚠️ Important**: These documents contain **research findings**, not plans or proposals. For aspirational plans, see `../plans/`.

## Research Categories

### Request Chains
Analysis of request flows across services and protocols.

- **[RTE Dependency Chains](request-chains/rte-dependency-chains.md)** ✅ Validated
  - 69 dependency chains for RTE service (longest: 7 hops)
  - Validated with surveyor `chains-to-service` command

- **[Member ID Flow Trace](request-chains/member-id-flow-trace.md)** ✅ Validated
  - ACT-2861: Backwards trace from Stedi Gateway through 8 layers
  - Code locations verified

- **[E2E Tracing Capabilities](request-chains/e2e-tracing-capabilities.md)** ⚠️ Needs Validation
  - Current state of multi-hop RPC tracing
  - GraphQL chain tracing status

### Timeouts
Analysis of timeout issues and cascades.

- **[RTE Timeout Cascade](timeouts/rte-timeout-cascade.md)** ✅ Validated
  - Complete timeout chain: Frontend → coverage → RTE → Stedi
  - Code locations verified

- **[Production Timeout Errors](timeouts/production-timeout-errors.md)** ✅ Validated
  - 1,202 timeout occurrences in production
  - Root cause analysis (Five-Whys)

- **[Coverage Rules Timeouts](timeouts/coverage-rules-timeouts.md)** ⚠️ Needs Validation
  - GetCoverageRules timeout analysis (ACT-1385)
  - Database query performance issues

### Performance
Performance measurements and analysis.

- **[Stedi API Latency](performance/stedi-api-latency.md)** ⚠️ Needs Current Metrics
  - P95 > 10s, P99 > 30s measurements
  - Needs verification from current production metrics

## Validation Status

| Document | Code Locations | Surveyor | Metrics | Status |
|---------|---------------|----------|---------|--------|
| RTE Dependency Chains | ✅ | ✅ | N/A | ✅ Validated |
| Member ID Flow Trace | ✅ | ⚠️ | N/A | ✅ Validated |
| E2E Tracing Capabilities | ✅ | ⚠️ | N/A | ⚠️ Partial |
| RTE Timeout Cascade | ✅ | ✅ | ⚠️ | ⚠️ Partial |
| Production Timeout Errors | ✅ | N/A | ⚠️ | ⚠️ Partial |
| Coverage Rules Timeouts | ⚠️ | N/A | ⚠️ | ⚠️ Needs Validation |
| Stedi API Latency | N/A | N/A | ⚠️ | ⚠️ Needs Update |

## How to Cite Research

When referencing research findings in plans, PRs, or discussions:

```markdown
Based on research findings in [RTE Timeout Cascade Analysis](timeouts/rte-timeout-cascade.md),
we found that timeout cascades occur across Frontend → coverage → RTE → Stedi chain,
with P95 latency > 10s for Stedi API calls.
```

## Contributing Research

When adding new research:

1. Use the research document template (see [RESEARCH_DOCUMENTATION_PROPOSAL.md](../RESEARCH_DOCUMENTATION_PROPOSAL.md))
2. Include code references with file paths and line numbers
3. Run validation steps (code location check, surveyor, metrics)
4. Update this index
5. Link related research documents

## Related Documentation

- **Plans**: `../plans/` - Aspirational plans and proposals
- **Investigations**: `../investigations/` - Issue-specific investigations (if created)
- **Architecture**: `../architecture/` - System architecture documentation (if created)

## Key Research Findings Summary

**Request Chain Analysis:**
- **69 dependency chains** identified for RTE service (longest: 7 hops)
- Member ID flow traced through 8 layers (ACT-2861)
- Multi-hop tracing partially working (depth 1 only for RPC)

**Timeout Analysis:**
- **1,202 timeout occurrences** in production (member-sponsorship)
- **P95 latency > 10s, P99 > 30s** for Stedi API calls
- **Timeout cascades** across Frontend → coverage → RTE → Stedi chain
- **Batch job saturation** causing system-wide incidents (8 incidents in 14 months)

**Performance Analysis:**
- Stedi API latency: P95 > 10s, P99 > 30s, Max 120s+
- Coverage rules queries: Peak latency 4s, frequent timeouts
- Database query failures: "Prepared statement contains too many placeholders" (Error 1390)

## Validation Commands

### Code Location Verification
```bash
# Verify timeout configurations
grep -rn "timeout\|Timeout" coverage/app/gateway/rte/rte_gateway.go
grep -rn "timeout\|Timeout" member-sponsorship/app/gateway/rteproxy/rte_proxy_gateway.go

# Verify member ID flow
grep -rn "VerificationFieldsToRTESafeMap" member-sponsorship/
```

### Surveyor Validation
```bash
# Verify RTE dependency chains
surveyor chains-to-service \
  --service realtime-eligibility \
  --dependency-graph tng_descriptors/dependency_graph.json \
  --collapse

# Verify RPC tracing
surveyor trace-rpc \
  --target realtime-eligibility \
  --workspaces $IH_HOME
```

### Production Metrics Validation
```bash
# Check Prometheus/Grafana for latency metrics
# Verify: P95/P99 latency for Stedi API calls

# Check Rollbar for timeout errors
# Verify: Current timeout error counts
```
