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

**Figure E.7: Detection Lifecycle Sequence with Latency Analysis**

```mermaid
sequenceDiagram
    participant DB as PostgreSQL<br/>Database
    participant LL as LogListener<br/>LISTEN/NOTIFY
    participant DP as Detection<br/>Pipeline
    participant RE as RuleEngine<br/>Pattern Match
    participant DA as DetectionAgent<br/>AI Validation
    participant AA as AdvisorAgent<br/>Remediation
    participant QA as QualityAgent<br/>False Positive Filter
    participant WS as WebSocket<br/>EventBroadcaster
    participant UI as Frontend<br/>Analyst Dashboard
    
    Note over DB,UI: Complete Detection Lifecycle (Target: <5 seconds)
    
    DB->>LL: NOTIFY new_log_inserted
    Note right of LL: <50ms<br/>Real-time trigger
    
    LL->>DP: Process new log batch
    Note right of DP: <100ms<br/>Grouping & routing
    
    DP->>RE: Evaluate against rules
    Note right of RE: ~800ms<br/>YAML rule matching<br/>MITRE/OWASP mapping
    
    alt Rule Match Found
        RE->>DA: Validate threat context
        Note right of DA: ~1,800ms<br/>LLM analysis<br/>Context retrieval<br/>Behavioral assessment
        
        alt Confirmed Threat
            DA->>AA: Generate remediation plan
            Note right of AA: ~900ms<br/>MITRE technique lookup<br/>Action steps synthesis<br/>Reference gathering
            
            AA->>QA: Check false positive patterns
            Note right of QA: ~300ms<br/>Query feedback_patterns<br/>Compare indicators<br/>Confidence scoring
            
            alt Not Learned False Positive
                QA->>DB: Persist detection record
                Note right of DB: ~200ms<br/>Transaction commit<br/>detection_logs junction
                
                DB->>WS: Trigger detection.created
                Note right of WS: <100ms<br/>Broadcast to connections
                
                WS->>UI: Real-time alert notification
                Note right of UI: <50ms<br/>Dashboard update<br/>Alert sound/visual
                
                Note over DB,UI: Total Latency: ~3.8 seconds
                
            else Matches False Positive Pattern
                QA->>QA: Discard detection
                Note right of QA: No database write<br/>No analyst notification<br/>Learned pattern applied
            end
            
        else False Positive (AI Assessment)
            DA->>DA: Discard without remediation
            Note right of DA: No downstream processing<br/>Saves ~1,200ms
        end
        
    else No Rule Match
        RE->>RE: Skip - no detection
        Note right of RE: No AI processing needed<br/>Efficient filtering
    end
    
    Note over DB,UI: Performance Breakdown:<br/>Rule Evaluation: 800ms (21%)<br/>AI Validation: 1,800ms (47%)<br/>Remediation: 900ms (24%)<br/>Quality Check: 300ms (8%)<br/>Persistence & Broadcast: <100ms (0%)
```