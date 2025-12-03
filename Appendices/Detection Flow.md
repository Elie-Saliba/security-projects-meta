# Appendix E - Detection Flow Diagrams

This appendix documents the detection flows implemented in the AI-Security backend. Each Mermaid diagram illustrates how normalised logs move from ingestion to detection, highlighting rule-only, hybrid, and AI-only modes plus the feedback loop and status lifecycle.

---

## E.1 End-to-End Detection Lifecycle

Shows the journey from new logs in PostgreSQL to detections broadcast to analyst dashboards.

**Figure E.1 - End-to-End Detection Lifecycle**

```mermaid
sequenceDiagram
    participant PG as PostgreSQL<br/>(normalized_logs)
    participant LL as LogListener
    participant DP as DetectionPipeline
    participant DO as DetectionOrchestrator
    participant RE as RuleEngine
    participant AG as AI Agents
    participant DETDB as PostgreSQL<br/>(detections)
    participant WS as WebSocket Clients

    PG-->>LL: Poll or LISTEN for new logs
    LL->>DP: deliver(logBatch)
    DP->>DO: processLogs(logBatch)

    alt Rule match / hybrid mode
        DO->>RE: evaluateRules(logBatch)
        RE-->>DO: ruleMatches[]
    else AI-only mode
        DO->>AG: DetectionAgent(logBatch)
        AG-->>DO: aiDetections[]
    end

    DO->>AG: enrichWithContext(detections)
    AG-->>DO: enrichedDetections
    DO->>DETDB: saveDetections(enrichedDetections)
    DETDB-->>DO: detectionIds[]
    DO->>WS: broadcast(detection.created)
```

---

## E.2 Rule-Only Detection Flow

Covers the pure rule-processing path when AI enrichment is disabled.

**Figure E.2 - Rule-Only Detection Flow**

```mermaid
flowchart LR
    A[(normalized_logs)] --> B[LogListener]
    B --> C[DetectionPipeline]
    C --> D[DetectionOrchestrator]
    D --> E[RuleEngine<br/>Evaluate rules]

    E -->|no match| F[No detection created]
    E -->|match| G[Create Detection Entity]
    G --> H[(detections)]
    H --> I[WebSocket Broadcaster]
    I --> J[Analyst Dashboard]
```

---

## E.3 Hybrid Detection Flow (Rules + AI Agents)

Illustrates how rule hits are validated, contextualised, and filtered by AI agents before persistence.

**Figure E.3 - Hybrid Detection Flow**

```mermaid
sequenceDiagram
    participant PG as PostgreSQL<br/>(normalized_logs)
    participant LL as LogListener
    participant DO as DetectionOrchestrator
    participant RE as RuleEngine
    participant DA as DetectionAgent
    participant AA as AdvisorAgent
    participant QA as QualityAgent
    participant DETDB as PostgreSQL<br/>(detections)
    participant WS as WebSocket Clients

    PG-->>LL: provide(logBatch)
    LL->>DO: process(logBatch)

    DO->>RE: evaluateRules(logBatch)
    RE-->>DO: ruleMatches[]

    DO->>DA: validateAndScore(ruleMatches)
    DA-->>DO: scoredDetections

    DO->>AA: buildRecommendations(scoredDetections)
    AA-->>DO: recommendations

    DO->>QA: applyFeedbackPatterns(scoredDetections)
    QA-->>DO: filteredDetections

    DO->>DETDB: save(filteredDetections)
    DETDB-->>DO: detectionIds[]
    DO->>WS: broadcast(detection.created)
```

---

## E.4 AI-Only Detection Flow

Describes the pathway when detections are driven solely by AI agents.

**Figure E.4 - AI-Only Detection Flow**

```mermaid
flowchart TD
    A[(normalized_logs)] --> B[LogListener<br/>fetch candidate logs]
    B --> C[DetectionOrchestrator]
    C --> D[Group logs by host/user/session]

    D --> E[DetectionAgent<br/>LLM-based analysis]
    E --> F[AdvisorAgent<br/>Context & remediation]
    F --> G[QualityAgent<br/>Noise suppression]

    G --> H[Create AI-only Detection]
    H --> I[(detections)]
    I --> J[WebSocket Broadcaster]
    J --> K[Analyst Dashboard]
```

---

## E.5 Feedback Loop and Continuous Learning

Shows how analyst feedback becomes suppression patterns applied by the QualityAgent.

**Figure E.5 - Feedback Loop and Pattern Application**

```mermaid
graph LR
    subgraph UI["Analyst Dashboard"]
        D1["Mark detection as false positive"]
        D2["Provide feedback reason & context"]
    end

    subgraph API["AI-Security Backend API"]
        FB["POST /api/feedback"]
        S["Store / update<br/>feedback_patterns"]
    end

    subgraph DB["PostgreSQL"]
        PAT[("feedback_patterns")]
        DET[("detections")]
    end

    subgraph Engine["Detection Engine"]
        QA["QualityAgent"]
        ORCH["DetectionOrchestrator"]
    end

    D1 --> D2 --> FB
    FB --> S --> PAT
    PAT --> QA
    DET --> QA
    QA --> ORCH["Apply suppression rules<br/>during next run"]
```

---

## E.6 Detection Status Lifecycle

Summarises the main state transitions for detections.

**Figure E.6 - Detection Status State Machine**

```mermaid
stateDiagram-v2
    [*] --> Active
    Active --> Under_Review: analyst opens detection
    Under_Review --> Confirmed: marked as true positive
    Under_Review --> False_Positive: marked as false positive
    False_Positive --> Patterned: feedback pattern created
    Patterned --> Suppressed: similar future detections auto-suppressed
    Confirmed --> Closed: incident resolved
    Suppressed --> [*]
    Closed --> [*]
```
