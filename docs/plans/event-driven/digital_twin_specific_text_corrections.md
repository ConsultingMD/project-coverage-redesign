# Digital Twin MCP Pattern - Specific Text Corrections

## Document: `ih_digital_twin_mcp_pattern_first_principles.md`

### Line 7-8: Problem Statement
**Current**:
> Today, building agentic experiences at Included Health requires each team to:

**Suggested Addition After**:
> Today, building agentic experiences at Included Health requires each team to:
> 
> [Note: This challenge is recognized in the 2025 Tech Vision, which defines the Digital Twin Platform as the solution for unified member representation.]

### Line 22: Vision Section
**Current**:
> Create a **Digital Twin MCP pattern** for Included Health: a standard way to represent...

**Corrected**:
> Implement an **MCP interface for the Digital Twin Platform** defined in IH's 2025 Tech Vision: a standard way to access...

### Line 90-95: Architecture & Integration
**Add New Subsection Before "MCP Server Layer"**:
```markdown
**Relationship to 2025 Tech Vision**

The Digital Twin MCP pattern implements the interface layer for the Digital Twin Platform, 
which is already defined as the "Sense" component in IH's official 2025 Sense-Decide-Act 
architecture. This MCP interface provides the standard way for agents and applications 
to interact with the Digital Twin's real-time member model.
```

### Line 424: Amazon Press Release Date
**Current**:
> **San Francisco, CA – [TBD Date]**

**Suggested**:
> **San Francisco, CA – [Implementation Q1-Q2 2025]**

### Line 426-428: Press Release Context
**Current**:
> Instead of integrating with dozens of tools and APIs, AI agents and internal applications will be able to interact...

**Corrected**:
> Building on the Digital Twin Platform defined in our 2025 Tech Vision, agents and internal applications will interact through a unified MCP interface...

---

## Document: `member_twin_resources_documents_draft_catalog.md`

### Line 454: MemberTwin Press Release
**Current**:
> **Included Health Launches MemberTwin: A Digital Twin MCP Service for Every Member**

**Corrected**:
> **Included Health Launches MemberTwin MCP Interface: Unified Agent Access to the Digital Twin Platform**

### Line 458-460: Opening Context
**Current**:
> MemberTwin is the first concrete realization of the company's 2025 Tech Vision...

**Corrected**:
> MemberTwin MCP is the interface implementation for the Digital Twin Platform, which serves as the "Sense" layer in the company's 2025 Tech Vision...

### Line 92: Service Requests Creation
**Current**:
> - Service requests can be created in various platforms such as Hub, Care App, Athena, or Jarvis.

**Keep As-Is** ✅ (This is accurate - Hub is Salesforce, Jarvis is being phased out but still used)

### Line 346: Proto-store Reference
**Add Clarification**:
```markdown
- Stored in proto-store or another durable store keyed by member_id and context
  [Note: Proto-store integrates with Schema Registry for type management and 
   Kafka for event streaming. Other persistence layers may include domain-specific stores.]
```

### Line 508: Availability Section
**Current**:
> MemberTwin will initially roll out as a **walking skeleton**...

**Suggested**:
> The MemberTwin MCP interface will initially roll out as a **walking skeleton** 
> connecting to the Digital Twin Platform infrastructure...

---

## Critical Context to Add in Both Documents

### 1. Existing MCP Infrastructure Section
Add after introduction:
```markdown
## Leveraging Existing MCP Infrastructure

Included Health already has a growing MCP ecosystem that this pattern builds upon:

- **Omnibus MCP**: Demonstrates the search/readDocument pattern for documentation, 
  serving as the model for the Digital Twin's interface
- **MCProxy**: Provides unified access to multiple MCP servers, which could 
  aggregate Digital Twin servers across domains
- **GitHub MCP**: Shows integration patterns for external systems
- **Service MCPs**: Configured via go-common, providing patterns for 
  service-specific MCP implementations

The Digital Twin MCP interface follows these established patterns while 
extending them for person-centric, privacy-aware data access.
```

### 2. Integration with Digital Session Platform
Add to chat tool description:
```markdown
## Integration with Digital Session Platform

The `memberTwin.chat` tool integrates with IH's existing Digital Session Platform, which:
- Represents any member interaction (app sessions, chat, voice, video, clinical encounters)
- Produces real-time event streams that frontends can subscribe to
- Tracks service requests being processed, tasks completed, and care plans updating

When `chat` creates or attaches to a session, it leverages this existing infrastructure
rather than creating new session management capabilities.
```

### 3. Clarify Sense-Decide-Act Components
Add prominent callout box:
```markdown
> **Official 2025 Tech Vision Components**
> 
> - **Sense**: Digital Twin Platform (real-time member model) - THIS MCP INTERFACE
> - **Decide**: The Brain / Recommendation Platform (NBA generation)
> - **Act**: CareFlow + Agent Platform (workflow execution)
> 
> This document defines how agents interact with the Sense layer through MCP tools.
```

---

## Summary of Key Changes

1. **Reframe from "creating" to "implementing interface for"** the Digital Twin
2. **Acknowledge existing MCP servers** (Omnibus, MCProxy, GitHub)
3. **Clarify Digital Session Platform** integration vs. creation
4. **Add explicit 2025 Tech Vision** alignment statements
5. **Keep accurate system references** (Hub, Jarvis, CareFlow, etc.)
6. **Add context for proto-store** and storage layers

These corrections ensure the documents accurately represent IH's current state while properly positioning the MCP interface as an implementation detail of the already-approved Digital Twin Platform.
