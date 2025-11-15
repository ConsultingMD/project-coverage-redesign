# Digital Twin MCP Documentation

## 🎯 Quick Start

The Digital Twin MCP implements the "Sense" layer of IH's 2025 Tech Vision, providing a unified interface to member data through the Model Context Protocol.

**Core Concept**: Replace 100+ agent tools with 3 universal verbs accessing unified resources.

### Start Here Based on Your Role

| If you are... | Start with... | Then read... |
|---------------|---------------|--------------|
| **Executive/Product** | [Executive Summary](./MCP_RESOURCES_AND_EVENTS_SUMMARY.md) | [Migration Benefits](./AGENT_PLATFORM_DIGITAL_TWIN_MIGRATION.md#simplification-metrics) |
| **Architect** | [Core Pattern](./DIGITAL_TWIN_MCP_PATTERN.md) | [Master Guide](./DIGITAL_TWIN_MCP_MASTER.md) |
| **Developer** | [MVP POC](./DIGITAL_TWIN_MCP_MVP_POC.md) | [Resource Catalog](./MEMBER_TWIN_RESOURCES_CATALOG.md) |
| **Security** | [Security Best Practices](./MCP_SECURITY_BEST_PRACTICES_IH.md) | [Authorization](./MCP_AUTHORIZATION_AUTHZILLA_INTEGRATION.md) |

---

## 📚 Complete Documentation Set

### Core Architecture (Start Here)
1. **[DIGITAL_TWIN_MCP_PATTERN.md](./DIGITAL_TWIN_MCP_PATTERN.md)**
   - First principles design
   - Three-verb pattern (search, readDocument, chat)
   - Integration with 2025 Tech Vision

2. **[MEMBER_TWIN_RESOURCES_CATALOG.md](./MEMBER_TWIN_RESOURCES_CATALOG.md)**
   - Complete resource definitions
   - URI structure and schemas
   - Memory and synthetic views

### Integration Guides
3. **[DIGITAL_TWIN_MCP_RESOURCES_INTEGRATION.md](./DIGITAL_TWIN_MCP_RESOURCES_INTEGRATION.md)**
   - Event-driven architecture integration
   - Kafka topics and WebSocket delivery
   - Real-time updates

4. **[MCP_AUTHORIZATION_AUTHZILLA_INTEGRATION.md](./MCP_AUTHORIZATION_AUTHZILLA_INTEGRATION.md)**
   - OAuth 2.1 with Authzilla
   - Authzed (SpiceDB) fine-grained permissions
   - Token management

5. **[MCP_SECURITY_BEST_PRACTICES_IH.md](./MCP_SECURITY_BEST_PRACTICES_IH.md)**
   - Security requirements
   - Token passthrough prevention
   - Session management

6. **[MCP_SAMPLING_ELICITATION_PATTERNS.md](./MCP_SAMPLING_ELICITATION_PATTERNS.md)**
   - Intelligent resource selection
   - Conversational information gathering
   - Clinical use cases

### Implementation
7. **[DIGITAL_TWIN_MCP_MVP_POC.md](./DIGITAL_TWIN_MCP_MVP_POC.md)**
   - 3-week POC plan
   - Golang implementation
   - Test with Claude Desktop

8. **[AGENT_PLATFORM_DIGITAL_TWIN_MIGRATION.md](./AGENT_PLATFORM_DIGITAL_TWIN_MIGRATION.md)**
   - 28-week migration plan
   - Tool mapping (100+ → 3)
   - Addresses Walmart Coverage ADR

### Reference
9. **[MCP_RESOURCES_AND_EVENTS_SUMMARY.md](./MCP_RESOURCES_AND_EVENTS_SUMMARY.md)**
   - Executive summary
   - Three-layer architecture
   - Key benefits

10. **[DIGITAL_TWIN_MCP_MASTER.md](./DIGITAL_TWIN_MCP_MASTER.md)**
    - Document hierarchy
    - Terminology guide
    - Consistency standards

---

## 🚀 Implementation Roadmap

### Phase 1: POC (3 weeks)
- Build minimal MCP server
- 3 resources (profile, coverage, care)
- 2 tools (search, readDocument)
- Test with Claude Desktop
- **Deliverable**: Working POC proving pattern

### Phase 2: MVP (8 weeks)
- Production MCP server
- Full authorization (Authzilla + Authzed)
- Event integration (Kafka)
- 15-20 core resources
- **Deliverable**: Production-ready Digital Twin

### Phase 3: Migration (28 weeks)
- Migrate Agent Platform tools
- Migrate ai-workflows (Walmart)
- Deprecate old tools
- **Deliverable**: 80% code reduction

---

## 🏗️ Architecture Overview

```mermaid
graph LR
    subgraph "Before: 100+ Tools"
        T1[get_coverage_answers]
        T2[find_provider]
        T3[get_claims_status]
        T4[...97 more tools]
    end
    
    subgraph "After: 3 Universal Tools"
        SEARCH[memberTwin.search]
        READ[memberTwin.readDocument]
        CHAT[memberTwin.chat]
    end
    
    subgraph "Unified Resources"
        R1[mcp://twins/member/{id}/profile]
        R2[mcp://twins/member/{id}/coverage]
        R3[mcp://twins/member/{id}/care]
    end
    
    T1 --> SEARCH
    T2 --> SEARCH
    T3 --> READ
    T4 --> CHAT
    
    SEARCH --> R1
    READ --> R2
    CHAT --> R3
```

---

## 💡 Key Concepts

### The Three Verbs
1. **search**: Find resources by query
2. **readDocument**: Get specific resource
3. **chat**: Conversational interface with routing

### Resource URIs
```
mcp://twins/member/{id}/coverage
mcp://twins/member/{id}/profile
mcp://twins/member/{id}/care/tasks
```

### Tool Naming Convention
```yaml
memberTwin.search       # ✅ Dot notation
memberTwin.readDocument # ✅ Consistent prefix
memberTwin.chat        # ✅ Standard pattern
```

---

## 📊 Impact Metrics

| Metric | Before | After | Improvement |
|--------|--------|-------|-------------|
| **Number of Tools** | 100+ | 3 | 97% reduction |
| **Code Lines** | ~50,000 | ~10,000 | 80% reduction |
| **Integration Points** | 100+ APIs | 1 MCP server | 99% reduction |
| **Onboarding Time** | 2-4 weeks | 2-3 days | 85% faster |
| **Maintenance Burden** | High | Low | Dramatic reduction |

---

## 🔗 External References

### MCP Specification
- [MCP Overview](https://modelcontextprotocol.io)
- [MCP Resources](https://modelcontextprotocol.io/specification/2025-06-18/server/resources)
- [MCP Authorization](https://modelcontextprotocol.io/specification/2025-06-18/basic/authorization)
- [MCP Security](https://modelcontextprotocol.io/specification/2025-06-18/basic/security_best_practices)

### IH Systems
- [Agent Platform](https://github.com/ConsultingMD/agent-platform)
- [ai-workflows](https://github.com/ConsultingMD/ai-workflows)
- [Proto-common](https://github.com/ConsultingMD/proto-common)
- [Walmart Coverage ADR](https://github.com/ConsultingMD/ai-workflows/blob/main/docs/adr/0002_walmart-coverage-info.md)

---

## 📝 Document Status

| Document | Status | Last Updated | Owner |
|----------|--------|--------------|-------|
| Core Pattern | ✅ Complete | 2025-01-20 | Platform Team |
| Resource Catalog | ✅ Complete | 2025-01-20 | Platform Team |
| Event Integration | ✅ Complete | 2025-01-20 | Platform Team |
| Authorization | ✅ Complete | 2025-01-20 | Security Team |
| Security | ✅ Complete | 2025-01-20 | Security Team |
| Sampling | ✅ Complete | 2025-01-20 | AI Team |
| MVP POC | ✅ Complete | 2025-01-20 | Tiger Team |
| Migration Plan | ✅ Complete | 2025-01-20 | Tiger Team |

---

## 🤝 Contributing

To maintain consistency:
1. Follow terminology in [DIGITAL_TWIN_MCP_MASTER.md](./DIGITAL_TWIN_MCP_MASTER.md)
2. Use standard tool names (dots, not underscores)
3. Reference the 2025 Tech Vision
4. Update cross-references when adding documents

---

## ❓ FAQ

**Q: Why "Digital Twin MCP" not "MemberTwin MCP"?**
A: Digital Twin is the platform supporting multiple twin types (member, practitioner, etc.). MemberTwin is one implementation.

**Q: How does this relate to the 2025 Tech Vision?**
A: Digital Twin MCP is the "Sense" layer, providing real-time member state to The Brain ("Decide") and CareFlow ("Act").

**Q: What about the Walmart Coverage ADR?**
A: Perfect example of the problem we're solving. Currently requires porting tools between systems. With Digital Twin MCP, ai-workflows just uses standard tools.

**Q: When can we start using this?**
A: POC in 3 weeks, MVP in 8 weeks, full migration over 28 weeks.

---

## 📞 Contact

- **Platform Team**: For architecture questions
- **Tiger Team**: For migration planning
- **Security Team**: For authorization/security
- **#digital-twin-mcp** Slack channel
