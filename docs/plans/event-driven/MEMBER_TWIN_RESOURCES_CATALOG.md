# MemberTwin Resources & Documents – Draft Catalog

> Draft proposal of what kinds of resources and documents the **MemberTwin** should expose via MCP.

---

## 1. Design Principles

Before listing specific resources, a few principles:

1. **Member-centric, not system-centric**  
   Resources are organized around how a human would think about a member (identity, coverage, care, money, communication), not around individual services (CareFlow, claims, EHR connectors).

2. **Views, not raw tables**  
   Each document type is a *view* tuned for a use case (guide, clinician, member self-service, agent) and prompt-friendly, rather than a direct mirror of an underlying schema.

3. **Stable resource URIs + typed payloads**  
   Every resource has a stable MCP URI (e.g., `mcp://twins/member/{id}/careflow/{service_delivery_id}`) and resolves to a Schema Registry-backed type.

4. **Privacy-first**  
   All documents include field-level annotations for PHI/PII categories and purpose-of-use so MemberTwin can enforce minimum-necessary sharing per tool and caller.

5. **Incremental slices**  
   We start with a narrow but high-value slice (identity, open actions, key benefits, recent care) and expand over time.

---

## 2. Top-Level MCP Resource Layout

At a high level, MemberTwin could expose a resource tree like this:

- `mcp://twins/member/{member_id}`  
  Root handle for the member twin.
  - `/profile` – canonical member identity & enrollment profile.
  - `/coverage` – current and upcoming coverage & benefits.
  - `/care/summary` – compact care summary across conditions, plans, programs.
  - `/care/plan/{plan_id}` – longitudinal care plan views (CareFlow-backed).
  - `/care/task/{task_id}` – individual tasks/actions.
  - `/encounter/{encounter_id}` – visits, telehealth encounters, hospitalizations.
  - `/clinical/condition/{condition_id}` – problem list / major conditions.
  - `/clinical/medication/{med_id}` – active and key historical meds.
  - `/clinical/lab/{lab_result_id}` – important labs & diagnostics.
  - `/program/{program_enrollment_id}` – IPU/program enrollments (e.g., BH, chronic care).
  - `/benefit/{benefit_doc_id}` – benefit/coverage summary docs for common questions.
  - `/financial/claim/{claim_id}` – claims & EOB views (highly summarized at first).
  - `/financial/estimate/{estimate_id}` – cost estimate summaries.
  - `/communication/thread/{thread_id}` – message threads and outreach history.
  - `/preferences` – communication preferences, language, accessibility, channel preferences.
  - `/consents/{consent_id}` – consents, authorizations, and privacy flags.
  - `/recommendation/{rec_id}` – Brain next-best-action artifacts.
  - `/risk/{risk_profile_id}` – risk & stratification views.
  - `/attachments/{doc_id}` – PDFs and external documents (benefit books, EMO PDFs, etc.).

---

## 3. Identity & Profile Resources

### 3.1 Member Profile

**Resource**  
- `mcp://twins/member/{member_id}/profile`

**Document Types**

- `MemberTwinProfile`  
  High-level identity & enrollment snapshot:
  - Demographics (name, DOB, gender, address, contact; PHI-tagged).
  - Primary coverage (plan, group, effective dates).
  - PCP & key providers (names + IDs, not full rosters).
  - Program enrollments (e.g., BH program, chronic care, maternity support).
  - High-level flags (e.g., high-utilizer, complex needs, language preference).

- `MemberTwinSummary`  
  Prompt-ready summary:
  - 3–7 bullet “story so far” about the member.
  - Top conditions, top active programs.
  - 3–5 most important open actions.

---

## 4. Coverage & Benefits Resources

### 4.1 Coverage Overview

**Resource**  
- `mcp://twins/member/{member_id}/coverage`

**Document Types**

- `CoverageOverviewSummary`  
  - Primary plan name, group, network.
  - Effective and termination dates.
  - High-level deductible / OOP scoreboard.

### 4.2 Benefits by Topic

**Resources**

- `.../coverage/benefit/{benefit_doc_id}`
- Topic-tagged views (e.g., `.../coverage/benefit/primary-care`, `.../coverage/benefit/therapy`, etc.).

**Document Types**

- `BenefitCoverageSummary` (per topic)
  - Eligibility and copay/coinsurance for:
    - Primary care visits.
    - Specialist visits.
    - Telehealth / virtual visits.
    - Behavioral health therapy & psychiatry.
    - Labs & imaging.
    - Emergency & urgent care.
  - Common limitations (e.g., “X visits per year”, “prior auth required”).
  - Links/refs to full plan docs in `/attachments`.

- `BenefitNetworkSummary`  
  - In-network vs out-of-network rules.
  - PCP selection requirements.

---

## 5. Care Plans, Tasks & Programs

### 5.1 Care Summary & Open Actions

**Resource**  
- `mcp://twins/member/{member_id}/care/summary`

**Document Types**

- `CareSnapshot`  
  - Top conditions + associated plans/programs.
  - Active care plans and key milestones.
  - Open actions (tasks to member, tasks to care team).
  - Recently completed actions and outcomes (high level).

- `OpenActionsList`  
  - Flattened list of open actions:
    - Human-readable label.
    - Owner (member vs care team vs external partner).
    - Status, due date, urgency.
    - References to underlying tasks/plans.

### 5.2 Care Plans

**Resource**

- `.../care/plan/{plan_id}`

**Document Types**

- `CarePlanSummary`  
  - Plan name, goals, owner (IPU/program).
  - Start/end dates.
  - Key goals and metrics.
  - High-level structure of tasks and milestones.

- `CarePlanDetail`  
  - More detailed step-level or phase-level view.
  - Mapping to CareFlow service deliveries/tasks.

### 5.3 Tasks & Service Deliveries

**Resources**

- `.../care/task/{task_id}`
- `.../care/serviceDelivery/{service_delivery_id}`

**Document Types**

- `CareFlowTaskSummary`  
  - Task description, status, due date.
  - Assigned party (member, clinician, care team).
  - Linked plan/goal.

- `ServiceDeliverySummary`  
  - Encounter-like view for CareFlow service deliveries.
  - Actions taken, outcomes, next steps.

### 5.4 Programs & IPUs

**Resource**

- `.../program/{program_enrollment_id}`

**Document Types**

- `ProgramEnrollmentSummary`  
  - Program (IPU) name, type, and focus (e.g., BH, chronic care, maternity).
  - Enrollment status and dates.
  - Key contacts (coach, clinician).
  - Integration points with plans and tasks.

---

## 6. Clinical Summary, Conditions, Medications, Labs, Encounters

> Note: Especially sensitive; many fields will be heavily annotated and restricted.

### 6.1 Clinical Summary

**Resource**

- `mcp://twins/member/{member_id}/clinical/summary`

**Document Types**

- `ClinicalSnapshot`  
  - Top N active conditions.
  - Major procedures/surgeries.
  - Key meds and allergies.
  - High-level vitals trends (if available) or risk markers.

### 6.2 Conditions / Problem List

**Resource**

- `.../clinical/condition/{condition_id}`

**Document Types**

- `ConditionSummary`  
  - Condition name, code (ICD/SNOMED where available).
  - Onset, status (active/resolved), severity.
  - Key related encounters or plans.

### 6.3 Medications

**Resource**

- `.../clinical/medication/{med_id}`

**Document Types**

- `MedicationSummary`  
  - Drug name, class, route, dose, frequency.
  - Status (active/discontinued).
  - Prescriber and last refill.
  - Key warnings or interactions (if available).

### 6.4 Labs & Diagnostics

**Resource**

- `.../clinical/lab/{lab_result_id}`

**Document Types**

- `LabResultSummary`  
  - Test name, date, ordering provider.
  - Result values (highlighting abnormal values only for prompt views).
  - Interpretation or flags.

### 6.5 Encounters & Visits

**Resource**

- `.../encounter/{encounter_id}`

**Document Types**

- `EncounterSummary`  
  - Type (ER visit, primary care, IH virtual visit, BH therapy, etc.).
  - Date/time, location (physical or virtual).
  - High-level reason for visit.
  - Diagnoses and key orders.
  - Follow-up instructions and next steps.

---

## 7. Financial: Claims, EOBs, Cost Estimates

> Likely phased in later; start with light, member-friendly views.

### 7.1 Claims & EOBs

**Resource**

- `.../financial/claim/{claim_id}`

**Document Types**

- `ClaimSummary`  
  - Service date, provider, high-level service category.
  - Billed vs allowed vs paid vs member responsibility.
  - Status (pending, paid, denied).
  - Link to EOB attachment.

- `EOBSummary`  
  - Member-focused explanation in plain language.
  - Highlighted line items that impact member cost.

### 7.2 Cost Estimates

**Resource**

- `.../financial/estimate/{estimate_id}`

**Document Types**

- `CostEstimateSummary`  
  - Procedure/visit type.
  - Estimated range of member responsibility.
  - Assumptions (network status, typical utilization).

---

## 8. Communication, Preferences, Consents

### 8.1 Communication History

**Resource**

- `.../communication/thread/{thread_id}`

**Document Types**

- `CommunicationThreadSummary`  
  - Type (SMS, email, in-app message, call log).
  - Participants (member, IH guides, clinicians).
  - High-level topic.
  - Key decisions or outcomes.

### 8.2 Preferences & Channels

**Resource**

- `.../preferences`

**Document Types**

- `CommunicationPreferences`  
  - Preferred channels (SMS, email, app, phone).
  - Quiet hours / do-not-disturb windows.
  - Language preference.
  - Accessibility needs.

### 8.3 Consents & Authorizations

**Resource**

- `.../consents/{consent_id}`

**Document Types**

- `ConsentSummary`  
  - Type of consent (data sharing, EMO, program enrollment, text messaging).
  - Scope and duration.
  - Status (active, revoked, expired).

---

## 9. Recommendations, Risk & Next Best Actions

### 9.1 Recommendations & Actions

**Resource**

- `.../recommendation/{rec_id}`

**Document Types**

- `RecommendationSnapshot`  
  - Next-best-action from the Brain.
  - Rationale (why recommended).
  - Suggested channel and timing.
  - Links to underlying evidence (care gaps, risk scores, claims history).

### 9.2 Risk Profiles

**Resource**

- `.../risk/{risk_profile_id}`

**Document Types**

- `RiskProfileSummary`  
  - Risk bands (e.g., low/medium/high for cost, utilization, clinical risk).
  - Key drivers (e.g., chronic conditions, recent events).
  - Relationship to active programs and plans.

---

## 10. Attachments & External Documents

### 10.1 Attachments

**Resource**

- `.../attachments/{doc_id}`

**Document Types**

- `AttachmentDescriptor`  
  - Type (benefit booklet, EMO PDF, external report, uploaded document).
  - Source system.
  - Privacy level.
  - Link to download or render.

For prompt use, `search` would typically return **summaries** of attachments with safe snippets, and `readDocument` would either:

- Return a redacted text view, or
- Return a link + metadata for out-of-band human review.

---

## 11. Prompt-Friendly Synthetic Views

Finally, MemberTwin should define some **synthetic, cross-cutting documents** specifically optimized for LLM prompts and human dashboards:

- `GuideHandoffSummary`  
  - Short summary of who the member is, why they’re here, and what’s most urgent now.

- `ClinicianVisitPrepSummary`  
  - Key conditions, meds, recent labs, open actions, and member concerns for an upcoming visit.

- `MemberSelfServiceSummary`  
  - Plain-language summary tuned for member-facing agents: “Here’s what’s going on and what you can do next.”

These synthetic documents would be resolved from resources like:

- `.../care/summary`
- `.../clinical/summary`
- `.../coverage`

and would be first-class Schema Registry types with strong privacy annotations.

---

## 12. Next Steps

- Choose a **P0 slice** for the walking skeleton (e.g., `/profile`, `/care/summary`, `/coverage`, `/care/task/{task_id}`).
- Define concrete Schema Registry types for those slices.
- Implement search/readDocument patterns around these resources.
- Iterate with real agent use cases (Guide Copilot, Member FAQ Agent) to refine which documents are most valuable and what fields they need.

---

## 13. Chat, Digital Session, and Synthetic Views

While most of this catalog focuses on **documents and resources**, the `chat` tool is how agents and frontends actually **instantiate a session** around those documents.

### 13.1 How `chat` Enables Digital Session

At the MemberTwin layer, `chat` is more than just free-form messaging. It is the bridge between:

- The **member’s digital twin** (all the views above), and
- The **Digital Session** concept in the Agent Platform / frontends.

Proposed behavior for `memberTwin.chat`:

- **Session orchestration**
  - If no active session exists for `(member_id, surface)`, `chat` can request a new Digital Session from the Agent Platform.
  - If a session already exists, `chat` attaches to it using a `session_id` parameter.
  - All subsequent `chat` calls for that `session_id` can:
    - Append to the conversation transcript.
    - Link SRs, Tasks, and documents referenced in the conversation.

- **Context linkage**
  - Each `chat` call carries references to the Twin resources in play (e.g., `/care/summary`, `care/task/{task_id}`, `/coverage`).
  - The Agent Platform/Digital Session can store these references so any copilot or human agent attached to the session sees the same context anchors.

- **Synthetic session views**
  - MemberTwin can maintain synthetic documents like `DigitalSessionSynopsis` or `GuideHandoffSummary` that are updated as `chat` interactions occur.
  - These synthetic views become first-class documents that `search` and `readDocument` can retrieve later.

Example `DigitalSessionSynopsis` synthetic view:

```json
{
  "session_id": "DS-abc",
  "member_id": "M123",
  "topics": ["coverage", "MRI scheduling"],
  "referenced_resources": [
    "mcp://twins/member/M123/coverage",
    "mcp://twins/member/M123/care/task/T-321"
  ],
  "summary": "Member is checking coverage before scheduling an MRI; coverage is active, next step is picking an appointment.",
  "last_updated_at": "2025-11-15T16:10:00Z"
}
```

### 13.2 Synthetic Views Driven by Session

`chat` is the natural trigger for updating several synthetic views described earlier:

- **Member 360 Work-in-Progress** – enriched with the current conversation’s goals and outcomes.
- **Guide Handoff Summary** – updated whenever a copilot hands off to a human guide.
- **Member Self-Service Summary** – updated when a self-service flow completes via chat.

By treating these as **documents** behind stable resources (e.g., `.../session/{session_id}/synopsis`, `.../session/{session_id}/handoff`), the MemberTwin keeps sessions observable and reusable across agents and channels.

---

## 14. Memory at the Twin MCP Layer

Memory in this architecture should be **explicit, typed, and shared**, not a hidden, per-LLM scratchpad. The MemberTwin is the right layer to host most of that memory because it already owns identity, context, and privacy.

### 14.1 Memory Primitives

We can think about three main kinds of memory:

1. **Long-term member memory** (lives with the twin)
   - Stable facts: conditions, preferences, consents, long-running programs.
   - Evolving summaries: `MemberTwinSummary`, `ClinicalSnapshot`, `CareSnapshot`.
   - Episode-level artifacts: "knee surgery episode", "pregnancy episode" views.

2. **Medium-term episode/session memory** (bridges sessions)
   - `DigitalSessionSynopsis` and `GuideHandoffSummary` as above.
   - Episode-of-care documents that track a journey over weeks/months (e.g., pregnancy, post-op rehab).

3. **Short-term conversational memory** (within a single session)
   - Recent turns, clarifications, and member goals.
   - Stored as part of the Digital Session transcript, with safe projections available to MemberTwin.

### 14.2 How Memory is Represented

Rather than opaque vector stores, memory should appear as **Schema Registry-backed documents** that can be:

- **Read** via `search` and `readDocument` (e.g., "find the last GuideHandoffSummary for this member").
- **Updated** by Twin logic when:
  - CareFlow events occur.
  - Brain recommendations are emitted.
  - `chat` interactions add new information (e.g., preferred providers, clarified goals).

Example memory-oriented documents:

- `MemberMemorySnapshot` – consolidated view of what we "know" about the member that is especially useful for agents.
- `EpisodeMemorySummary` – memory scoped to a specific episode-of-care.
- `SessionMemorySummary` – distilled memory of a single Digital Session (questions asked, decisions made).

All of these would be:

- Stored in proto-store or another durable store keyed by member_id and context (episode_id, session_id).
- Tagged with privacy annotations, so that projections differ by caller (member-facing vs agent-facing vs clinician-facing).

### 14.3 How LLM Agents Interact with Memory

From an agent’s point of view, memory access is still just `search`/`readDocument`:

- To **retrieve** memory:
  - Call `memberTwin.search` with a query like "session synopsis for this session" or "recent memory about MRI coverage".
  - The twin returns references to synthetic memory documents (`SessionMemorySummary`, `EpisodeMemorySummary`).

- To **update** memory:
  - Agents don’t directly write; instead they send a `chat` message with **structured intents** (e.g., "member_preference.update") that MemberTwin can interpret.
  - MemberTwin may respond by:
    - Creating/updating CareFlow SRs/Tasks.
    - Updating a `MemberMemorySnapshot` or `EpisodeMemorySummary` document.

This separation keeps LLMs from making arbitrary writes while still letting them **suggest** memory updates, which Twin logic then validates, normalizes, and persists.

### 14.4 Why Memory at the Twin Layer Works Well

- **Shared across agents** – Multiple LLM-based agents, frontends, and human users see the same memory artifacts. There is one story, not many.
- **Privacy-aware** – Memory is filtered by schema annotations before it ever leaves the Twin; sensitive fields can be masked or excluded per role.
- **Inspectable and debuggable** – Memory is just documents. Ops, clinicians, and engineers can read them directly, or have a "why does the system think this?" UX.
- **Incrementally upgradable** – We can improve how memory is summarized (better synthetic views, richer features) without changing the external interface.

In combination with `chat` and Digital Session, this makes MemberTwin not just a **snapshot** of the member, but a **living, evolving memory** that all participants in the Sense–Decide–Act loop can rely on.

---

## 15. `memberTwin.chat` – Example Tool Schema

Below is a concrete proposal for the `memberTwin.chat` tool contract as exposed via MCP. This is intentionally high-level and language-agnostic, but it can be translated into JSON Schema or TypeScript interfaces.

### 15.1 Request Shape

**Tool name:** `memberTwin.chat`

```jsonc
{
  "type": "object",
  "properties": {
    "member_resource": {
      "type": "string",
      "description": "MCP URI for the member twin, e.g., mcp://twins/member/{member_id}"
    },
    "session_id": {
      "type": "string",
      "description": "Optional Digital Session ID; if omitted, Twin may request a new session from Agent Platform."
    },
    "surface": {
      "type": "string",
      "description": "Logical surface making the call (member_app, agent_desktop, clinician_console, etc.)."
    },
    "message": {
      "type": "object",
      "description": "The message to append to the session transcript.",
      "properties": {
        "role": {
          "type": "string",
          "enum": ["member", "agent", "assistant", "system"],
          "description": "Who is speaking from the perspective of the Digital Session."
        },
        "text": {
          "type": "string",
          "description": "Natural language message content."
        }
      },
      "required": ["role", "text"]
    },
    "intents": {
      "type": "array",
      "description": "Optional structured intents derived by the LLM, used for memory/work updates.",
      "items": {
        "type": "object",
        "properties": {
          "name": {
            "type": "string",
            "description": "Machine-readable intent name (e.g., member_preference.update)."
          },
          "payload": {
            "type": "object",
            "description": "Intent-specific fields (e.g., preference type, value)."
          }
        },
        "required": ["name"]
      }
    },
    "context_references": {
      "type": "array",
      "description": "Optional list of Twin resource URIs that this chat turn is about.",
      "items": {
        "type": "string",
        "description": "e.g., mcp://twins/member/{id}/coverage, .../care/task/{task_id}"
      }
    }
  },
  "required": ["member_resource", "message"]
}
```

### 15.2 Response Shape

```jsonc
{
  "type": "object",
  "properties": {
    "session_id": {
      "type": "string",
      "description": "The active Digital Session ID (new or existing)."
    },
    "reply": {
      "type": "object",
      "description": "Optional message from the Twin/Agent Platform that the caller may display to the member or agent.",
      "properties": {
        "role": {
          "type": "string",
          "enum": ["assistant", "system", "agent"],
          "description": "Voice of the reply."
        },
        "text": {
          "type": "string",
          "description": "Natural language reply content."
        }
      }
    },
    "linked_resources": {
      "type": "array",
      "description": "Twin resources that this chat turn has created or linked to (SRs, Tasks, documents).",
      "items": {
        "type": "string",
        "description": "e.g., mcp://twins/member/{id}/care/task/{task_id}"
      }
    },
    "updated_synthetic_views": {
      "type": "array",
      "description": "Synthetic view documents that were updated as a result of this chat turn.",
      "items": {
        "type": "object",
        "properties": {
          "type": {
            "type": "string",
            "description": "View type, e.g., DigitalSessionSynopsis, MemberMemorySnapshot."
          },
          "resource": {
            "type": "string",
            "description": "MCP URI for the view, e.g., mcp://twins/member/{id}/session/{session_id}/synopsis."
          }
        },
        "required": ["type", "resource"]
      }
    },
    "side_effects": {
      "type": "array",
      "description": "High-level description of actions taken (for logging/debugging).",
      "items": {
        "type": "object",
        "properties": {
          "kind": {
            "type": "string",
            "description": "e.g., SR_CREATED, TASK_UPDATED, MEMORY_UPDATED."
          },
          "details": {
            "type": "object",
            "description": "Implementation-specific metadata; IDs should reference Twin resources or CareFlow objects."
          }
        },
        "required": ["kind"]
      }
    }
  },
  "required": ["session_id"]
}
```

### 15.3 How Agents Would Use This

- **Open or continue a session**  
  Call `memberTwin.chat` with `member_resource`, an optional `session_id`, the current user/agent message, and any inferred `intents`. Use the returned `session_id` for subsequent turns.

- **Track and reuse context**  
  Look at `linked_resources` and `updated_synthetic_views` to know which documents to pull into future prompts via `search`/`readDocument`.

- **Reason about side effects safely**  
  Use `side_effects` as a human- and machine-readable log of what changed (SRs created, tasks updated, memory updated), without having to inspect raw events.

This keeps `chat` as the *control plane* for interaction, while ensuring all long-term memory and work artifacts remain visible as **documents** under the MemberTwin resource tree.

