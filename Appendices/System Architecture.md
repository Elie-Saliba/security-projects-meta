# Appendix A - System Architecture Diagrams

This appendix provides the complete set of architectural diagrams referenced throughout the project. They summarise the modular design, two-tier microservice structure, and end-to-end execution flows. All diagrams are expressed in Mermaid to remain consistent with the architectural documentation maintained in the Log-Transformer and AI-Security repositories.

---

## A.1 High-Level System Architecture

The system is composed of two independent yet integrated services:

- **Log-Transformer (.NET 8)**: Handles ingestion, parsing, and normalisation of heterogeneous logs.
- **AI-Security Backend (Node.js / TypeScript)**: Performs hybrid detection that combines rule-based logic with AI-driven contextual analysis.

Both services share a PostgreSQL database and publish real-time detections to connected dashboards.

**Figure A.1 - High-Level System Architecture**

```mermaid
graph TB
    subgraph Sources["Log Sources"]
        direction LR
        EVTX["Windows EVTX"]
        SYS["Syslog"]
        WZ["Wazuh"]
        FLB["FluentBit"]
        JSON["JSON Logs"]
    end

    subgraph LT["Log-Transformer (.NET 8)"]
        direction LR
        LTAPI["Ingestion API"]
        Parser["Parser Engine"]
        Normaliser["Normalisation Pipeline"]
        LTPersist["PostgreSQL Writer"]
    end

    subgraph DB["PostgreSQL Database"]
        direction LR
        Norm[("normalized_logs")]
        Det[("detections")]
        FP[("feedback_patterns")]
    end

    subgraph AS["AI-Security Backend (Node.js)"]
        direction LR
        Listener["LogListener"]
        Pipeline["DetectionPipeline"]
        Orchestrator["DetectionOrchestrator"]
        RE["RuleEngine"]
        Agents["AI Agents<br/>Detection / Advisor / Quality"]
        Broadcaster["WebSocket Broadcaster"]
    end

    Sources --> LTAPI
    LTAPI --> Parser --> Normaliser --> LTPersist
    LTPersist --> Norm
    Norm --> Listener
    Listener --> Pipeline --> Orchestrator
    Orchestrator --> RE
    Orchestrator --> Agents
    Orchestrator --> Det
    Orchestrator --> FP
    Orchestrator --> Broadcaster
    Det --> Broadcaster
```

---

## A.2 Ingestion Pipeline Architecture (Log-Transformer)

This diagram represents the flow from raw log input to normalised storage. It is sourced from LT-3-architecture.md, LT-5-core-components.md, and LT-8-end-to-end-flow.md.

**Figure A.2 - Log-Transformer Ingestion Architecture**

```mermaid
flowchart TD
    A[Log Input<br/>EVTX, Syslog, Wazuh, JSON] --> B[Ingestion API]
    B --> C[Job Queue]
    C --> D[Worker Service]
    D --> E[Parser Selection<br/>via Plugin Registry]
    E --> F[Normalized Record Builder]
    F --> G[Batch Writer]
    G --> H[(PostgreSQL<br/>normalized_logs)]
```

---

## A.3 Detection Pipeline Architecture (AI-Security Backend)

Derived from as-core-components.md and as-architecture.md, this pipeline handles log polling, grouping, rule matching, AI contextual reasoning, and final detection creation.

**Figure A.3 - Detection Pipeline (Rule + AI Hybrid)**

```mermaid
sequenceDiagram
    participant PG as PostgreSQL
    participant LL as LogListener
    participant DP as DetectionPipeline
    participant DO as DetectionOrchestrator
    participant RE as RuleEngine
    participant AG as AI Agents
    participant DB as DB
    participant WS as WebSocket Clients

    PG-->>LL: New or unprocessed logs
    LL->>DP: processLogs(logs)
    DP->>DO: processLogs(logs)

    alt Rule-Based Flow
        DO->>RE: EvaluateRules()
        RE-->>DO: Rule Match / No Match
    end

    DO->>AG: DetectionAgent + AdvisorAgent + QualityAgent
    AG-->>DO: Contextual Evaluation

    DO->>DB: Save Detection
    DB-->>DO: Confirmation

    DO->>WS: Broadcast Detection Event
```

---

## A.4 AI Agent Collaboration Architecture (MCP Integration)

This diagram reflects the MCP-based tool orchestration across DetectionAgent, AdvisorAgent, and QualityAgent, sourced from as-mcp-integration.md and as-ai-agents.md.

**Figure A.4 - AI Agent Collaboration via MCP**

```mermaid
graph LR
    subgraph Agents["AI Agents"]
        DA["DetectionAgent"]
        AA["AdvisorAgent"]
        QA["QualityAgent"]
    end

    subgraph MCP["Model Context Protocol Servers"]
        MCP_D["Detection MCP<br/>getLogs, MITRE, OWASP"]
        MCP_A["Advisor MCP<br/>context + remediation"]
        MCP_Q["Quality MCP<br/>patterns + false positives"]
    end

    subgraph DB["PostgreSQL"]
        LOGS["normalized_logs"]
        DET["detections"]
        PAT["feedback_patterns"]
    end

    DA -- uses --> MCP_D
    AA -- uses --> MCP_A
    QA -- uses --> MCP_Q

    MCP_D --> LOGS
    MCP_A --> LOGS
    MCP_Q --> PAT
    QA --> PAT
    AA --> DET
```

---

## A.5 End-to-End Log-to-Detection Flow

This diagram captures the comprehensive pipeline from ingestion through detection and analyst notification.

**Figure A.5 - End-to-End System Flow**

```mermaid
flowchart LR
    A[Raw Log<br/>Windows/Syslog/Wazuh] --> B[Ingestion API]
    B --> C[Parser Engine]
    C --> D[Normalized Record]
    D --> E[(PostgreSQL)]
    E --> F[LogListener]
    F --> G[DetectionPipeline]
    G --> H[Rule Engine]
    H --> I[AI Agents]
    I --> J[Save Detection<br/>Database]
    J --> K[WebSocket<br/>Broadcast]
    K --> L[Analyst Dashboard]
```
