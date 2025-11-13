# Event-Driven RTE System - Executive Summary

**Full Plan**: See `EVENT_DRIVEN_RTE_PLAN.md` (2074 lines)

---

## The Problem

**Current Architecture**: Clients wait synchronously for slow Stedi API responses (P95 > 10s, can exceed 120s)

**Impact**:
- 8 major incidents in 14 months (6 SEV-1, 2 SEV-2)
- 5-10% timeout failure rate during slow payer periods
- Poor user experience (30s waits, frequent failures)
- Batch jobs saturate 15-slot Stedi concurrency limit → cascading failures

**Timeout Locations Found**:
| Service | File | Timeout Value |
|---------|------|---------------|
| coverage-server | `app/gateway/rte/rte_gateway.go:182` | 90 seconds |
| member-sponsorship | `app/gateway/rteproxy/rte_proxy_gateway.go:80` | 60 seconds |
| Frontend | GraphQL/HTTP clients | 10-30 seconds |
| realtime-eligibility | Temporal workflows | 120+ seconds |

---

## Comparison: Agent Platform Streaming vs. Event-Driven RTE

### Context: IncludedHealth Already Uses Streaming

The **Agent Platform** (member chat AI) already implements **real-time streaming** for LLM responses. Understanding this implementation helps contextualize the event-driven RTE approach.

#### Agent Platform Streaming Architecture

**How it works**:
```python
# Agent Platform: Python async generator pattern
async def live_streaming_generator():
    """Stream LLM chunks in real-time to Sendbird (messaging)"""
    async for agent_response in session_manager.astream_chat(messages):
        for response in agent_response.responses:
            yield (text_chunk, is_first_chunk)  # Stream to frontend

# Sendbird receives chunks through ONE HTTP connection
await messaging_client.send_streaming_chunks(
    bot_user_id,
    channel_url,
    live_streaming_generator()  # Generator yields chunks as LLM produces them
)
```

**Key characteristics**:
- **Technology**: Python async generators + HTTP POST streaming
- **Use case**: LLM text generation (Claude/GPT responses)
- **Latency**: Users see text appear in 1-2 seconds as LLM generates
- **Connection**: Single HTTP connection, kept open during generation
- **Durability**: Temporal handles retries at activity level (if stream fails, retry entire conversation turn)
- **Frontend**: Sendbird messaging UI (iOS/Android native SDKs)

### Why RTE Needs a Different Approach

While the agent platform uses **synchronous streaming** (one connection, real-time chunks), RTE requires **asynchronous events** due to fundamental differences:

| Aspect | Agent Platform (LLM Streaming) | RTE Event-Driven |
|--------|-------------------------------|------------------|
| **Duration** | 2-10 seconds (LLM generation) | 10-120 seconds (Stedi API call) |
| **Process** | Stream of text chunks | Long-running external API call |
| **User expectation** | Watch text appear live | Submit request, check later |
| **Connection** | User actively waiting | User may close app |
| **Retry semantics** | Retry entire conversation turn | Retry at multiple levels (cache, workflow, slot) |
| **Concurrency** | Unlimited (each user gets LLM) | 15 concurrent slots (shared resource) |
| **Backend complexity** | LLM inference (fast) | Scheduler, fairness, rate limiting |

### Why Not Use Streaming for RTE?

**Option**: Use Python async generators for RTE like agent platform
```python
# Hypothetical: Stream RTE progress
async def stream_rte_progress():
    yield "Queuing request..."
    yield "Waiting for slot..."
    yield "Calling Stedi..."
    yield "Response received!"
```

**Why this doesn't work well**:

1. **Long Wait Times**: 
   - LLM streaming: Text appears immediately, keeps user engaged
   - RTE: User waits 30-120s with nothing happening → bad UX

2. **Connection Management**: 
   - LLM: Connection open for 10s max
   - RTE: Connection open for 120s → timeout issues, resource waste

3. **Mobile/Offline**:
   - LLM: User must stay in app to see response
   - RTE: User should be able to close app, get notified when done

4. **Retry Complexity**:
   - LLM: Retry entire turn (regenerate from start)
   - RTE: Complex retry (scheduler, cache, workflow) → can't replay generator

5. **Multiple Consumers**:
   - LLM: One user waits for response
   - RTE: Multiple systems need result (frontend, cache updater, analytics, etc.)

### The Event-Driven Solution

**Instead of streaming progress, emit events at state transitions**:

```typescript
// Event-driven: Submit and disconnect
const sr = await session.submitServiceRequest({ service: 'RTE' });
// Returns immediately with workflow ID

// Subscribe to events (push notifications)
sr.on('queued', () => console.log('In queue...'));
sr.on('scheduled', () => console.log('Slot acquired...'));
sr.on('processing', () => console.log('Calling Stedi...'));
sr.on('completed', (result) => displayCoverage(result));

// User can close app - events delivered via push notification
```

**Key advantages over streaming**:

1. **Decoupled**: Frontend doesn't hold connection open
2. **Durable**: Events persist in Kafka (can replay, audit)
3. **Multi-consumer**: Multiple services consume same events
4. **Offline-capable**: Push notifications when app closed
5. **Retry-friendly**: Emit new events on retry, don't replay generator
6. **Scalable**: Kafka handles 1000s of events/sec, not connection-limited

### When to Use Each Pattern

| Pattern | Best For | IncludedHealth Use Cases |
|---------|----------|-------------------------|
| **Synchronous Streaming** (Agent Platform) | Fast, continuous data generation (2-10s) | • LLM chat responses<br>• AI-generated content<br>• Real-time analysis results |
| **Event-Driven Push** (This Plan) | Long-running processes (10-120s+) | • RTE eligibility checks<br>• Coverage verification<br>• Enrollment workflows<br>• Claims processing<br>• Prior authorization |
| **Polling** (Current RTE) | ❌ Deprecated (causes timeout issues) | Legacy - migrate to events |

### Hybrid Approach: Best of Both Worlds

**For optimal UX, combine both patterns**:

```typescript
// Example: Care Plan generation with LLM

// 1. Submit long-running service request (event-driven)
const carePlanSR = await session.submitServiceRequest({
  service: 'GENERATE_CARE_PLAN',
  member_id: memberId,
});

// 2. Listen for events (long-running steps)
carePlanSR.on('queued', () => showProgress('Analyzing member data...'));
carePlanSR.on('processing', () => showProgress('Generating plan with AI...'));

// 3. When LLM step starts, switch to streaming (real-time text)
carePlanSR.on('llm_generation_started', async (streamUrl) => {
  const stream = await connectToStream(streamUrl);
  for await (const chunk of stream) {
    appendText(chunk);  // Show AI writing in real-time
  }
});

// 4. Final event (durable result)
carePlanSR.on('completed', (plan) => savePlan(plan));
```

**This gives us**:
- ✅ Event-driven orchestration for long-running steps
- ✅ Real-time streaming for LLM generation (engaging UX)
- ✅ Durability and multi-consumer support (Kafka events)
- ✅ Best user experience (instant feedback + real-time text)

---

## The Solution

**Event-Driven Architecture**: Replace synchronous request-response with asynchronous event push notifications

### Core Components

1. **Event Producer Service** (NEW) - Emits RTE completion/failure/timeout events to Kafka
2. **Kafka Event Topics** (NEW) - Durable event stream with 7-30 day retention
3. **WebSocket Gateway** (NEW) - Pushes events to connected frontend clients in real-time
4. **Event Consumer Framework** (NEW) - Go library for consuming events in backend services

### Flow Comparison

#### Current (Synchronous)
```
User clicks "Check Coverage" 
  → Frontend waits 30s 
  → Timeout error 
  → User retries
```

#### Proposed (Event-Driven)
```
User clicks "Check Coverage"
  → Instant "Processing..." response
  → User sees spinner (not frozen)
  → Push notification when ready (10-30s later)
  → Coverage displayed
  → No timeout failures
```

---

## Key Benefits

1. **Eliminates Timeouts**: Frontend gets instant response, not blocked by slow Stedi API
2. **Reduces Incidents**: 8 incidents in 14 months → <2 per year target
3. **Improves UX**: Instant feedback, real-time updates, no frozen screens
4. **Prevents Saturation**: Batch jobs process async, don't block user traffic
5. **Enables Proactive Cache Warming** ⭐: ML-predicted cache warming → <500ms coverage display (see Advanced Feature section below)

---

## Opportunities by Service

### coverage-server (7 opportunities)

1. **GraphQL Coverage Queries** - Add subscription support for async updates
2. **Enrollment Instant Coverage (ACT-2819)** - Eliminate first-enrollment failures with event-driven finalization
3. **Parity Testing** - Async experiment execution, compare results via events
4. **Timeout Handling** - Remove 90s polling loops
5. **Cache Updates** - Consume events to update cache (eliminate polling)
6. **Downtime Notifications** - Real-time DTE alerts to frontend
7. **Metrics Collection** - Event-driven observability pipeline

**Files to Modify**:
- `app/gateway/rte/rte_gateway.go` - Remove polling, add async methods
- `app/graph/coverage.graphqls` - Add subscription types
- `app/servicev2/enrollment/enrollment.go` - Add pending states, event-driven finalization
- NEW: `app/consumer/rte/consumer.go` - Event consumer

### member-sponsorship (5 opportunities)

1. **GetMemberSponsorship RPC** - New async endpoint, event-driven cache updates
2. **Async Beneficiary Verification** - Remove blocking RTE calls, finalize via events
3. **Enrollment Processing** - Pending state enrollments, real-time status updates
4. **Timeout Elimination** - Remove 60s polling
5. **Cache Updates** - Event-driven cache population

**Files to Modify**:
- `app/gateway/rteproxy/rte_proxy_gateway.go` - Remove polling
- `proto/sponsorship.proto` - Add async RPC methods
- `app/service/enrollment/service.go` - Pending states
- NEW: `app/consumer/rte/consumer.go` - Event consumer

### realtime-eligibility (3 opportunities)

1. **Event Emission** - Emit events on workflow completion/failure/timeout
2. **Proactive Cache Warming** - Consume member-cron events, pre-warm cache
3. **Metrics & Observability** - Event-driven metrics pipeline

**Files to Modify**:
- `app/workflows/fetchrte/v1/internal/fetch_rte_worker.go` - Emit events
- NEW: `app/producer/events/producer.go` - Event producer
- NEW: `app/consumer/membercron/consumer.go` - Cache warming

### Frontend (2 opportunities)

1. **Real-Time Coverage Updates** - WebSocket subscriptions, push notifications
2. **Offline Support** - Submit request while online, receive result notification later

**Files to Create**:
- NEW: `frontend-sdk/src/events/websocket.ts` - WebSocket client
- NEW: `frontend-sdk/src/events/reconnect.ts` - Auto-reconnect logic

---

## Advanced Feature: Proactive Cache Warming

### The Ultimate Goal: Zero Wait Time

The event-driven infrastructure enables a powerful advanced feature: **Proactive Cache Warming** (detailed in `./proactive-cache-warming.md`).

#### How It Works

```
8:00am: ML model predicts member will open app at 9am
        ↓
8:30am: Member Cron emits event (app_usage_probability: 0.85)
        ↓
8:31am: RTE Cache Warmer consumes event
        ↓
8:32am: Submit batch RTE request (Stedi Batch API - doesn't use realtime slots)
        ↓
8:45am: Cache populated (15-minute batch processing)
        ↓
9:00am: Member opens app → Cache hit → Instant (<500ms) ✨
```

**Result**: Member doesn't even know we called Stedi - they just see instant coverage!

#### Four Components

1. **Member Cron** - Scheduled event emitter with ML predictions
   - Emits `member-cron-events` daily for active members
   - Enriched with app usage probability (ML model)
   - Kafka topic: `member-cron-events`

2. **Cache Control Parameter** - Explicit cache behavior
   - PREFER_CACHE, REQUIRE_FRESH, DEFAULT modes
   - Allow clients to opt into cached data

3. **Batch RPC Interface** - Non-realtime bulk processing
   - Use Stedi Batch API (10,000 checks/batch)
   - Doesn't consume realtime slots (unlimited concurrency)
   - Takes 15-30 min (acceptable for proactive warming)

4. **Cache Warming Consumer** - Predictive cache population
   - Subscribes to Member Cron events
   - Filters high-probability members (p > 0.7)
   - Pre-warms cache before predicted activity

#### Integration with Digital Twin

Member Cron is part of IncludedHealth's **Digital Twin architecture** (2025 Tech Vision):

- **SENSE**: Digital Twin ingests member data + ML predictions
- **DECIDE**: The Brain generates Next Best Actions
- **ACT**: CareFlow + RTE Cache Warmer execute actions

**Multiple consumers** of Member Cron events:
- ✅ RTE Cache Warmer (pre-warm eligibility caches)
- ✅ The Brain (trigger recommendations)
- ✅ Air Traffic Control (schedule proactive outreach)
- ✅ Care Teams (surface members needing follow-up)
- ✅ CareFlow (trigger scheduled care actions)

#### Expected Impact

| Metric | Without Warming | With Warming | Improvement |
|--------|----------------|--------------|-------------|
| Cache hit rate | 40-50% | 60-70% | +20% |
| User-perceived latency | 10-30s | <500ms | 95% faster |
| Realtime slot saturation | 5-10 incidents/year | 0 incidents | 100% elimination |
| Member satisfaction | Baseline | Significant increase | Instant coverage |

#### When Implemented

**Phase 7** (Weeks 25-28) of the event-driven RTE plan:
- Week 25-26: Member Cron service with ML predictions
- Week 27-28: RTE Cache Warmer consumer + Stedi Batch API

**Depends on**: Phases 1-6 (event infrastructure must be in place)

**See**: `./proactive-cache-warming.md` for 3960-line full specification

---

## Replacing Pusher Channels: Strategic Opportunity

### CRITICAL CLARIFICATION

IncludedHealth uses **TWO separate Pusher products**:

1. **Pusher Beams** (Web push notifications) → ✅ **ALREADY MIGRATED TO FIREBASE (Oct 2025)**
   - Epic: RET-402, complete
   - No action needed!

2. **Pusher Channels** (Real-time WebSocket) → ⚠️ **OPPORTUNITY FOR REPLACEMENT**
   - Video visit events (call_accepted, call_ended, join_call, etc.)
   - Real-time presence
   - Cost: ~$99-299+/month ($1.2-3.6k+/year)

### The Opportunity

**Key Insight**: We're already building a WebSocket Gateway for the event-driven RTE system. **Extending it to replace Pusher Channels is nearly free!**

### Evidence: Pusher Channels is Unreliable

**From iOS Video Visits README**:
> "Because **Pusher is not 100% reliable**, we use the call-status API to get the end status whenever we receive the roomCompleted or roomNotFound events **as well as the pusher event**."

**INC-801 Incident** (Pusher Beams, June 2025):
- Only 97 of 57,000 notifications delivered (0.17% success rate)
- Support took 7-8 days to respond
- Quote: *"PusherBeam support is unreliable non-existent (Beside 1 AI bot)"*

### Benefits of Self-Hosted WebSocket Gateway

| Pusher Channels (Current) | Self-Hosted | Advantage |
|---------------------------|-------------|-----------|
| ~$99-299+/month | $0 (reuses RTE infrastructure) | $1.2-3.6k+/year savings (Year 2+) |
| Vendor lock-in | Full control | Own our destiny |
| Unreliable (iOS docs) | Reliable | No fallback polling needed |
| Fixed features | Extensible | Custom optimizations |
| Separate system | Unified platform | Kafka + WebSocket |

### What's Required

**5-Month Migration Plan** (overlaps with RTE Phases 1-4):

1. **Weeks 1-8**: Build WebSocket Gateway for RTE (Go + Redis + Kafka)
2. **Weeks 9-12**: Frontend SDKs (TypeScript, Swift, Kotlin)
3. **Weeks 13-16**: Migrate Pusher Channels events (video visits)
   - `call_accepted`, `call_ended`, `join_call`, `appointment_ready`, `paid_extend_call_request`
4. **Weeks 17-20**: Deprecate Pusher Channels, remove GraphQL field

**Incremental Cost**: ~$10-15k (client SDKs + migration)  
**Annual Savings**: $1.2-3.6k+ (Year 2+)  
**Strategic Value**: Immediate (full control, reliability, unified platform)

### Migration Strategy

**Dual-Run Period**:
- Keep Pusher running
- Add WebSocket gateway support
- Feature flag to toggle
- Gradual rollout (5% → 25% → 50% → 100%)

**Current Pusher Channels GraphQL**:
```graphql
query pusherInfo {
  me {
    member {
      pusherSubscribeInfo {
        appKey              # Pusher Channels app identifier
        presenceChannelName # presence-member-{id}
        authURL             # Channel authentication endpoint
      }
    }
  }
}
```

**New WebSocket GraphQL** (alongside pusherSubscribeInfo during dual-run):
```graphql
query websocketInfo {
  me {
    member {
      websocketSubscribeInfo {
        url          # wss://events.includedhealth.com/v1/ws
        authToken    # JWT for connection
      }
    }
  }
}
```

**Video Visit Events** (replaces Pusher Channels):
- `call.accepted` → replaces Pusher `call_accepted`
- `call.ended` → replaces Pusher `call_ended`
- `call.join` → replaces Pusher `join_call`
- `call.ready` → replaces Pusher `appointment_ready`
- `call.extend` → replaces Pusher `paid_extend_call_request`

### Recommendation

**Option 1: Full Migration** (20 weeks)
- Replace Pusher completely
- Single unified event platform
- Long-term strategic value

**Option 2: Incremental** (Recommended)
- Start with WebSocket Gateway for RTE events (Weeks 1-12)
- Extend to video visit events (Weeks 13-16)
- Deprecate Pusher Channels (Weeks 17-20)
- **Incremental cost**: Only $10-15k (infrastructure shared with RTE)

**Rationale**: 
- ✅ **Build once, use twice** - Single infrastructure for RTE + video visits
- ✅ **Proven fallback** - iOS already polls when Pusher fails
- ✅ **INC-801 lessons** - Vendor support unreliable (7-8 day response)
- ✅ **Lower total cost** - Shared operations, no additional infrastructure

**See**: 
- `EVENT_DRIVEN_INDEX.md` → "Replacing Pusher Channels" for 450+ line detailed specification
- `PUSHER_RESEARCH_FINDINGS.md` → Complete Glean research on Pusher usage

---

## Implementation Timeline

**Total Duration**: 28 weeks (7 months)

### Phase 1: Foundation (Weeks 1-4)
- Kafka cluster setup (UAT)
- Event producer service
- Event consumer framework (go-common)

### Phase 2: RTE Event Emission (Weeks 5-8)
- Emit events from realtime-eligibility
- Shadow mode (events emitted but not required)
- Validation & observability

### Phase 3: Backend Consumption (Weeks 9-12)
- coverage-server event consumer
- member-sponsorship event consumer
- Cache updates via events (shadow mode)

### Phase 4: Async Endpoints (Weeks 13-16)
- Async GraphQL mutations
- Async RPC methods
- Backward compatible

### Phase 5: WebSocket Gateway (Weeks 17-20)
- WebSocket gateway service
- Frontend SDK
- Push notifications

### Phase 6: Migration & Cleanup (Weeks 21-24)
- Gradual traffic migration (10% → 100%)
- Remove polling logic
- Deprecate sync endpoints

### Phase 7: Advanced Features (Weeks 25-28)
- Proactive cache warming (Member Cron)
- Stedi Batch API integration
- Advanced metrics & dashboards

---

## Success Metrics

### Before/After Targets

| Metric | Current | Target | Improvement |
|--------|---------|--------|-------------|
| Frontend timeout rate | 5-10% | <1% | 90% reduction |
| Enrollment first-attempt success | 60-70% | >95% | +35% |
| RTE-related incidents | 8 in 14mo | <2 per year | 75% reduction |
| Mean time to coverage display | 10-30s | 500ms + async | 95% faster |
| Stedi slot saturation incidents | 3 in 14mo | 0 | 100% elimination |

---

## Technical Architecture

### Event Flow Diagram

```
Frontend                     WebSocket Gateway              Kafka                Backend Services
   │                              │                          │                         │
   │──Subscribe to events────────>│                          │                         │
   │                              │                          │                         │
   │──Submit RTE request─────────────────────────────────────┼────────────────────────>│
   │                              │                          │                         │
   │<─Instant response (workflow ID)                         │                         │
   │  "Processing..."             │                          │                         │
   │                              │                          │                         │
   │                              │                          │<────Emit completion event│
   │                              │<────Consume event────────┤                         │
   │                              │                          │                         │
   │<─Push notification───────────│                          │                         │
   │  "Coverage ready!"           │                          │                         │
   │                              │                          ├────────────────────────>│
   │                              │                          │  Update cache/DB        │
```

### Event Types

| Event Type | Purpose | Consumers |
|-----------|---------|-----------|
| `rte.submitted` | Request submitted | Metrics, audit |
| `rte.enqueued` | Added to queue | Monitoring |
| `rte.scheduled` | Slot assigned | Progress bars |
| `rte.started` | Stedi call started | Timeout tracking |
| `rte.completed` | Success | All consumers |
| `rte.failed` | Failure | All consumers |
| `rte.timeout` | Timeout | All consumers |
| `rte.downtime` | Payer down | Downtime tracking |

---

## Migration Strategy

### Backward Compatibility Approach

1. **Shadow Mode** (Weeks 5-12): Events emitted but not required, old polling still works
2. **Async Endpoints Added** (Weeks 13-16): New endpoints alongside old ones
3. **Feature Flags** (Weeks 17-20): Gradual rollout 10% → 50% → 100%
4. **Deprecation** (Week 21): Sync endpoints deprecated with 6-month notice
5. **Removal** (6 months later): Old polling code removed

### Feature Flags

| Flag | Purpose | Default | Rollout Week |
|------|---------|---------|--------------|
| `enable-rte-events` | Emit events | false → true | Week 5 |
| `enable-rte-event-consumers` | Consume events | false → true | Week 9 |
| `enable-async-rte-endpoints` | Async APIs | false → true | Week 13 |
| `enable-websocket-gateway` | WebSocket push | false → true | Week 17 |
| `disable-rte-polling` | Remove polling | false → true | Week 21 |

### Rollback Plan

**If issues arise**:
1. Disable feature flag (instant rollback)
2. Services fall back to polling
3. Events continue flowing (no data loss)
4. Fix issue, re-enable

---

## Infrastructure Requirements

### New Components

1. **Kafka Cluster**
   - 3 brokers (HA)
   - 50-100 partitions per topic
   - Retention: 7-30 days
   - Cost estimate: $5k-10k/month (production)

2. **Event Producer Service**
   - Go microservice
   - 3 replicas (HA)
   - Minimal resource requirements

3. **WebSocket Gateway**
   - Go microservice
   - Auto-scaling (10k connections per instance)
   - Redis for connection state

4. **Event Consumer Instances**
   - In existing services (coverage, member-sponsorship)
   - 2-3 replicas per consumer group

### Estimated Costs

| Component | Cost/Month (Prod) |
|-----------|------------------|
| Kafka cluster (AWS MSK) | $3k-5k |
| Event producer service | $500 |
| WebSocket gateway | $1k-2k |
| Additional consumer instances | $1k |
| **Total** | **$5.5k-8.5k** |

**Cost Offset**: Reduced CPU from polling removal (~$2k/month savings)

**Net Cost**: ~$3.5k-6.5k/month

---

## Risks & Mitigations

| Risk | Impact | Probability | Mitigation |
|------|--------|-------------|------------|
| Kafka cluster outage | Events not delivered | Low | 3-broker cluster, fallback to polling |
| Event consumer lag | Stale updates | Medium | Auto-scaling, alerting |
| WebSocket instability | Missed events | Medium | Auto-reconnect, event replay |
| Schema evolution issues | Parse failures | Low | Schema registry, backward compatibility |
| Increased complexity | Ops burden | High | Managed Kafka, reusable libraries, training |

---

## Next Steps

1. **Review & Approval** (Week 0)
   - Architecture review with team
   - Budget approval for infrastructure
   - Assign engineers to project

2. **Kickoff** (Week 1)
   - Provision UAT Kafka cluster
   - Create event producer service skeleton
   - Set up monitoring/dashboards

3. **Weekly Demos** (Weeks 1-28)
   - Demo progress every Friday
   - Adjust timeline based on learnings

4. **Go-Live** (Week 21)
   - Gradual rollout to production
   - Monitor metrics closely
   - Be ready to rollback

---

## Key Files Modified/Created

### New Microservices (3)
- `event-producer/` - Kafka event producer service
- `websocket-gateway/` - WebSocket push notification gateway
- `member-cron/` - Scheduled member event emitter (for cache warming)

### New Shared Libraries (2)
- `go-common/eventconsumer/` - Event consumer framework
- `frontend-sdk/src/events/` - WebSocket client SDK

### Modified Services (3)

**coverage-server**:
- Modified: `app/gateway/rte/rte_gateway.go` (remove polling)
- Modified: `app/servicev2/enrollment/enrollment.go` (pending states)
- Created: `app/consumer/rte/consumer.go` (event consumer)

**member-sponsorship**:
- Modified: `app/gateway/rteproxy/rte_proxy_gateway.go` (remove polling)
- Modified: `proto/sponsorship.proto` (async RPCs)
- Created: `app/consumer/rte/consumer.go` (event consumer)

**realtime-eligibility**:
- Modified: `app/workflows/fetchrte/v1/internal/fetch_rte_worker.go` (emit events)
- Created: `app/producer/events/producer.go` (event producer)

---

## Alternative Architectures Considered (and why not chosen)

1. **Server-Sent Events (SSE)** - Simpler but one-way only, less flexible
2. **GraphQL Subscriptions** - Too coupled to Apollo, not generic
3. **gRPC Streaming** - Poor browser support
4. **Redis Pub/Sub** - No durability, no replay capability
5. **AWS EventBridge** - Vendor lock-in, higher latency/cost

**Chosen**: Kafka + WebSocket for durability, flexibility, and performance

---

## Related Documentation

- **Full Plan**: `EVENT_DRIVEN_RTE_PLAN.md` - Comprehensive 2074-line specification
- **RTE Docs**: `realtime-eligibility/docs/traffic-control.md` - Current architecture
- **Cache Warming**: `./proactive-cache-warming.md` - Member Cron design
- **Request Chains**: `coverage/REQUEST_CHAINS_ACT_2819.md` - Timeout analysis
- **Implementation Plan**: `coverage/IMPLEMENTATION_PLAN_ACT_2819.md` - ACT-2819 context

---

**Document Version**: 1.0  
**Created**: 2025-01-15  
**Status**: DRAFT  
**Owner**: TBD
