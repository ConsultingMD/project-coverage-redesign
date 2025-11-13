# Pusher Research Findings from Glean

## Executive Summary

IncludedHealth uses **TWO different Pusher products**:

1. **Pusher Beams** - Web push notifications → **ACTIVELY BEING MIGRATED TO FIREBASE**
2. **Pusher Channels** - Real-time WebSocket communications → **STILL IN ACTIVE USE**

## Pusher Beams (Web Push Notifications)

### Status: MIGRATION IN PROGRESS

**Epic**: RET-402 - Move away from PusherBeam as third party vendor for Web Push  
**Proposal**: [Replace Pusher Beams with Firebase Cloud Messaging](https://includedhealth.atlassian.net/wiki/spaces/ENG/pages/5344264259)  
**EDD**: [PusherBeam migration to FCM](https://includedhealth.atlassian.net/wiki/spaces/ENG/pages/5357830231)

### Why Migrating?

**Critical Incident**: INC-801 (June 2025) - Web pushes completely blocked

**Four Critical Issues**:
1. **Slow support response times** - Took 7-8 days to get response, support tickets abruptly closed
2. **Connection limit quotas** - Exceeded subscription quota (stale connections not cleared)
3. **Unreliable delivery** - Only 97 of 57K push notifications delivered in 7 days (0.17%)
4. **Vendor lock-in** - Single person ownership, no company-level visibility

**Incident Impact**:
- Severity: SEV-2 (EPDD), SEV-3 (Members)
- Duration: Partially mitigated / Unable to fully mitigate
- Root cause: Lack of monitoring, stale connection cleanup, vendor support unreliable

**Quote from incident report**:
> "That PusherBeam support is unreliable non-existent (Beside 1 AI bot that auto-replies to emails)"

### Migration Timeline

**M1: Core FCM Integration** (1-2 weeks)
- Backend replaces Pusher Beams API calls with FCM APIs
- ~1 BE engineer

**M2: SFMC Hook Migration** (1-2 weeks)
- Update SFMC hook to call Notification Service with correct payload
- Fallback to legacy Pusher if urgent (short-term)

**M3: Pusher Decommission & Silent Push Rollout**
- Remove all Pusher Beams logic
- Enable silent push notifications (not available with Pusher)

**Status as of Oct 2025**: Migration complete, Pusher Beams deprecated

---

## Pusher Channels (Real-Time WebSocket)

### Status: STILL IN ACTIVE USE

**No migration plans identified in Glean search results**

### Current Usage

#### 1. Video Visits (Urgent Care, Appointments)

**GraphQL Query**: `pusherInfo` / `pusherSubscribeInfo`

```graphql
query pusherInfo {
  me {
    member {
      pusherSubscribeInfo {
        appKey
        presenceChannelName
        authURL
      }
    }
  }
}
```

**Clients**:
- iOS app (member-ios-app)
- Android app (member-android-app)
- Web (jarvis)

**Key Events**:
- `call_accepted` - Provider accepted call
- `appointment_ready` - Appointment ready to join
- `call_ended` - Call completed
- `join_call` - Notification to join video visit
- `paid_extend_call_request` - Provider extending call time

**Implementation Details** (from iOS README):

> "When the provider ends the call, the client side will receive a callback from the Zoom SDK that the provider has left the room and will check the call status using the memberCall() method... the call_ended pusher and the roomCompleted events will be received. **Because Pusher is not 100% reliable**, we use the call-status API to get the end status whenever we receive the roomCompleted or roomNotFound events as well as the pusher event."

**Key Insight**: iOS app already has fallback logic because **"Pusher is not 100% reliable"**

#### 2. Real-Time Presence

**Use case**: Show which members/providers are online

**Implementation**: Presence channels authenticated via `authURL`

---

## Technical Details

### Pusher Channels Architecture

```
Member iOS/Android/Web
    ↓ (Query pusherSubscribeInfo)
coverage-server (GraphQL)
    ↓ (Return appKey, presenceChannelName, authURL)
Member Client
    ↓ (Connect to Pusher Channels WebSocket)
Pusher Channels SaaS
    ↑ (Backend publishes events)
Backend Services (Call Service, etc.)
```

### Authentication Flow

1. Frontend queries `pusherSubscribeInfo` (GraphQL)
2. Backend returns:
   - `appKey` - Pusher app identifier
   - `presenceChannelName` - Channel name (e.g., `presence-member-{id}`)
   - `authURL` - Endpoint for channel authentication
3. Frontend connects to Pusher Channels WebSocket
4. Frontend authenticates using `authURL` token

### Reliability Issues Documented

From iOS app README:
- **"Pusher is not 100% reliable"** - iOS team built fallback polling
- Call status checked via API as backup
- Polling used when Pusher events miss

---

## Cost Analysis

### Pusher Beams (Web Push)

**Historical Cost**: $99-499/month  
**Issues**: Connection quota limits, unreliable delivery (0.17% success rate during incident)  
**Replacement**: Firebase Cloud Messaging (FCM) - Free tier sufficient  
**Savings**: $1.2-6k/year

### Pusher Channels (WebSocket)

**Current Cost**: Unknown (not found in Glean search)  
**Estimated**: Likely $99-299+/month based on Pusher pricing tiers  
**Annual**: $1.2-3.6k+/year

---

## Strategic Recommendations

### Recommendation 1: Replace Pusher Channels (Aligned with Event-Driven Plan)

**Why**:
1. ✅ **Already building WebSocket Gateway** for RTE event-driven system
2. ✅ **iOS team already has fallback logic** - knows Pusher is unreliable
3. ✅ **Vendor lock-in concerns** - INC-801 showed poor support response
4. ✅ **Cost savings** - $1.2-3.6k+/year
5. ✅ **Full control** - No quota limits, custom optimizations

**Incremental Cost**: ~$10-15k (since WebSocket Gateway already planned)

**Timeline**: 16-20 weeks (overlapping with RTE event-driven Phases 1-5)

### Recommendation 2: Keep Pusher Channels Temporarily

**Why**:
1. ✅ **Pusher Beams migration already in progress** - Don't overwhelm team
2. ✅ **Video visits working** - Don't break what's working
3. ✅ **Focus on RTE timeouts first** - Solve critical problem first
4. ⚠️ **Migrate later** - After RTE system stable (6+ months)

**Action Items**:
- Add monitoring/alerting (learn from INC-801)
- Implement connection cleanup (prevent quota issues)
- Document fallback logic (polling when Pusher fails)

---

## Key Findings Summary

| Aspect | Pusher Beams | Pusher Channels |
|--------|--------------|-----------------|
| **Status** | Migrating to FCM (Oct 2025) | Active use |
| **Use Case** | Web push notifications | Video call events, presence |
| **Reliability** | 0.17% delivery (incident) | "Not 100% reliable" (iOS docs) |
| **Support** | Unreliable (7-8 day response) | Unknown |
| **Cost** | $99-499/mo | Unknown (~$99-299+/mo) |
| **Migration Plan** | Complete (FCM) | None (opportunity!) |
| **Clients** | Web only | iOS, Android, Web |
| **Replacement** | Firebase (free tier) | Self-hosted WebSocket Gateway |

---

## References

**Confluence**:
- [Proposal – Replace Pusher Beams with Firebase Cloud Messaging](https://includedhealth.atlassian.net/wiki/spaces/ENG/pages/5344264259)
- [EDD - PusherBeam migration to FCM](https://includedhealth.atlassian.net/wiki/spaces/ENG/pages/5357830231)
- [Care App - Practitioner Notifications](https://includedhealth.atlassian.net/wiki/spaces/ENG/pages/4215799826)

**Jira**:
- [RET-402: Move away from PusherBeam](https://includedhealth.atlassian.net/browse/RET-402)
- [INC-801: Web Pushes are blocked](https://includedhealth.atlassian.net/browse/INC-801)

**GitHub**:
- [member-ios-app Video Visits README](https://github.com/ConsultingMD/member-ios-app/blob/main/App/Features/Sources/VideoVisits/README.md)
- [UnifiedAPI get_pusher_key.graphql](https://github.com/ConsultingMD/UnifiedAPI/blob/main/operations/get_pusher_key.graphql)

**Incident Report**:
- [INC-801: Web Pushes are blocked (Google Doc)](https://docs.google.com/document/d/1s470TicedvdRb9qiCOsS7rHfYHHjf-PkLYB7yam7nqg)

---

## Action Items for Event-Driven Plan

### Update Documentation

1. ✅ **Clarify two Pusher products** - Beams vs Channels
2. ✅ **Note Pusher Beams already migrated** - Don't duplicate work
3. ✅ **Focus on Pusher Channels replacement** - Real opportunity
4. ✅ **Highlight iOS fallback logic** - Proves Pusher unreliable
5. ✅ **Add INC-801 context** - Vendor support concerns

### Update Cost Analysis

1. ✅ **Remove Pusher Beams savings** - Already migrated to FCM
2. ✅ **Focus on Pusher Channels cost** - $1.2-3.6k+/year
3. ✅ **Emphasize strategic value** - Full control, no vendor lock-in

### Update Migration Strategy

1. ✅ **Video visit events** - Specific use case (join_call, call_ended, etc.)
2. ✅ **Real-time presence** - Optional (nice-to-have)
3. ✅ **Fallback logic** - iOS already polls, extend to all clients
4. ✅ **Monitoring** - Learn from INC-801 (alerting, connection cleanup)

---

## Quotes to Include

> "Because Pusher is not 100% reliable, we use the call-status API to get the end status whenever we receive the roomCompleted or roomNotFound events as well as the pusher event."  
> — iOS Video Visits README

> "That PusherBeam support is unreliable non-existent (Beside 1 AI bot that auto-replies to emails)"  
> — INC-801 Incident Report

> "Only 97 of 57K push notifications delivered in 7 days (0.17%)"  
> — INC-801 Incident Report

