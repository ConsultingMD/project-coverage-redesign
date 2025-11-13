# Event-Driven RTE & Digital Session Platform

## Overview

This directory contains the complete technical specification and planning documents for replacing long-running RTE (Real-Time Eligibility) requests with an event-driven architecture using Kafka and WebSocket Gateway.

**Total Documentation**: 15,500+ lines across 9 documents

---

## Documents

### 📖 Navigation & Index

**[EVENT_DRIVEN_INDEX.md](./EVENT_DRIVEN_INDEX.md)** (1,378 lines)
- **Purpose**: Navigation guide for all documentation
- **Contents**: 
  - Quick start guides by role (Backend, Frontend, Platform, Product)
  - How documents relate to each other
  - FAQ section
  - Pusher Channels replacement strategy
  - Faye/Bayeux protocol patterns summary
- **Start here**: Best entry point for new readers

---

### 🎯 Core Technical Plans

**[EVENT_DRIVEN_RTE_PLAN.md](./EVENT_DRIVEN_RTE_PLAN.md)** (2,073 lines)
- **Purpose**: Comprehensive technical specification for event-driven RTE
- **Contents**:
  - Timeout cascade analysis (59+ services impacted)
  - Event-driven architecture (Kafka + WebSocket Gateway)
  - Application-specific use cases across all services
  - 28-week implementation roadmap
  - Infrastructure requirements
  - Cost analysis
- **Audience**: Backend engineers, architects, platform teams

**[EVENT_DRIVEN_RTE_SUMMARY.md](./EVENT_DRIVEN_RTE_SUMMARY.md)** (732 lines)
- **Purpose**: Executive summary and business case
- **Contents**:
  - Problem quantification (8 incidents in 14 months)
  - Solution overview
  - Timeline and costs ($140-180k)
  - Before/after metrics
  - Comparison: Agent Platform Streaming vs. Event-Driven RTE
  - Pusher Channels replacement strategy
- **Audience**: Product managers, engineering leadership, executives

---

### 🌐 Platform Extensions

**[DIGITAL_SESSION_PLATFORM_PLAN.md](./DIGITAL_SESSION_PLATFORM_PLAN.md)** (1,630 lines)
- **Purpose**: Extends event-driven RTE to a general-purpose Digital Session Platform
- **Contents**:
  - Digital Session concept (member sessions, encounters)
  - Event-driven platform for all frontends (MX app, iOS, Android, Care app)
  - Integration with Digital Twin and CareFlow
  - Frontend SDKs (TypeScript, Swift, Kotlin)
  - Use cases beyond RTE
- **Audience**: Frontend engineers, mobile teams, product managers

---

### 🔍 Research & Analysis

**[PUSHER_RESEARCH_FINDINGS.md](./PUSHER_RESEARCH_FINDINGS.md)** (258 lines)
- **Purpose**: Glean research findings on Pusher usage at IncludedHealth
- **Contents**:
  - Distinction between Pusher Beams (already migrated to FCM) and Pusher Channels (active)
  - INC-801 incident analysis (0.17% delivery rate)
  - iOS team reliability concerns
  - Cost analysis ($1.2-3.6k/year savings)
  - Current usage patterns (video visits)
- **Audience**: Platform teams, infrastructure engineers

**[FAYE_BAYEUX_WEBSOCKET_DESIGN.md](./FAYE_BAYEUX_WEBSOCKET_DESIGN.md)** (1,124 lines)
- **Purpose**: Analysis of Faye/Bayeux protocol patterns for WebSocket Gateway design
- **Contents**:
  - Bayeux protocol architecture
  - IncludedHealth's existing fallback patterns (iOS, member-sponsorship, agent-platform)
  - Proposed WebSocket Gateway with Bayeux-inspired fallbacks
  - GraphQL Subscriptions integration
  - ConnectRPC Streaming integration
  - Complete code examples (TypeScript + Go)
- **Audience**: Backend engineers, platform engineers, architects

**[proactive-cache-warming.md](./proactive-cache-warming.md)** (3,959 lines)
- **Purpose**: Predictive system for pre-warming RTE caches before members need them
- **Contents**:
  - Member Cron service (scheduled event emitter)
  - ML-based prediction engine for app usage
  - Cache warming interface with Stedi Batch API
  - Integration with Digital Twin architecture
  - Implementation roadmap (Phase 7, Weeks 25-28)
  - Zero wait-time user experience design
- **Audience**: Backend engineers, ML/data teams, architects

**[ACCESS_CONTROL_DESIGN.md](./ACCESS_CONTROL_DESIGN.md)** (1,225 lines)
- **Purpose**: Authorization model for WebSocket Gateway and event filtering
- **Contents**:
  - Four-layer authorization (JWT, Authzed, event filtering, audit logging)
  - Relationship-based access control (members, care teams, family)
  - Event annotation schema (visibility, sensitivity, PHI tracking)
  - HIPAA-compliant audit logging
  - GraphQL + ConnectRPC integration patterns
  - Frontend SDK authorization examples
- **Audience**: Backend engineers, security teams, compliance, architects

**[CLIENT_EVENT_PUBLISHING.md](./CLIENT_EVENT_PUBLISHING.md)** (NEW!)
- **Purpose**: Bidirectional event streaming - frontend clients publish domain events to backend
- **Contents**:
  - Durable client-side event queue (IndexedDB/SQLite)
  - Reliable delivery with acknowledgments
  - Offline support and sync
  - Idempotency and deduplication
  - Event authorization and rate limiting
  - Use cases: analytics, CQRS commands, state sync
  - Complete TypeScript SDK implementation
- **Audience**: Frontend engineers, mobile teams, backend engineers, architects

---

## Quick Navigation by Role

### Backend Engineer
1. Read **EVENT_DRIVEN_RTE_PLAN.md** → Architecture & implementation
2. Review **FAYE_BAYEUX_WEBSOCKET_DESIGN.md** → WebSocket Gateway patterns
3. Refer to **EVENT_DRIVEN_INDEX.md** → FAQ & cross-references

### Frontend Engineer
1. Read **DIGITAL_SESSION_PLATFORM_PLAN.md** → Frontend SDKs & integration
2. Review **FAYE_BAYEUX_WEBSOCKET_DESIGN.md** → Client-side fallback patterns
3. Refer to **EVENT_DRIVEN_INDEX.md** → Quick start guide

### Platform/Infrastructure Engineer
1. Read **FAYE_BAYEUX_WEBSOCKET_DESIGN.md** → WebSocket Gateway implementation
2. Review **EVENT_DRIVEN_RTE_PLAN.md** → Infrastructure requirements
3. Review **PUSHER_RESEARCH_FINDINGS.md** → Pusher replacement strategy
4. Refer to **EVENT_DRIVEN_INDEX.md** → System architecture

### Product Manager / Leadership
1. Read **EVENT_DRIVEN_RTE_SUMMARY.md** → Executive summary & business case
2. Review **EVENT_DRIVEN_INDEX.md** → Overview & timeline
3. Refer to **DIGITAL_SESSION_PLATFORM_PLAN.md** → Frontend use cases

---

## Key Concepts

### Event-Driven Architecture
- **Problem**: 90+ services depend on slow RTE (P95 > 10s), causing timeout cascades
- **Solution**: Decouple request initiation from response delivery via Kafka + WebSocket
- **Benefit**: Zero client timeouts, better user experience, incident reduction

### WebSocket Gateway
- **Purpose**: Push RTE completion events to frontend clients in real-time
- **Transports**: WebSocket (primary), REST polling (fallback)
- **Patterns**: Bayeux-inspired fallbacks (proven by iOS team, member-sponsorship)
- **Features**: Message acknowledgment, heartbeat, exponential backoff, wildcards

### Digital Session Platform
- **Purpose**: Extend event-driven pattern to all frontend use cases
- **Concept**: MemberSession, DigitalEncounter (from Digital Twin architecture)
- **Clients**: MX app, iOS, Android, Care app
- **Events**: RTE updates, CareFlow task updates, encounter updates, The Brain recommendations

### Pusher Channels Replacement
- **Current**: Pusher Channels (video visit events) unreliable (INC-801: 0.17% delivery)
- **Future**: Self-hosted WebSocket Gateway with REST polling fallback
- **Savings**: $1.2-3.6k/year + no vendor lock-in
- **Note**: Pusher Beams already migrated to FCM (RET-402)

---

## Implementation Timeline

| Phase | Duration | Deliverables |
|-------|----------|--------------|
| **Phase 1: Core Infrastructure** | Weeks 1-8 | Kafka, WebSocket Gateway, JWT auth |
| **Phase 2: RTE Integration** | Weeks 9-12 | RTE async RPC, event publishing, frontend SDK |
| **Phase 3: Coverage Server** | Weeks 13-16 | Deprecate parity testing, event-driven flow |
| **Phase 4: Rollout** | Weeks 17-20 | Feature flags, canary, full rollout |
| **Phase 5: Video Visits** | Weeks 21-24 | Replace Pusher Channels for video events |
| **Phase 6: Digital Session Platform** | Weeks 25-28 | Extend to all frontends, CareFlow integration |

---

## Related Documentation

- `./proactive-cache-warming.md` - Proactive cache warming (Phase 7)
- `../../realtime-eligibility/docs/traffic-control.md` - RTE traffic control (Gates 1-3)
- `../../member-sponsorship/docs/explanation/request-deduplication.md` - Hybrid pub/sub + polling pattern
- `../../member-ios-app/App/Features/Sources/VideoVisits/README.md` - iOS Pusher fallback pattern

---

## Questions?

- **Architecture**: See EVENT_DRIVEN_RTE_PLAN.md
- **Business case**: See EVENT_DRIVEN_RTE_SUMMARY.md
- **Frontend integration**: See DIGITAL_SESSION_PLATFORM_PLAN.md
- **WebSocket patterns**: See FAYE_BAYEUX_WEBSOCKET_DESIGN.md
- **Pusher replacement**: See PUSHER_RESEARCH_FINDINGS.md
- **Navigation/FAQ**: See EVENT_DRIVEN_INDEX.md

---

## Status

- **Created**: 2025-11-13
- **Last Updated**: 2025-11-13
- **Status**: Planning/Design Phase
- **Owner**: TJ Singleton
- **Related Tickets**: ACT-2366 (Traffic Control), CLOUD-6296 (GraphQL Inbound Graphs), RET-402 (Pusher Beams Migration)
