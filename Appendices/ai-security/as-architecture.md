# Architecture Overview

Complete architectural overview of the AI Security Backend system.

## Table of Contents


# Appendix F-1.3 — Architecture

```mermaid
graph TD
    subgraph External["External Systems"]
        Wazuh[Wazuh Agent]
        Windows[Windows Events]
        IIS[IIS Logs]
        Other[Other Sources]
    end

    subgraph Database["PostgreSQL"]
        NormLogs[(Normalized Logs<br/>JSONB Data)]
    end

    subgraph Backend["AI Security Backend"]
        subgraph Listener["Log Listener"]
            Realtime[Realtime<br/>LISTEN/NOTIFY]
            Polling[Polling<br/>Query]
            AIOnly[AI-Only<br/>Mode]
        end

        Grouping[Log Grouping<br/>- Source IP<br/>- User/Host<br/>- Time Window]

        Pipeline[Detection Pipeline<br/>- Retry Logic<br/>- Error Handling<br/>- Metrics Tracking]

        subgraph Orchestrator["Detection Orchestrator"]
            subgraph RuleBased["Rule-Based Detection"]
                RE[1. RuleEngine<br/>- Evaluate YAML rules<br/>- Pattern matching<br/>- Aggregations]
                DA[2. DetectionAgent AI<br/>- Validate matches<br/>- Reduce false positives<br/>- Assign severity]
                AA[3. AdvisorAgent AI<br/>- Generate remediation<br/>- Map to MITRE/OWASP<br/>- Actionable steps]
                QA[4. QualityAgent AI<br/>- Check feedback history<br/>- Filter false positives<br/>- Final validation]
                RE --> DA --> AA --> QA
            end

            subgraph AIOnlyFlow["AI-Only Detection"]
                DA2[1. DetectionAgent AI<br/>- Analyze without rules<br/>- Identify threats]
                AA2[2. AdvisorAgent AI<br/>- Generate remediation]
                QA2[3. QualityAgent AI<br/>- Validate and filter]
                DA2 --> AA2 --> QA2
            end
        end

        SaveDB[(Save Detection<br/>to Database)]
        Broadcaster[Event Broadcaster<br/>- Manage WS clients<br/>- Broadcast events<br/>- Filter subscriptions]
    end

    subgraph MCP["Model Context Protocol"]
        MCP1[Detection MCP Server<br/>Port 3100<br/>Tools: getLogs, MITRE, OWASP]
        MCP2[Advisor MCP Server<br/>Port 3101<br/>Tools: getLogs, MITRE, OWASP, searchRem]
        MCP3[Quality MCP Server<br/>Port 3102<br/>Tools: getLogs, MITRE, OWASP, patterns]
    end

    subgraph Clients["External Clients"]
        API[REST API<br/>/api/detections<br/>/api/logs<br/>/api/stats<br/>etc.]
        WS[WebSocket<br/>/ws/detections<br/>Real-time events]
    end

    Wazuh & Windows & IIS & Other --> Ingestion
    Ingestion --> NormLogs
    NormLogs --> Listener
    Listener --> Grouping
    Grouping --> Pipeline
    Pipeline --> Orchestrator
    Orchestrator --> SaveDB
    SaveDB --> Broadcaster
    Broadcaster --> API
    Broadcaster --> WS

    DA -.uses.-> MCP1
    AA -.uses.-> MCP2
    QA -.uses.-> MCP3
```

## Core Processing Flow

### 1. Log Ingestion

```mermaid
graph LR
    PG[(PostgreSQL<br/>New Log)] --> LL{LogListener<br/>Mode?}

    LL -->|Realtime| RT[PostgreSQL<br/>LISTEN/NOTIFY<br/>Instant trigger]
    LL -->|Polling| PL[Periodic Query<br/>SELECT WHERE<br/>processed_at IS NULL]
    LL -->|AI-Only| AI[Polling +<br/>Skip RuleEngine]

    RT --> Check
    PL --> Check
    AI --> Check

    Check{Grouping<br/>Enabled?} -->|Yes| Group[Group Logs<br/>- Match by fields<br/>- Time window<br/>- Include historical]
    Check -->|No| LogArray[Single Log Array]

    Group --> LogArray


```

### 2. Detection Pipeline

```mermaid
graph TD
    Start[Pipeline Start<br/>Retry logic<br/>Error tracking] --> Counter{Attempt<br/>Counter}
    Counter --> Process[Process Logs<br/>via Orchestrator]
    Process -->|Success| Continue[Continue<br/>Update metrics]
    Process -->|Failure| Retry{Attempts < 3?}
    Retry -->|Yes| Wait[Wait<br/>delay × attempt]
    Wait --> Counter
    Retry -->|No| Error[Log Error<br/>Update metrics<br/>Throw]
```

### 3. Detection Workflow (Rule-Based)

```mermaid
graph TD
    RE[RuleEngine] --> Applies{Rule<br/>Applies?}
    Applies -->|No| Skip1[Skip Rule]
    Applies -->|Yes| Eval[Evaluate Conditions<br/>AND/OR/Nested]

    Eval --> Match{Match?}
    Match -->|No| Skip2[Skip Rule]
    Match -->|Yes| DA[DetectionAgent AI<br/>Analyze rule match<br/>with context]

    DA --> Real{Is Real<br/>Threat?}
    Real -->|No| Discard1[Discard<br/>False positive]
    Real -->|Yes| Output1[Output:<br/>Detection Result<br/>title, description<br/>severity, confidence]

    Output1 --> AA[AdvisorAgent AI<br/>Generate plan<br/>Tools: MITRE, OWASP]

    AA --> Output2[Output:<br/>Remediation Plan<br/>summary, steps<br/>references]

    Output2 --> QA[QualityAgent AI<br/>Load feedback patterns<br/>Compare detection]

    QA --> Save{Should<br/>Save?}
    Save -->|No| Discard2[Discard<br/>Matches FP pattern]
    Save -->|Yes| Output3[Output:<br/>Quality Validation<br/>shouldSave: true<br/>confidence, reasoning]

    Output3 --> DB[(Save to DB<br/>Table: detections<br/>Link: detection_logs)]

    DB --> Broadcast[Broadcast via WS<br/>Event: detection.created]
```

### 4. Detection Workflow (AI-Only)

```mermaid
graph TD
    DA[DetectionAgent AI<br/>No rule matching] --> Analyze[Analyze logs<br/>Open-ended analysis<br/>No predefined patterns]

    Analyze --> Identify[Identify Patterns<br/>AI determines threats]

    Identify --> Multiple[Output:<br/>Multiple Detections<br/>Each with details]

    Multiple --> Loop[For each detection...]
    Loop --> AA[AdvisorAgent AI]
    AA --> QA[QualityAgent AI]
    QA --> Save[(Save to DB)]
    Save --> Broadcast[Broadcast via WS]
```

## Component Diagram

```mermaid
graph TD
    App[SecurityApp<br/>Application lifecycle<br/>Graceful shutdown<br/>Metrics logging]

    App --> API[API Server<br/>Fastify]
    App --> Listener[LogListener]
    App --> Orch[Orchestrator]

    Listener --> Pipeline[DetectionPipeline]
    Orch --> Pipeline

    API --> WS[WebSocket<br/>EventBroadcaster]

    Pipeline --> RE[RuleEngine<br/>+ AI Agents]

    RE --> Repos[(Database<br/>Repositories)]
```

### Component Relationships

| Component | Depends On | Used By |
|-----------|------------|---------|
| SecurityApp | All components | None (entry point) |
| LogListener | Pipeline, LogRepository | SecurityApp |
| DetectionPipeline | DetectionOrchestrator | LogListener |
| DetectionOrchestrator | RuleEngine, Agents, Repositories | DetectionPipeline |
| RuleEngine | RuleLoader | DetectionOrchestrator |
| AI Agents | Codex SDK, MCP Servers | DetectionOrchestrator |
| MCP Servers | Repositories, Utils | AI Agents (via Codex) |
| API Server | Repositories, Orchestrator, EventBroadcaster | Frontend, External clients |
| EventBroadcaster | WebSocket connections | DetectionOrchestrator, API Server |

## Data Flow

### Log → Detection Flow

```mermaid
sequenceDiagram
    participant Ext as External Source
    participant PG as PostgreSQL
    participant LL as LogListener
    participant DP as DetectionPipeline
    participant DO as DetectionOrchestrator
    participant DB as Database
    participant EB as EventBroadcaster
    participant WS as WebSocket Clients

    Ext->>PG: Send Log

    alt Realtime Mode
        PG-->>LL: NOTIFY new_log_inserted
    else Polling Mode
        LL->>PG: SELECT unprocessed
        PG-->>LL: Return logs
    end

    LL->>DP: processLogs(logs)
    DP->>DO: processLogs(logs)
    DO->>DO: RuleEngine + AI Agents
    DO->>DB: Save Detection
    DB-->>DO: Detection saved
    DO->>EB: broadcastDetection()
    EB->>WS: Send detection event
```

### Feedback → Learning Flow

```mermaid
sequenceDiagram
    participant User as Frontend User
    participant API as REST API
    participant FR as FeedbackRepository
    participant FS as FeedbackService
    participant FP as FeedbackPatternRepo
    participant QA as QualityAgent

    User->>API: POST /api/feedback<br/>isHelpful: false
    API->>FR: create(feedback)
    FR->>FR: Save to feedbacks table

    FR->>FS: aggregatePatterns()
    FS->>FS: Group by rule_id<br/>Calculate confidence
    FS->>FP: Save patterns

    Note over QA: Next detection...
    QA->>FP: getFeedbackPatterns()
    FP-->>QA: Return patterns
    QA->>QA: Compare detection<br/>Filter if matches FP
```

## Technology Stack

### Core Technologies

```mermaid
graph LR
    subgraph Runtime
        Node[Node.js 18+]
        TS[TypeScript 5.x<br/>ES Modules]
        NPM[npm<br/>Workspace Monorepo]
    end

    subgraph Web
        Fastify[Fastify 5.x<br/>High Performance]
        WS[WebSocket @fastify/websocket]
    end

    subgraph Data
        PG[PostgreSQL 12+]
        Seq[Sequelize 6.x<br/>ORM]
        Driver[pg driver<br/>node-postgres]
    end

    subgraph AI
        Codex[Codex SDK<br/>@cme/codex-sdk]
        MCP[MCP SDK<br/>Tool definitions]
        Providers[OpenAI, Azure<br/>Mistral, Ollama]
    end

    subgraph Config
        Env[dotenv<br/>Environment]
        YAML[yaml<br/>Config files]
        Zod[Zod<br/>Validation]
    end

    subgraph DevTools
        Jest[Jest 29.x<br/>Testing]
        tsup[tsup<br/>Build]
        tsx[tsx<br/>Dev Server]
        Winston[Winston 3.x<br/>Logging]
    end
```

- **Runtime**: Node.js 18+
- **Language**: TypeScript 5.x (ES Modules)
- **Package Manager**: npm (workspace-based monorepo)
- **API**: Fastify 5.x (high performance, schema validation)
- **WebSocket**: @fastify/websocket (real-time communication)
- **Database**: PostgreSQL 12+ with Sequelize 6.x ORM
- **AI & ML**: Codex SDK with MCP integration (OpenAI, Azure, Ollama)
- **Configuration**: dotenv + YAML
- **Validation**: Zod schemas
- **Logging**: Winston 3.x
- **Testing**: Jest 29.x with ES Modules
- **Build Tools**: tsup (compiler), tsx (dev server)

## Layer Architecture

```mermaid
graph TD
    subgraph Presentation["Presentation Layer"]
        Routes[REST API Routes]
        WSHandlers[WebSocket Handlers]
        Validation[Request Validation]
        Response[Response Formatting]
    end

    subgraph Application["Application Layer"]
        Controllers[Controllers]
        Services[Services]
        BusinessLogic[Business Logic]
        Orchestration[Orchestration]
    end

    subgraph Domain["Domain Layer"]
        Agents[AI Agents]
        RuleEngine[Rule Engine]
        Pipeline[Detection Pipeline]
        Broadcasting[Event Broadcasting]
    end

    subgraph Infrastructure["Infrastructure Layer"]
        Database[Database<br/>Sequelize]
        Repositories[Repositories]
        ExternalAPIs[External APIs<br/>Codex SDK]
        MCPServers[MCP Servers]
        Config[Configuration]
        Logging[Logging]
    end

    Presentation --> Application
    Application --> Domain
    Domain --> Infrastructure
```

## Design Decisions

### 1. Hybrid Rule-Based + AI Approach

**Decision**: Combine YAML rules with AI validation

**Rationale**:
- Rules provide fast, deterministic detection
- AI reduces false positives
- AI adapts to context
- Best of both worlds

### 2. Codex SDK + MCP

**Decision**: Use Codex SDK with MCP for AI agents

**Rationale**:
- Tool use capabilities
- Model abstraction
- Workspace isolation
- Structured outputs

### 3. Log Grouping

**Decision**: Group related logs before analysis

**Rationale**:
- Detect attack chains
- Provide context to AI
- Identify patterns (brute force, lateral movement)
- More accurate detections

### 4. Quality Agent

**Decision**: Add quality validation step with feedback

**Rationale**:
- Learn from user feedback
- Reduce false positives over time
- Continuous improvement
- Pattern recognition

### 5. Polling Default Mode

**Decision**: Default to polling instead of realtime

**Rationale**:
- More reliable
- Simpler deployment
- No persistent connection
- Easier debugging

**Alternative**: Realtime available for low-latency needs

### 6. Monorepo Structure

**Decision**: Use npm workspaces for monorepo

**Rationale**:
- Share dependencies
- Unified versioning
- Easier development
- Codex SDK integration

### 7. Fastify over Express

**Decision**: Use Fastify for API server

**Rationale**:
- Higher performance
- Built-in schema validation
- Better TypeScript support
- Modern async/await support

---

For more details:
- [Core Components](./core-components.md)
- [AI Agents](./ai-agents.md)
- [Rule System](./rule-system.md)
- [Database](./database.md)
