# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a **research and analysis repository** for the Coverage Redesign project at IncludedHealth. It contains:

1. **Data extraction scripts** (Node.js/Python) for analyzing Apollo GraphQL operations and traffic patterns
2. **Traffic control simulation planning** for the Real-Time Eligibility (RTE) service
3. **Architecture analysis** of the RTE service request chains and dependencies
4. **Client profiling tools** for understanding API usage patterns

This is NOT a production codebase - it's a collection of analysis tools, data files, and design documents.

## Repository Architecture

### Meta-Project Workspace Structure

This repository is part of a **multi-repository workspace** managed through `project.yaml`:

- References 11 related repositories (coverage, realtime-eligibility, member-sponsorship, etc.)
- Uses VS Code/Cursor workspace configuration (`coverage-redesign.code-workspace`)
- **No symlinks** - repositories are accessed via relative paths in workspace folders
- See [AGENTS.md](AGENTS.md) for workspace management details

### Key Repository Groups

1. **Production Services** (referenced, not in this repo):
   - `coverage` - GraphQL gateway serving 59+ client services
   - `realtime-eligibility` - RTE service managing 15 Stedi connections
   - `member-sponsorship` - Alternative RTE gateway
   - `support-tools` - Internal support tools

2. **This Repository** - Analysis tools organized by purpose:
   - Apollo Studio data extraction (JavaScript)
   - Prometheus/Grafana metrics analysis (Python)
   - Client profiling and traffic analysis (Python)
   - Design documents and specifications (Markdown)

## Common Commands

### Data Extraction & Analysis

**GraphQL Operation Analysis:**
```bash
# Fetch Apollo Studio operation details (requires APOLLO_KEY)
export APOLLO_KEY='your-key'
./fetch-apollo-operation-details.js --all

# Match operations to persisted query manifest
node match-by-field-signature.js

# Generate client operation statistics
node operations-by-client-stats.js --improved
```

**Metrics Analysis (Python):**
```bash
# Analyze GraphQL operations by month
python graphql_operations_monthly.py

# Generate client profiles from metrics
python client_profile.py

# Inspect Prometheus metrics
python inspect_metrics.py

# Analyze trading partner histograms
python trading_partner_histogram_weekly.py
```

**Client Profiling:**
```bash
# Generate client profile for specific service
python client_profile.py --service realtime_eligibility

# Generate profiles for all services
python client_profile.py --all-services
```

### No Build/Test Commands

This repository contains only scripts - there is **no build process, no test suite, no deployment**. Scripts are run directly via `node` or `python`.

## Project Context & Architecture

### The Real-Time Eligibility (RTE) Problem

This project addresses traffic control challenges for the RTE service:

**System Constraints:**
- 15 concurrent connection limit to Stedi eligibility service
- 90+ services depend on RTE through coverage-server gateway
- P95 latency > 10 seconds for Stedi calls
- 8 major incidents in 14 months (6 SEV-1, 2 SEV-2)

**Core Problems (P1-P5):**
1. **P1: Request Collapsing** - In-memory singleflight limited to single pod (25x amplification)
2. **P2: Aberrant Client Protection** - No rate limiting; single client can saturate all 15 connections
3. **P3: Payer Fairness** - Slow payers monopolize connections, starving others
4. **P4: Backpressure** - No admission control; system relies on timeouts
5. **P5: Priority Metadata** - Cannot distinguish frontend from batch traffic

**Architecture Layers ("Three Gates"):**
- **Gate 1**: Per-member request collapsing (coverage-server, member-sponsorship)
- **Gate 2**: RTE response cache (realtime-eligibility)
- **Gate 3**: Stedi concurrency scheduler (15-slot resource manager)

See [spec.md](spec.md) for complete architecture specification.

### Key Design Documents

**Read these first for context:**
- [spec.md](spec.md) - Complete traffic control architecture specification
- [prd.md](prd.md) - Scheduler Simulation Framework requirements
- [phase1-data-plan.md](phase1-data-plan.md) - Data collection strategy for simulation
- [SOLUTION_SUMMARY.md](SOLUTION_SUMMARY.md) - Apollo operation matching solution
- [RTE_ANALYSIS_INDEX.md](RTE_ANALYSIS_INDEX.md) - Complete RTE dependency analysis index

**RTE Analysis Suite (4 comprehensive documents):**
1. [RTE_REQUEST_CHAIN_ANALYSIS.md](RTE_REQUEST_CHAIN_ANALYSIS.md) - 7 request chains from clients to Stedi
2. [RTE_UPSTREAM_DEPENDENCIES.md](RTE_UPSTREAM_DEPENDENCIES.md) - Complete backward trace of 59 services
3. [RTE_ERROR_BACKPRESSURE_ANALYSIS.md](RTE_ERROR_BACKPRESSURE_ANALYSIS.md) - Failure scenarios & user experience
4. [RTE_BACKPRESSURE_CURRENT_STATE_ANALYSIS.md](RTE_BACKPRESSURE_CURRENT_STATE_ANALYSIS.md) - First principles analysis

## Data Flow Architecture

### Apollo Studio → Persisted Query Matching Pipeline

**Problem Solved:** Apollo Studio operation IDs use different hashing than persisted query manifests, requiring field signature matching.

**Pipeline Flow:**
```
1. fetch-apollo-operation-details.js
   ↓ (apollo-operation-details.json)
2. match-by-field-signature.js
   ↓ (apollo-field-signature-matching.json)
3. integrate-improved-matching.js
   ↓ (coverage-clients-analysis-improved.json)
4. operations-by-client-stats.js --improved
   ↓ (coverage-operations-by-client.{json,csv})
```

**Key Files:**
- `apollo-operation-details.json` - Raw Apollo Studio API responses (794 operations)
- `apollo-field-signature-matching.json` - Field signature match results (49.7% exact, 14.4% ambiguous)
- `coverage-operations-by-client.json` - Final analysis with improved match data

### Grafana/Prometheus Metrics Pipeline

**Phase 1 Data Collection** (for simulation framework):

```
Prometheus Metrics → payer_profile_generator.py → payer_definitions.yaml
                  ↘ workload_profile_generator.py → client_definitions.yaml
                  ↘ incident_analyzer.py → timeline_events.yaml

Loki Logs → log_analyzer.py → member_distribution.yaml (Zipf parameters)
```

**Key Metrics:**
- `stedi_request_duration` - Latency histogram by trading_partner_id
- `stedi_http_status` - HTTP status codes by trading_partner_id
- `stedi_errors` - DTE error codes (80, 42)
- `traces_spanmetrics_calls_total` - GraphQL operation call counts

See [phase1-data-plan.md](phase1-data-plan.md) for complete metric collection strategy.

## Script Organization

### JavaScript Scripts (Apollo Studio Analysis)

**Data Extraction:**
- `fetch-apollo-operation-details.js` - Queries Apollo Studio API for operation signatures
- `identify-coverage-clients.js` - Identifies clients hitting coverage subgraph
- `extract-*.js` - Various extraction utilities for manifest/query data

**Matching & Analysis:**
- `match-by-field-signature.js` - Matches Apollo ops to manifest by field signatures
- `integrate-improved-matching.js` - Updates analysis with improved matches
- `operations-by-client-stats.js` - Generates statistics with `--improved` flag

**Query Plan Analysis:**
- `generate-query-plans.js` - Generates query plans for operations
- `normalize-query-plan-cache.js` - Normalizes query plan cache files
- `generate-dependency-graph.js` - Creates subgraph dependency graphs

### Python Scripts (Metrics & Profiling)

**Metrics Analysis:**
- `graphql_operations_monthly.py` - Analyzes GraphQL operations from Prometheus
- `inspect_metrics.py` - Inspects Prometheus metrics for analysis
- `trading_partner_histogram_weekly.py` - Analyzes trading partner request patterns
- `trading_partner_profile.py` - Generates trading partner profiles

**Client Profiling:**
- `client_profile.py` - Main client profiling script (generates YAML definitions)
  - Supports `--service <name>` for specific service
  - Supports `--all-services` for batch processing
  - Outputs to `client_definitions_<service>.yaml` and `client_profiles_<service>.csv`

## Key Patterns & Conventions

### Apollo Studio Authentication

Scripts use environment variable `APOLLO_KEY` or auto-detect from rover config:
```bash
export APOLLO_KEY='your-api-key'
export APOLLO_GRAPH_REF='application-layer-graph-kwpy1@production'  # optional
```

### Field Signature Matching

Apollo Studio canonicalizes field order in operation signatures, so matching by **field set** (ignoring order) works better than exact string matching:
- Extract fields from both Apollo operations and manifest queries
- Compare as sets (order-independent)
- Classify as: exact match, ambiguous match (multiple variants), or unmatched

### Distribution Modeling (Phase 1 Data Plan)

**Payer Latency:**
- Default: Log-normal distribution (per PRD spec)
- Alternative: Pareto distribution for long-tail payers (P95 > 10s)
- Parameters: P50, P90, P95, P99 from `stedi_request_duration` histogram

**Member Distribution:**
- Zipf distribution (models "popular member" collapsing problem)
- Extracted from Loki logs (account_id frequency analysis)
- Parameters: s (skew), N (pool size)

**Arrival Rate:**
- Poisson distribution (random independent arrivals)
- Parameter: meanRequestsPerSecond from `traces_spanmetrics_calls_total`

### YAML Output Format

Client definitions follow simulation.yaml schema from [prd.md](prd.md):
```yaml
clientDefinitions:
  - name: "Frontend-Sponsorship"
    priorityTier: "gold"      # gold/silver/bronze
    requestType: "sponsorship"

payerDefinitions:
  - name: "Payer-Fast"
    latency:
      type: "log-normal"
      mean_ms: 800
      stddev_ms: 300
    errorRates:
      http_5xx: 0.001
      dte_80: 0.0
      dte_42: 0.0
```

## Important Context

### This is NOT Production Code

- Scripts are exploratory and analysis-focused
- Output files (JSON/CSV/YAML) are checked in for reference
- No CI/CD, no deployment, no production dependencies
- Scripts may require manual configuration (API keys, environment variables)

### Data Sources

**External APIs:**
- Apollo Studio GraphQL API (requires authentication)
- Grafana/Prometheus (production metrics)
- Loki (production logs)

**Static Files:**
- `persisted-query-manifest-*.json` - Persisted query definitions from Apollo Studio
- `application-layer-graph-*.json` - GraphQL schema from Apollo Studio
- `*.json`, `*.csv` - Extracted data and analysis results

### Multi-Repository Context

This repository references production services defined in separate repositories:
- Use workspace folders to navigate to service code
- Service implementations are in `coverage/`, `realtime-eligibility/`, etc.
- Platform metadata is in `platform-map/tng_descriptors/`

When analyzing service behavior, cross-reference:
1. This repo's analysis documents (RTE_*.md)
2. Service source code (via workspace folders)
3. Platform-map descriptors for GraphQL operations

## Troubleshooting

### Apollo Studio Scripts Fail

Check `APOLLO_KEY` environment variable:
```bash
# Verify key is set
echo $APOLLO_KEY

# Or let script auto-detect from rover config
# (script looks for ~/.rover/config.toml)
```

### Prometheus Queries Timeout

Queries in `phase1-data-plan.md` use 30-day windows. For large datasets:
- Reduce time window (e.g., `[7d]` instead of `[30d]`)
- Use sampling (e.g., 1-hour windows across 30 days)
- Query cold storage for historical data beyond hot retention

### Member Distribution Analysis

`member_id`/`account_id` NOT available in Prometheus metrics (too high cardinality). Use:
1. **Primary**: Loki logs via LogQL queries
2. **Secondary**: Distributed traces (7-day hot storage, 30+ day cold storage)
3. **Historical**: Potluck data warehouse via Lasik SQL client

See [phase1-data-plan.md](phase1-data-plan.md) Section 2.3 for complete strategy.

## Related Tickets

**Current Work:**
- ACT-2366: Improve Traffic Control (rate limiting, priority queuing)
- CLOUD-6296: GraphQL Inbound Request Graphs by Client (caller visibility)
- CLOUD-6297: Kafka Replay Protection (prevent INC-864 recurrence)
- PHARM-1038: Update Consumer Offset Handling (medication service)

**Incident References:**
- INC-864 (Oct 2025): Medication Kafka replay saturated Stedi connections
- INC-824 (July 2025): Unknown client flooding coverage-server
- INC-821 (July 2025): Traffic surge outage
- See [RTE_ANALYSIS_INDEX.md](RTE_ANALYSIS_INDEX.md) for complete incident history (8 incidents)
