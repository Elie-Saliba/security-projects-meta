# Appendix F-2.1 — Log-Transformer Global README

# Global Documentation Navigation Guide

**AI-Powered Security Intelligence Platform**  
**Complete System Documentation Index**

This guide provides a chronological reading path for AI tools and developers to understand the complete security log analysis platform consisting of two integrated systems:

1. **Log-Transformer** - Log ingestion and normalization service (.NET 8.0)
2. **AI-Security** - AI-powered threat detection and analysis backend (Node.js/TypeScript)

**How They Work Together**: Log-Transformer ingests raw security logs from multiple sources (EVTX, Syslog, Wazuh, etc.), parses them through source-specific parsers, and normalizes them into a unified format in the shared PostgreSQL database. AI-Security then reads these normalized logs, evaluates them against security rules, validates threats using AI agents, generates remediation plans with MITRE ATT&CK mappings, and exposes real-time detections via REST API and WebSocket.

## System Overview

```mermaid
graph TB
    subgraph Sources["Log Sources"]
        EVTX[Windows EVTX]
        SYSLOG[Syslog]
        FB[FluentBit]
        WAZUH[Wazuh]
        JSON[JSON Logs]
    end
    
    subgraph LT["Log-Transformer (.NET)"]
        API[Upload API :5001]
        PARSER[Parser Plugins]
        NORM[Normalizer]
    end
    
    subgraph DB["Shared Database"]
        PG[(PostgreSQL<br/>normalized_logs)]
    end
    
    subgraph AS["AI-Security (Node.js)"]
        LISTEN[Log Listener]
        RULES[Rule Engine]
        DETECT[Detection Agent]
        ADVISOR[Advisor Agent]
        QUALITY[Quality Agent]
        WEBAPP[Web UI :8080]
    end
    
    EVTX --> API
    SYSLOG --> API
    FB --> API
    WAZUH --> API
    JSON --> API
    
    API --> PARSER
    PARSER --> NORM
    NORM --> PG
    
    PG --> LISTEN
    LISTEN --> RULES
    RULES --> DETECT
    DETECT --> ADVISOR
    ADVISOR --> QUALITY
    QUALITY --> PG
    
    PG --> WEBAPP
    
    style LT fill:#e1f5ff
    style AS fill:#ffe1f5
    style DB fill:#e1ffe1
```

## Documentation Reading Order

### Phase 1: Docker Deployment (Start Here)

If you're deploying via Docker, read these first:

1. **[../ai-security/DOCKER-GETTING-STARTED.md](../../ai-security/DOCKER-GETTING-STARTED.md)**
   - Deploy AI-Security platform first (database + backend + frontend)
   - ~15 minutes to complete
   - Prerequisites: Docker, 8GB RAM
   - **Deploy this first** - provides shared PostgreSQL database

2. **[../DOCKER-GETTING-STARTED.md](../DOCKER-GETTING-STARTED.md)**
   - Deploy Log-Transformer API (connects to AI-Security database)
   - ~5 minutes to complete
   - Prerequisites: AI-Security running
   - **Deploy this second** - enables log ingestion

---

### Phase 2: Understanding Log-Transformer (Ingestion Layer)

Read Log-Transformer documentation to understand how logs are collected and normalized:

3. **[LT-README.md](./LT-README.md)**
   - Log-Transformer overview and navigation
   - Service capabilities and features
   - Quick start guide

4. **[LT-1-getting-started.md](./LT-1-getting-started.md)**
   - Prerequisites and installation
   - Configuration setup
   - First log upload
   - Common commands

5. **[LT-3-architecture.md](./LT-3-architecture.md)**
   - System architecture overview
   - Parser plugin system design
   - Data flow from upload to normalization
   - Component interactions

6. **[LT-5-core-components.md](./LT-5-core-components.md)**
   - LogController and upload API
   - Parser plugin interface (ILogIngestParser)
   - Available parsers (EVTX, Syslog, JSON, etc.)
   - IngestWorker background processing
   - Normalization engine

7. **[LT-6-database.md](./LT-6-database.md)**
   - Shared database schema (normalized_logs table)
   - NormalizedLog model structure
   - JSONB field patterns
   - Database context and connections

8. **[LT-7-scalability-and-extensibility.md](./LT-7-scalability-and-extensibility.md)**
   - Adding new log source parsers
   - Horizontal scaling patterns
   - Performance optimization
   - Real-world integration examples (FluentBit, Wazuh, Syslog forwarders)

9. **[LT-2-api-reference.md](./LT-2-api-reference.md)**
   - Upload API endpoints
   - Request/response schemas
   - Error handling
   - Integration examples

10. **[LT-4-configuration.md](./LT-4-configuration.md)**
    - appsettings.json configuration
    - Connection strings
    - Storage settings
    - Import worker configuration

11. **[LT-8-end-to-end-flow.md](./LT-8-end-to-end-flow.md)**
    - Complete flow from log upload to database storage
    - Sequence diagrams
    - State transitions
    - Error handling paths

---

### Phase 3: Understanding AI-Security (Analysis Layer)

Read AI-Security documentation to understand threat detection and AI analysis:

12. **[../../ai-security/documentation/as-README.md](../../ai-security/documentation/as-README.md)**
    - AI-Security backend overview
    - System capabilities and features
    - Navigation hub for all AI-Security docs

13. **[../../ai-security/documentation/as-getting-started.md](../../ai-security/documentation/as-getting-started.md)**
    - Prerequisites and installation
    - Environment configuration
    - Database setup and migrations
    - First run and verification

14. **[../../ai-security/documentation/as-architecture.md](../../ai-security/documentation/as-architecture.md)**
    - Hybrid detection architecture (rule-based + AI)
    - Three detection modes (realtime, polling, ai-only)
    - Component interactions
    - Data flow through detection pipeline
    - Design decisions and rationale

15. **[../../ai-security/documentation/as-core-components.md](../../ai-security/documentation/as-core-components.md)**
    - SecurityApp (application orchestrator)
    - LogListener (realtime/polling log monitoring)
    - DetectionPipeline (retry logic and metrics)
    - DetectionOrchestrator (coordinates detection flow)
    - RuleEngine (YAML rule evaluation)
    - EventBroadcaster (WebSocket broadcasting)

16. **[../../ai-security/documentation/as-rule-system.md](../../ai-security/documentation/as-rule-system.md)**
    - YAML rule structure and schema
    - Condition operators (equals, matches, count, sum, etc.)
    - MITRE ATT&CK and OWASP mappings
    - Creating custom rules
    - Rule examples (brute force, SQL injection, privilege escalation)
    - Best practices

17. **[../../ai-security/documentation/as-ai-agents.md](../../ai-security/documentation/as-ai-agents.md)**
    - BaseAgent (Codex SDK integration)
    - DetectionAgent (threat validation and identification)
    - AdvisorAgent (remediation plan generation)
    - QualityAgent (false positive filtering)
    - Prompt system
    - MCP tool integration

18. **[../../ai-security/documentation/as-mcp-integration.md](../../ai-security/documentation/as-mcp-integration.md)**
    - Model Context Protocol (MCP) overview
    - MCP servers (Detection :3100, Advisor :3101, Quality :3102)
    - Available tools per agent
    - Session management
    - Creating custom tools
    - Testing and debugging

19. **[../../ai-security/documentation/as-database.md](../../ai-security/documentation/as-database.md)**
    - Complete schema (normalized_logs, detections, feedbacks, feedback_patterns)
    - Models and relationships
    - Repositories and data access patterns
    - JSONB query examples
    - Migrations

20. **[../../ai-security/documentation/as-configuration.md](../../ai-security/documentation/as-configuration.md)**
    - Environment variables (.env)
    - config.yaml structure
    - Detection mode configuration
    - AI provider setup (OpenAI, Azure, Ollama, Custom)
    - MCP server configuration
    - Logging and monitoring

21. **[../../ai-security/documentation/as-api-reference.md](../../ai-security/documentation/as-api-reference.md)**
    - REST API endpoints (detections, logs, feedback, stats)
    - Request/response schemas
    - Error handling
    - Authentication considerations

22. **[../../ai-security/documentation/as-websocket.md](../../ai-security/documentation/as-websocket.md)**
    - Real-time detection streaming
    - WebSocket connection and events
    - Event structure (detection.created, detection.updated)
    - Client implementation examples
    - Event filtering
    - EventBroadcaster architecture

---

## Reading Paths by Role

### For System Administrators

**Goal**: Deploy and maintain the platform

1. Docker Deployment: DOCKER-GETTING-STARTED.md (both projects)
2. Configuration: LT-4-configuration.md, as-configuration.md
3. Troubleshooting: Sections in Docker guides
4. Monitoring: as-configuration.md (logging sections)

### For Developers Extending Log-Transformer

**Goal**: Add new log source parsers

1. LT-README.md
2. LT-3-architecture.md (parser plugin system)
3. LT-5-core-components.md (ILogIngestParser interface)
4. LT-7-scalability-and-extensibility.md (adding parsers)
5. LT-6-database.md (normalization schema)

### For Developers Extending AI-Security

**Goal**: Add custom rules or modify AI agents

1. as-README.md
2. as-architecture.md
3. as-rule-system.md (custom rules)
4. as-ai-agents.md (agent behavior)
5. as-mcp-integration.md (adding tools)
6. as-core-components.md (DetectionOrchestrator, RuleEngine)

### For Security Analysts

**Goal**: Understand detections and create rules

1. as-README.md
2. as-architecture.md (detection flow)
3. as-rule-system.md (rule structure and examples)
4. as-ai-agents.md (how AI validates threats)
5. as-api-reference.md (accessing detections)
6. as-websocket.md (real-time alerts)

### For AI Tool Agents

**Goal**: Comprehensive understanding for troubleshooting and extending

**Read in this exact order:**

1. Docker deployment guides (both)
2. All LT- prefixed files (3, 4, 5, 6, 7, 8, 2, 1 order for architecture-first)
3. All as- prefixed files (12-22 in order listed above)

This provides complete context from log ingestion → normalization → detection → AI analysis → remediation.

---

## Key Integration Points

### 1. Shared Database

Both systems use the same PostgreSQL database:

- **Table**: `normalized_logs`
- **Connection**: Log-Transformer writes, AI-Security reads
- **Schema**: Defined in `LT-6-database.md` and `as-database.md`

### 2. Normalized Log Format

Standard format for all log sources:

```typescript
{
  id: uuid,
  source: string,              // "windows", "wazuh", "syslog", etc.
  event_type: string,          // "failed_login", "file_access", etc.
  raw_data: jsonb,             // Original log data
  normalized_data: jsonb,      // Standardized fields
  timestamp: datetime,
  processed_at: datetime,
  status: string
}
```

### 3. Detection Flow

```
Log Source → Log-Transformer (parse & normalize) 
          → PostgreSQL (normalized_logs) 
          → AI-Security (detect & analyze)
          → PostgreSQL (detections)
          → Web UI / API / WebSocket
```

### 4. Network Communication

- **Log-Transformer → Database**: Direct PostgreSQL connection
- **AI-Security → Database**: Sequelize ORM connection
- **Frontend → Backend**: REST API + WebSocket
- **Backend → AI Providers**: Via Codex SDK

---

## Environment Setup Summary

### Log-Transformer (.NET 8.0)

```bash
Port: 5001
Database: Host=ai-security-db;Port=5432;Database=Security
Tech: ASP.NET Core, Entity Framework Core, Docker
```

### AI-Security (Node.js 18+)

```bash
Port: 3000 (API), 8080 (Frontend), 3100-3102 (MCP)
Database: postgresql://postgres:TestDb123@database:5432/Security
Tech: Fastify, Sequelize, Codex SDK, React, Docker
```

---

## Documentation Maintenance

### File Naming Convention

- **AI-Security files**: Prefix with `as-`
  - Example: `as-architecture.md`, `as-api-reference.md`
  
- **Log-Transformer files**: Prefix with `LT-`
  - Example: `LT-3-architecture.md`, `LT-5-core-components.md`

### Adding New Documentation

1. Create file with appropriate prefix
2. Add entry to relevant README (LT-README.md or as-README.md)
3. Update this GlobalReadme.md in appropriate section
4. Cross-reference related documents

### Documentation Standards

- Use Mermaid diagrams for architecture
- Include code examples with syntax highlighting
- Provide complete configuration samples
- Add troubleshooting sections
- Reference related documents
- Keep examples up-to-date with codebase

---

## Quick Reference Table

| Topic | Log-Transformer | AI-Security |
|-------|----------------|-------------|
| **Overview** | LT-README.md | as-README.md |
| **Getting Started** | LT-1-getting-started.md | as-getting-started.md |
| **Architecture** | LT-3-architecture.md | as-architecture.md |
| **API Reference** | LT-2-api-reference.md | as-api-reference.md |
| **Configuration** | LT-4-configuration.md | as-configuration.md |
| **Core Components** | LT-5-core-components.md | as-core-components.md |
| **Database** | LT-6-database.md | as-database.md |
| **Docker Deployment** | ../DOCKER-GETTING-STARTED.md | ../../ai-security/DOCKER-GETTING-STARTED.md |
| **Extensibility** | LT-7-scalability-and-extensibility.md | as-ai-agents.md, as-mcp-integration.md |
| **Rules** | N/A | as-rule-system.md |
| **Real-time Events** | N/A | as-websocket.md |
| **End-to-End Flow** | LT-8-end-to-end-flow.md | as-architecture.md (detection flow) |

---

## Support and Resources

### GitHub Repositories

- **AI-Security**: [GitHub Repository](https://github.com/Elie-Saliba/AI-log-anaomaly-detection)
- **Log-Transformer**: [Local Path](../)

### External References

- **MITRE ATT&CK**: https://attack.mitre.org/
- **OWASP Top 10**: https://owasp.org/www-project-top-ten/
- **Codex SDK**: https://github.com/dmitry-brazhenko/codex-sdk
- **Model Context Protocol**: https://modelcontextprotocol.io/

### Contact

- **Issues**: Create GitHub issue in AI-log-anaomaly-detection repository
- **Documentation**: `documentation/` directories in both projects

---

**Documentation Version**: 1.0.0  
**Last Updated**: November 7, 2025  
**Systems Covered**: Log-Transformer v1.0 + AI-Security v0.1.0
