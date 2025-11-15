# IH Digital Twin MCP Pattern – Overview & First-Principles Rationale

---

## 1. Concept Overview (One-Pager)

### Problem

Today, building agentic experiences at Included Health requires each team to:

- Understand the full internal topology (CareFlow, Brain / Recommendation Platform, Air Traffic Control, Agent Platform, eligibility/claims partners, EHR connectors, etc.).
- Wire up many small tools and APIs per use case (dozens of gRPC calls, GraphQL queries, Kafka topics, and workflows).
- Re-solve identity, authorization, and privacy questions for every new agent.

Representations of **who** we are helping (member, clinician, employee) are split across many services and data models (FHIR, internal protos, GraphQL types), which:

- Slows time-to-market.
- Increases privacy/compliance risk.
- Makes it harder to deliver coherent, IPU-style care experiences.

### Vision

Create a **Digital Twin MCP pattern** for Included Health: a standard way to represent a person or entity in the IH ecosystem as a **digital twin** behind a small, stable set of MCP tools.

Agents and copilots no longer talk directly to a zoo of tools. Instead, they talk to a **twin** via three simple verbs, inspired by the Glean MCP server:

1. **search** – "Find relevant information for this twin."
2. **readDocument** – "Fetch a specific artifact or record."
3. **chat** – "Engage in a stateful, auditable interaction / session."

Everything else (CareFlow, ATC, Brain, Agent Platform, benefits, eligibility, etc.) is hidden behind the twin’s implementation.

### Core Concept: Digital Twins

A **Digital Twin (IH-Twin)** is a long-lived, typed representation of a real-world actor in IH’s ecosystem:

- **MemberTwin** – a member’s longitudinal record, CareFlow plans/tasks, benefits, preferences, communication history.
- **PractitionerTwin** – clinicians, guides, therapists, nurses, coaches, and other caregivers (aligned with FHIR Practitioner + RelatedPerson).
- **EmployeeTwin** – IH staff represented as FHIR Practitioners in specific operational roles (guide, navigator, engineer-on-call, etc.).
- Future: **PlanTwin**, **FacilityTwin**, **EmployerTwin**, **ProgramTwin**, etc.

Each twin:

- Has schemas registered in **Schema Registry**, mixing FHIR resources and IH domain types (e.g., `MemberTwinProfile`, `CareFlowTaskSummary`, `RecommendationSnapshot`).
- Carries **privacy annotations** (PHI/PII class, purpose-of-use, consent scopes, allowed tools/targets) at the field level.
- Exposes **MCP resources** (for example `mcp://twins/member/{memberId}` and related child resources).
- Implements a standard set of MCP tools:
  - `twins.search`
  - `twins.readDocument`
  - `twins.chat`

### How Agents Use It

Example: An LLM agent helps a member with a care question.

1. The agent is given a **MemberTwin resource handle** for the active member.
2. It calls `twins.search(memberTwin, query)`
   - The twin queries proto-store, CareFlow, Brain, claims, and partner APIs.
   - It merges, de-duplicates, ranks, and returns a concise, typed result set.
3. It calls `twins.readDocument(memberTwin, docRef)` to load structured artifacts:
   - CareFlow form submissions
   - Care plans and tasks
   - EMO reports, visit summaries
   - Benefits & coverage documents
4. It uses `twins.chat(memberTwin, message)` to:
   - Open or attach a **session in Agent Platform**.
   - Route to the right human/virtual agent or workflow.
   - Track state, next-best-actions, and audit trail for the interaction.

The agent never needs to know how many systems, tools, or queues sit behind the twin; it just uses three verbs on one object.

### Architecture & Integration

**MCP Server Layer**

- A set of **Twin MCP servers** (`twin-mcp-member`, `twin-mcp-practitioner`, etc.).
- Each server exposes:
  - Twin instances as MCP resources.
  - The common tool surface: `search`, `readDocument`, and `chat`.

**Schema & Privacy (Schema Registry)**

- All twin types and their views are defined in **Schema Registry**.
- Each field includes privacy annotations: PHI category, retention, allowed tools, allowed destinations.
- MCP middleware enforces "minimum necessary" disclosure by caller, purpose, and tool type.

**IH Domain Integration**

- **CareFlow** – twin composes care plans, tasks, orders, and form data into member- or practitioner-centric views.
- **Recommendation Platform / Brain** – twin fetches or computes Next Best Actions and surfaces them in `search` results and `chat` sessions.
- **Air Traffic Control (ATC)** – twin uses ATC for timing and channel selection when a `chat` interaction leads to outreach.
- **Agent Platform** – `chat` maps to session creation/update, agent routing, and logging.

**Infrastructure Integration (Twin SDK)**

A **Twin SDK** makes it trivial to define and host new twins in separate repos:

- Codegen from Schema Registry to strongly typed models (Go/TS/etc.).
- Helpers for reading/writing canonical data in **proto-store**.
- RPC and GraphQL clients with consistent tracing, retries, and caching.
- Kafka/Temporal helpers for event-driven updates and long-running workflows.
- Built-in observability for twin interactions (metrics, logs, traces).

### Benefits

**For Agent Builders**

- Minimal, stable tool surface: `search`, `readDocument`, `chat`.
- Typed responses modeled in IH’s domain language.
- No need to understand internal topology or reimplement privacy rules.

**For IH**

- Faster time-to-market for new agents and IPU workflows.
- Centralized enforcement of privacy and compliance policies.
- Clean pattern for future digital twins (plans, facilities, employers).

**For Members & Clinicians**

- More coherent, context-aware experiences.
- Fewer "please re-explain everything" moments.
- Safer, more transparent AI assistance.

---

## 2. First Principles: Why the Three-Primitive Twin Interface Works

This section reasons from first principles about **why** the digital twin + `search` / `readDocument` / `chat` pattern is so powerful.

### 2.1 Start with the Real Problem

At the core, we’re trying to solve three intertwined problems:

1. **Understanding** – The system (human or AI agent) needs to understand a member’s or clinician’s situation well enough to help.
2. **Acting** – It needs to take effective actions (or trigger workflows) that change reality in a safe, auditable way.
3. **Trust & Safety** – It must respect privacy, consent, and clinical safety while doing (1) and (2).

Underneath those are constraints:

- IH has many heterogeneous systems (EHRs, CareFlow, Brain, claims, benefits, etc.).
- Agents are often LLM-based and operate through **tool calls**.
- We need something humans and machines can both reason about.

### 2.2 First Principle: Nouns vs. Verbs

All complex systems can be decomposed into **nouns** (entities) and **verbs** (actions on those entities).

- At IH, the most important nouns are people and organizations: members, clinicians, employees, plans, facilities, employers.
- The verbs we need in practice boil down to:
  - **Look around** (what’s going on here?).
  - **Open the file** (show me the exact record or artifact).
  - **Do something / talk to someone** (start or continue an interaction that changes state).

The digital twin pattern encodes this cleanly:

- **Noun:** `MemberTwin`, `PractitionerTwin`, etc. (the entity).
- **Verbs:** `search`, `readDocument`, and `chat`.

Instead of exposing every low-level verb ("getCareFlowTask", "listClaims", "startAgentSession", etc.), we elevate to three universal verbs that cover almost every useful behavior while hiding implementation details.

### 2.3 First Principle: Information Flow for Agents

Any agent trying to help must:

1. **Gather context** from a large, messy universe of data.
2. **Zoom in** on the exact artifact needed to act safely.
3. **Take action** in a way that is reversible, auditable, and human-interpretable.

These map almost exactly to:

1. **`search`** – broad, contextual retrieval.
2. **`readDocument`** – precise retrieval of a specific object.
3. **`chat`** – stateful, high-level interaction that may cause side effects.

From an information-theoretic point of view:

- `search` is a **many-to-few reduction**: it compresses a huge space of possibilities into a small set of candidates.
- `readDocument` is a **precision fetch**: it turns a reference into a structured, typed payload.
- `chat` is a **control channel**: it encodes intent, constraints, and next steps in a way humans can review.

This is exactly how LLM tools are most effective:

- Use `search` to pull in *just enough* context into the prompt.
- Use `readDocument` to avoid hallucinating on critical details (e.g., benefit limits, medication dosages, authorization notes).
- Use `chat` to negotiate and coordinate actions with humans and backend systems.

### 2.4 First Principle: Boundary of Responsibility

A good system design defines **clear responsibility boundaries**:

- The **twin** is responsible for knowing how to:
  - Locate and normalize data about its entity.
  - Enforce privacy rules on that data.
  - Invoke downstream systems (CareFlow, Brain, ATC, etc.) safely.
- The **agent** is responsible for:
  - Reasoning about goals.
  - Decomposing problems.
  - Choosing which verb to use next.

By collapsing many internal tools into three high-level verbs, we:

- Make it easier to prove that the twin enforces privacy and policy (fewer, more stable surfaces).
- Let agents remain agnostic to system topology.
- Keep implementation churn localized inside the twin instead of in every agent.

### 2.5 First Principle: Composability and Minimal Interfaces

Computer science gives us a rule of thumb: **small, orthogonal primitives** compose better than large, overlapping ones.

- `search` and `readDocument` are orthogonal:
  - `search` doesn’t promise stable identity or full fidelity; it promises *relevance*.
  - `readDocument` doesn’t promise relevance; it promises *correctness for a specific reference*.
- `chat` is the side-effectful channel, separate from pure retrieval.

This separation yields several advantages:

1. **Composability**
   - Agents can chain `search -> readDocument -> chat` in flexible ways.
   - Other tools (like summarizers or planners) can wrap these calls without caring about system internals.

2. **Optimizable Implementation**
   - `search` can evolve from a simple keyword search to vector search to hybrid RAG without changing the agent interface.
   - `readDocument` can introduce caching, partial hydration, or field-level masking behind the scenes.
   - `chat` can switch from one orchestration engine to another (Agent Platform v1/v2, new routing strategies) without touching agents.

3. **Testability & Safety**
   - Each verb can have its own policy checks, rate limits, and tests.
   - Side effects are located almost entirely in `chat`, making it easier to audit.

### 2.6 First Principle: Privacy & Schema as Gravity

IH needs to treat **schema + privacy annotations** as a source of truth.

- Without a twin, every tool must re-encode which fields are PHI, which can be used for what, and when masking is required.
- With a twin backed by **Schema Registry**:
  - Schemas define the shape of each view.
  - Privacy annotations define what may flow through `search`, `readDocument`, and `chat`.

In other words, schema and annotation act as **gravity**:

- Data naturally flows into twin views.
- Only allowed subsets of that data can flow out through MCP tools.
- Changes to privacy policy are made once, at the schema/annotation layer, and enforced automatically at runtime.

This makes it much more realistic to:

- Build many agents quickly.
- Still meet regulatory and contractual obligations.
- Provide a coherent story to Security, Legal, and Compliance.

### 2.7 First Principle: Human Mental Model

The pattern also matches how humans already think about getting help:

- **"Can you look this up for me?" → `search`**
- **"Can you pull up that document?" → `readDocument`**
- **"Can we talk about what to do next?" → `chat`**

By aligning system primitives with natural language primitives:

- Product teams can design experiences that map directly to these verbs.
- Care teams can understand what an agent is doing without reading code.
- It becomes easier to debug: "What did this agent *search*? What did it *read*? What did it *say* or *trigger* in `chat`?"

### 2.8 First Principle: Scaling Across Domains

Finally, we need a pattern that scales horizontally:

- New twin types (Plan, Facility, Employer) should not require a redesign of every agent.
- New domains within IH should plug into the same mental and technical model.

The digital twin + three-verb interface gives us that:

- Each new twin has:
  - Its own schemas in Schema Registry.
  - Its own internal integrations (proto-store, RPC, GraphQL, etc.).
  - The same external shape: `search`, `readDocument`, `chat`.

This keeps the **agent integration problem O(1)**:

- Agents learn *one* interface.
- IH can add *many* twins behind it.

### 2.9 First Principle: Context as the Most Critical Resource (and Why Delegation Helps)

In LLM-based systems, **context** is the scarcest and most expensive resource:

- The context window is finite.
- Tokens are costly (latency, dollars, carbon, and cognitive complexity).
- Most raw data is either **irrelevant** to the current question or **duplicative**.

Without a twin, every agent tends to:

- Pull large swaths of raw data (entire visit histories, full benefit books, long task lists).
- Re-derive the same member or clinician understanding repeatedly.
- Bloat prompts with details that rarely matter, increasing hallucination risk and cost.

The digital twin pattern **delegates context management** to the twin instead of the LLM:

1. **Context by Delegation, Not by Bulk**
   - The agent asks the twin: "search for what matters" rather than pulling everything.
   - The twin performs heavy lifting server-side (joins, filters, rankers) and returns **compact, typed summaries** instead of raw tables and logs.
   - This keeps the LLM’s prompt small while still semantically rich.

2. **Context by Reference, Not by Payload**
   - `search` results come back with **stable references** (e.g., `documentId`, `careFlowTaskId`, `benefitDocRef`).
   - The agent can **refer** to these IDs over multiple turns and only call `readDocument` when it truly needs full fidelity.
   - This avoids re-sending entire documents in the prompt when a short label or reference is enough.

3. **Context Layers**
   - **Twin Layer:** holds full-fidelity data, historical state, and pre-computed features (e.g., risk scores, open care gaps, program enrollment).
   - **LLM Layer:** sees only the current question, a small working set of summaries, and any documents it explicitly requested.
   - **UI / Human Layer:** can see both (for debugging / supervision), but the default is: 
     - broader context lives in the twin,
     - focused context lives in the agent’s prompt.

4. **Reuse and Stability Across Agents**
   - The same MemberTwin can serve multiple agents (member-facing, clinician-facing, internal operations) with **consistent context projections**.
   - Each agent sends small, targeted `search` requests; the twin reuses its understanding of the member’s state instead of every agent recomputing it.

5. **Cost and Safety**
   - Smaller prompts → lower cost and latency.
   - Less irrelevant data → fewer opportunities for the model to latch onto the wrong detail.
   - Explicit `search` / `readDocument` calls with references → easier to log and audit *what* context the model actually saw.

In short, the twin turns context into a **managed, delegated service** instead of something every agent bolts together ad hoc. The LLM is no longer responsible for:

- Remembering everything about the member or clinician.
- Manually filtering large, noisy datasets.

Instead, it is responsible for **asking good questions of the twin**. That shift—moving context management from the LLM’s prompt into a reusable, typed, privacy-aware twin—is what makes this pattern scale.

### 2.10 First Principle: Polymorphism – One Interface, Many Twins and Backends

There is also a strong **polymorphism** story baked into this pattern: the same small interface is implemented by many different kinds of twins and many different backend integrations.

1. **One Interface for Many Twin Types**
   - All twins – Member, Practitioner, Employee, Plan, Facility, etc. – implement the same MCP tool surface: `search`, `readDocument`, and `chat`.
   - Agents can therefore treat twins **generically**:
     - "Given *any* `TwinHandle`, call `search` with this query."
     - "Given a `documentRef` from *any* twin, call `readDocument`."
   - The differences (member vs clinician vs plan) show up in:
     - The **schema types** returned (e.g., `MemberTwinProfile` vs `PractitionerTwinProfile`).
     - The **resource URIs** (e.g., `mcp://twins/member/...` vs `mcp://twins/practitioner/...`).
   - But the *control plane* – how you call the tools – is uniform.

2. **One Interface for Many Backends**
   - Under the hood, different twins may:
     - Talk to different EHRs, eligibility providers, or employer systems.
     - Use different data stores (proto-store projections, GraphQL subgraphs, specialized indexes).
     - Apply different ranking, summarization, or feature pipelines.
   - None of that leaks into the agent layer. The MCP interface is the same across implementations.
   - This means:
     - You can swap or evolve backends (e.g., move Plan data from one vendor to another) without touching agents.
     - You can pilot new backends behind the same twin contract using configuration or routing rules.

3. **Reducing Integration Fan-Out**
   - Without polymorphic twins, each agent typically needs custom integrations to each system it cares about.
   - With polymorphic twins:
     - Agents integrate once with the **Twin MCP interface**.
     - New twin types – or new backend systems – plug into that interface.
   - Integration work becomes **O(number of twin types)** instead of **O(number of agents × number of systems)**.

4. **Stronger Guarantees, Shared Tooling**
   - Because the interface is uniform, you can:
     - Reuse the same **authorization and privacy middleware** across all twins.
     - Reuse the same **observability, rate limiting, and testing harnesses**.
     - Share client SDKs in multiple languages without per-domain special cases.
   - Polymorphism means "if it’s a twin, these guarantees and tools apply" – regardless of what’s behind it.

5. **Future-Proofing**
   - As new domains or external partners come online, they can be wrapped in a new twin implementation that conforms to the same interface.
   - Agents and orchestrators can start using these new twins immediately, often without any code changes – just new configuration or resource URIs.

In practice, this polymorphism is what allows the digital twin pattern to scale horizontally across IH’s entire ecosystem while keeping integration costs and cognitive load flat for agent builders.

---

## 3. Amazon-Style Press Release (Internal Draft)

**FOR IMMEDIATE RELEASE**

**Included Health Introduces Digital Twin MCP Platform to Power Next-Generation Healthcare Agents**

*New “twin” pattern gives AI agents and care teams a single, privacy-aware interface to members, clinicians, and employees.*

**San Francisco, CA – [TBD Date]** – Included Health today announced the **Digital Twin MCP Platform**, a new internal pattern and service that lets teams represent every person in the care ecosystem—members, clinicians, and employees—as a **digital twin** accessible through a simple Model Context Protocol (MCP) interface.

Instead of integrating with dozens of tools and APIs, AI agents and internal applications will be able to interact with a single digital twin using just three capabilities: **search**, **read document**, and **chat**. Behind the scenes, the twin orchestrates complex workflows across CareFlow, the Recommendation Platform (“The Brain”), Air Traffic Control, and the Agent Platform.

> “Our mission is to make healthcare more personal, proactive, and connected,” said the VP of Engineering at Included Health. “The Digital Twin MCP Platform gives us a consistent, safe way for both humans and AI agents to understand and act on a member’s whole story, without exposing them to the complexity of our underlying systems.”

### A Single Interface to a Complex Healthcare Universe

The **Digital Twin MCP Platform** launches with three primary twin types:

- **MemberTwin** – a unified, longitudinal view of a member’s care journey, tasks, benefits, preferences, and communication history.
- **PractitionerTwin** – a view of clinicians, guides, therapists, and other caregivers, modeled on FHIR Practitioner concepts.
- **EmployeeTwin** – a representation of internal staff in their operational roles, also mapped to FHIR Practitioner semantics.

Each twin is exposed through an MCP server as virtual resources (for example, `mcp://twins/member/{id}`) and accessed with three high-level tools:

1. **Search** – contextual search across CareFlow tasks, visit summaries, recommendations, EMO reports, and benefits.
2. **Read Document** – structured retrieval of specific artifacts such as care plans, CareFlow forms, coverage documents, and claim snapshots.
3. **Chat** – a stateful, auditable conversation channel that can open or attach to sessions in the Agent Platform, route to the right human or automated workflow, and keep track of next-best actions.

Inspired by the simplicity of the Glean MCP server, this pattern allows agent builders to stay focused on behavior and experience, rather than on wiring up dozens of internal tools.

### Built-In Privacy, Powered by Schema Registry

The Digital Twin MCP Platform uses **Included Health’s Schema Registry** as its source of truth for types and privacy rules:

- All twin schemas are centrally defined and versioned, from `MemberTwinProfile` and `CareFlowTaskSummary` to `RecommendationSnapshot` and `BenefitCoverageSummary`.
- Each field carries **privacy annotations** (PHI/PII category, purpose-of-use, allowed tools, allowed destinations).
- The MCP layer automatically masks or omits fields based on the caller, tool, and use case, enforcing a “minimum necessary” data philosophy.

This approach lets teams ship new agents and use cases quickly while maintaining strict alignment with regulatory, contractual, and internal privacy requirements.

### Integrated with CareFlow, The Brain, and Agent Platform

The Digital Twin MCP Platform is designed to sit on top of Included Health’s existing clinical and operational platforms:

- **CareFlow** – the engine for care plans, tasks, and forms that the twin surfaces to agents.
- **Recommendation Platform / Brain** – computes member- and clinician-specific next-best actions that are exposed via the twin.
- **Air Traffic Control** – coordinates outreach and engagement timing, invoked by twin workflows.
- **Agent Platform** – connects members and clinicians to the right human or automated endpoint; `chat` tools open and manage these sessions through the twin.

For engineers, a **Twin SDK** provides first-class integration with Included Health infrastructure, including proto-store, RPC clients, GraphQL subgraphs, event streams, and observability.

> “For care teams and members, the experience should feel like you’re talking to one intelligent, coordinated system,” said the Chief Medical Officer at Included Health. “Digital twins help make that possible—safely and at scale.”

### A Pattern, Not Just a Platform

Beyond the initial Member, Practitioner, and Employee twins, the Digital Twin MCP Platform defines a reusable **pattern** for future twins such as Plan, Facility, Employer, and Program:

- Teams can add new twin types in their own repositories using the shared SDK and Schema Registry.
- Every twin automatically benefits from the same privacy, observability, and integration standards.
- Agents see a consistent interface across all twins, accelerating innovation and reducing integration cost.

### Availability

The **MemberTwin MCP server** and **Twin SDK** will be piloted with selected Agent Platform and CareFlow-integrated use cases, with Practitioner and Employee twins to follow. Over time, Included Health plans to make the twin pattern the default way new internal agents and copilots interact with people-centric data.

For more information about the Digital Twin MCP Platform and how to integrate it into your product area, please contact the Engineering Platform and Agent Platform teams or visit the internal documentation hub.


---

## 5. MemberTwin Amazon-Style Press Release (Internal Draft)

**FOR IMMEDIATE RELEASE**

**Included Health Launches MemberTwin: A Digital Twin MCP Service for Every Member**

*New Digital Twin service gives agents and care teams a single, privacy-aware interface to understand and act on each member’s health journey.*

**San Francisco, CA – [TBD Date]** – Included Health today announced the launch of **MemberTwin**, a new Digital Twin MCP service that creates a unified, privacy-aware digital representation of each member and exposes it through a simple, three-verb interface: **search**, **read document**, and **chat**.

MemberTwin is the first concrete realization of the company’s 2025 Tech Vision for a **sense–decide–act** flywheel, where MemberTwin serves as the **sense** layer, the Recommendation Platform ("The Brain") is **decide**, and CareFlow plus the Agent Platform represent **act**.

> “Our members shouldn’t have to repeat their story to every product and every agent,” said the VP of Engineering at Included Health. “MemberTwin gives us one place where that story lives, and one interface our agents can use to understand and act on it safely.”

### A Single Interface for Member Context

MemberTwin represents each member as a long-lived digital twin, backed by Included Health’s Schema Registry and privacy annotations. Instead of integrating with dozens of APIs, AI agents and applications talk to the twin using three tools:

1. **Search** – context-aware retrieval of what matters now: open actions, recent care, key conditions, and relevant benefits.
2. **Read Document** – precise, typed retrieval of source-of-truth artifacts such as CareFlow tasks, visit summaries, and benefits documents.
3. **Chat** – a stateful conversation channel that can open or attach to sessions in the Agent Platform and coordinate real-world actions.

This pattern allows agent builders to keep their code simple and expressive while MemberTwin handles the hard parts: data integration, privacy enforcement, and backend orchestration.

### Context Management by Delegation

In LLM-based agents, context is the most critical and scarce resource. MemberTwin is designed to **manage context by delegation**, not by pushing more raw data into the model’s prompt.

- **Server-side reasoning about relevance** – MemberTwin performs joins, filters, and ranking across CareFlow, conditions, and benefits, returning compact summaries instead of raw tables.
- **Context by reference** – search results include stable document references that can be fetched on demand via `readDocument`, reducing prompt size and repetition.
- **Layered context** – full-fidelity data lives in MemberTwin; the agent sees only the working set it explicitly asked for, and humans can inspect both for supervision and debugging.

This makes agents faster, cheaper, and safer: less irrelevant information in the prompt, fewer opportunities for error, and a clear audit trail of which context the model actually used.

### Polymorphic Design for Future Twins

MemberTwin also establishes the **polymorphic interface** that future twins will follow.

- All Digital Twins (Member, Practitioner, Employee, Plan, Facility, Employer) implement the same MCP tool surface.
- Each twin provides its own schemas and backend integrations, but the way agents call them is identical.
- This dramatically reduces integration fan-out: agents integrate once with the Twin MCP interface and can immediately leverage new twins as they are introduced.

> “We want to build agents that understand people, not systems,” said the Chief Medical Officer at Included Health. “MemberTwin’s design lets us extend that understanding to clinicians, facilities, and plans without rewriting every integration.”

### Built-In Privacy and Compliance

MemberTwin uses Included Health’s Schema Registry to define both data shapes and privacy rules:

- Field-level annotations specify PHI/PII categories, purpose-of-use, and allowed destinations.
- The MCP layer automatically masks or omits fields based on the caller, the tool (`search`, `readDocument`, `chat`), and the use case.
- All calls are logged for auditing, with a clear record of what information was surfaced to which agent for what purpose.

This gives Security, Legal, and Compliance a single, enforceable choke point for member-centric context, instead of dozens of custom integrations.

### Integrated with CareFlow, The Brain, and Agent Platform

From day one, MemberTwin is wired into core Included Health platforms:

- **CareFlow** – source of truth for care plans, tasks, forms, and orders, exposed via MemberTwin views.
- **Recommendation Platform / Brain** – computes member-specific next-best actions that MemberTwin surfaces in search results and chat sessions.
- **Agent Platform** – `chat` tools open and manage member sessions, route to the right human or virtual agent, and track state.

For engineers, a **Twin SDK** provides first-class integration with proto-store, RPC, GraphQL subgraphs, and observability to extend MemberTwin’s capabilities over time.

### Availability

MemberTwin will initially roll out as a **walking skeleton** behind selected Agent Platform and CareFlow-integrated experiences, focusing on a narrow but meaningful slice of member context: open actions, recent care, and common benefits questions.

As the service matures, MemberTwin will expand to cover more clinical, behavioral, and financial dimensions of a member’s journey and will be joined by additional twins for practitioners, employees, plans, facilities, and employers.

For more information about MemberTwin and how to participate in the pilot, please reach out to the Engineering Platform and Agent Platform teams or visit the internal Digital Twin MCP documentation hub.

