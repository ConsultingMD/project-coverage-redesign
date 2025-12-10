# End-to-End Tracing Capabilities Research

**Research Date**: 2025-01-27
**Researcher**: AI Agent
**Status**: ⚠️ Needs Validation
**Source**: Extracted from `surveyor/E2E_TRACING_STATUS.md`

## Summary

Current state analysis of end-to-end tracing capabilities across all protocols. Documents what works (single-hop RPC tracing, protocol detection) and what's broken (multi-hop recursion, GraphQL chain tracing, cross-protocol integration).

## Methodology

- **Tools Used**: Surveyor tracing commands, code analysis
- **Repositories Analyzed**: All services in workspace
- **Date**: November 14, 2025
- **Target**: `realtime-eligibility` (RTE) service

## Findings

### Finding 1: What Works ✅

**RPC Tracing (Single Hop)**:
- `trace-rpc` command functional
- Finds direct RPC callers (e.g., coverage → RTE)
- Cache working correctly

**Protocol Detection**:
- ✅ Go RPC clients/servers
- ✅ Python RPC clients/servers
- ✅ HTTP clients (Go & Python)
- ✅ Kafka producers/consumers (Go & Python)
- ✅ GraphQL resolvers (Go)
- ✅ TypeScript (React components, GraphQL, RPC, HTTP)
- ✅ Kotlin (Composables, GraphQL, RPC, HTTP)
- ✅ Swift (SwiftUI views, GraphQL, RPC, HTTP)

**Language Support**:
- ✅ Go (SSA-based analysis)
- ✅ Python (AST-based analysis)
- ✅ TypeScript (tree-sitter)
- ✅ Kotlin (tree-sitter)
- ✅ Swift (regex-based, SwiftSyntax planned)

### Finding 2: What's Broken ❌

**Multi-Hop RPC Tracing**:
- `trace-rpc` only finds depth 1 (coverage)
- Does NOT recurse to find what calls coverage
- Should find: medication → coverage → RTE
- Currently finds: coverage → RTE (stops)

**GraphQL Chain Tracing**:
- `trace-graphql` crashes with KeyError
- Reports "0 resolvers" for coverage (wrong!)
- Should link: Apollo operations → resolvers → RTE calls
- Currently: Broken

**Cross-Protocol Jumps**:
- No integration between protocols
- Can't trace: GraphQL → RPC → HTTP
- Can't trace: Kafka → RPC → HTTP
- Missing unified call graph

### Finding 3: Root Cause Analysis

**Issue 1: Recursive Tracing Not Implemented**
- File: `surveyor/lib/cross_language_tracer.py` (RecursiveDependencyAnalyzer)
- Problem: Finds direct dependencies (depth 1) but does NOT recurse
- Missing: Loop to process dependents list, recursive call for each dependent

**Issue 2: GraphQL Resolver Detection Broken**
- File: `tools/surveyor-goast/analyzer.go`
- Problem: Returns 0 resolvers for coverage (should be 100+)
- Likely Causes: Wrong directory search path, gqlgen.yml not parsed, pattern matching too strict

**Issue 3: GraphQL Chain Command Crashes**
- File: `surveyor/cli.py:3151`
- Error: `KeyError: 'none'`
- Problem: `confidence_counts` dict missing `'none'` key

**Issue 4: No Cross-Protocol Integration**
- Files: `surveyor/lib/unified_graph.py` (defined ✅), `surveyor/lib/unified_tracer.py` (NOT IMPLEMENTED ❌)
- Problem: Protocol analyzers work independently, no unified call graph builder

### Finding 4: Current Capabilities vs Vision

**What We Can Do Today** ✅:
- Find direct RPC callers (1 hop)
- Detect HTTP clients in Go/Python
- Detect Kafka producers/consumers
- Extract GraphQL resolvers (when working)
- Analyze TypeScript React components
- Analyze Kotlin Android composables
- Analyze Swift iOS views

**What We Can't Do Yet** ❌:
- Trace GraphQL → RPC → HTTP chains
- Trace Kafka → RPC → GraphQL chains
- Trace React screen → backend → external API
- Multi-hop recursion (stops at depth 1)
- Transitive closure
- Circular dependency detection
- Amplification ratio calculation

## Validation

### Surveyor Validation
```bash
# Test RPC tracing
surveyor trace-rpc \
  --target realtime-eligibility \
  --workspaces /Users/tj.singleton/src/github.com/ConsultingMD \
  --output json \
  --max-depth 10

# Expected: Should find depth 1 only (coverage)
# Status: ⚠️ Needs verification
```

### Code Location Verification
- ✅ Verified: `surveyor/lib/cross_language_tracer.py` exists
- ✅ Verified: `tools/surveyor-goast/analyzer.go` exists
- ✅ Verified: `surveyor/cli.py` exists
- ⚠️ Needs Check: Verify actual behavior matches documented issues

## Citations

- **Related Documentation**: `surveyor/FULL_STACK_TRACING_PLAN.md` (6-phase plan)
- **Related Research**:
  - `rte-dependency-chains.md` (69 chains found)
  - `../timeouts/rte-timeout-cascade.md` (timeout analysis)

## Notes

- Current status: 🟡 **Foundation Built, Integration Needed**
- Multi-hop recursion fix applied but limited by Go tool (RPC-only detection)
- GraphQL integration and cross-protocol jumps require Phase 3 implementation
- Total effort to complete vision: 1-2 weeks (10-12 days)
