# Appendix H - MCP Tool Specifications

This appendix documents the Model Context Protocol (MCP) tools integrated into the AI-Security backend. Detection, Advisor, and Quality MCP servers expose structured commands so AI agents can retrieve context, generate recommendations, and manage suppression patterns.

---

## H.0 Overview of MCP Integration

Three MCP servers run alongside the backend:

- **Detection MCP** – log summaries, grouped events, lookup utilities.
- **Advisor MCP** – remediation recommendations and MITRE mappings.
- **Quality MCP** – feedback pattern retrieval and updates.

**Figure H.0: Model Context Protocol Tool Integration**

```mermaid
graph TB
    subgraph Agents["AI Agents (Codex SDK)"]
        subgraph AgentsSub[" "]
        DA[DetectionAgent<br/>Thread ID: det-xxx<br/>Workspace: detection/]
        AA[AdvisorAgent<br/>Thread ID: adv-xxx<br/>Workspace: advisor/]
        QA[QualityAgent<br/>Thread ID: qual-xxx<br/>Workspace: quality/]
        end
    end
    
    subgraph MCP["MCP Servers (HTTP Transport)"]
        subgraph MCPSub[" "]
        MCP1[Detection MCP<br/>Port 3100<br/>Session Management]
        MCP2[Advisor MCP<br/>Port 3101<br/>Session Management]
        MCP3[Quality MCP<br/>Port 3102<br/>Session Management]
        end
    end
    
    subgraph Tools["Tool Catalog"]
        subgraph ToolsSub[" "]
        T1[query_related_logs<br/>query_log_patterns<br/>get_log_statistics]
        T2[query_similar_detections<br/>lookup_mitre_technique<br/>lookup_owasp_category<br/>search_remediation_refs]
        T3[search_feedback_history<br/>query_fp_patterns<br/>get_feedback_for_rule]
        end
    end
    
    subgraph Data["Data Sources"]
        subgraph DataSub[" "]
        DB[(PostgreSQL<br/>normalized_logs<br/>detections<br/>feedbacks)]
        MITRE[MITRE ATT&CK<br/>Knowledge Base]
        OWASP[OWASP Top 10<br/>Documentation]
        FB[Feedback History<br/>Markdown Files]
        end
    end
    
    DA -->|HTTP/JSON-RPC| MCP1
    AA -->|HTTP/JSON-RPC| MCP2
    QA -->|HTTP/JSON-RPC| MCP3
    
    MCP1 --> T1
    MCP2 --> T2
    MCP3 --> T3
    
    T1 --> DB
    T2 --> DB
    T2 --> MITRE
    T2 --> OWASP
    T3 --> DB
    T3 --> FB
    
    style DA fill:#007EF9,color:#fff
    style AA fill:#007EF9,color:#fff
    style QA fill:#007EF9,color:#fff
    style MCP1 fill:#28a745,color:#fff
    style MCP2 fill:#28a745,color:#fff
    style MCP3 fill:#28a745,color:#fff
```



**Figure H.1 - MCP Tool Integration Architecture**

```mermaid
graph TB
    subgraph Agents["AI Agents"]
        DA["DetectionAgent"]
        AA["AdvisorAgent"]
        QA["QualityAgent"]
    end

    subgraph MCP["MCP Tool Servers"]
        DETS["Detection MCP"]
        ADVS["Advisor MCP"]
        QUALS["Quality MCP"]
    end

    subgraph DB["PostgreSQL Database"]
        LOGS["normalized_logs"]
        DET["detections"]
        PAT["feedback_patterns"]
    end

    DA --> DETS
    AA --> ADVS
    QA --> QUALS

    DETS --> LOGS
    ADVS --> LOGS
    ADVS --> DET
    QUALS --> PAT
```

---

## H.2 Detection MCP Tool

Provides the DetectionAgent with recent logs, grouped contexts, and summaries.

### H.2.1 Supported Commands

| Command             | Description                                        |
| ------------------- | -------------------------------------------------- |
| `getRecentLogs(limit)` | Returns the most recent N logs                  |
| `groupLogsBy(field)`  | Groups logs by host, username, or IP            |
| `getLogContext(logId)`| Retrieves related logs for the specified record |
| `summariseEvents(range)` | Summaries for a given time window            |

### H.2.2 Example Command: `getRecentLogs`

**Figure H.2 - Detection MCP Request**

```json
{
  "command": "getRecentLogs",
  "arguments": {
    "limit": 20
  }
}
```

**Response**

```json
{
  "logs": [
    {
      "id": "3f91a230-0fb9-4a57-8558-d0e8cd9f03e9",
      "source": "syslog",
      "event_type": "auth_failure",
      "timestamp": "2025-01-14T09:15:33Z"
    }
  ]
}
```

---

## H.3 Advisor MCP Tool

Assists the AdvisorAgent with remediation steps, MITRE mappings, and context explanations.

### H.3.1 Supported Commands

| Command                     | Description                                      |
| --------------------------- | ------------------------------------------------ |
| `generateRemediation(detection)` | Produces remediation steps                |
| `mapToMitre(detection)`     | Maps behaviour to MITRE techniques               |
| `explainContext(detection)` | Generates a human-readable explanation           |

### H.3.2 Example Command: `generateRemediation`

**Figure H.3 - Advisor MCP Request**

```json
{
  "command": "generateRemediation",
  "arguments": {
    "detectionId": "55e38e5a-6979-49b8-a9c8-9dd066357d05"
  }
}
```

**Response**

```json
{
  "remediation": [
    "Reset the affected credentials.",
    "Block the source IP address temporarily.",
    "Review audit logs for lateral movement indicators."
  ],
  "mitre": "T1110"
}
```

---

## H.4 Quality MCP Tool

Manages feedback patterns for false-positive suppression.

### H.4.1 Supported Commands

| Command               | Description                                 |
| --------------------- | ------------------------------------------- |
| `getPatterns()`       | Returns all feedback patterns               |
| `matchPattern(log)`   | Checks if a log matches an existing pattern |
| `addPattern(pattern)` | Stores a new pattern                        |
| `updatePattern(patternId)` | Updates an existing pattern            |

### H.4.2 Example Command: `addPattern`

**Figure H.4 - Quality MCP Request**

```json
{
  "command": "addPattern",
  "arguments": {
    "pattern_id": "fp-admin-maintenance",
    "reason": "Maintenance login window",
    "attributes": {
      "account_name": "svc_automation",
      "ip_range": "10.0.5.0/24",
      "time_window": "02:00-03:00"
    }
  }
}
```

**Response**

```json
{
  "status": "saved",
  "pattern_id": "fp-admin-maintenance"
}
```

---

## H.5 MCP Tool Schema Summary

| Field     | Description                                  |
| --------- | -------------------------------------------- |
| `command` | Operation invoked on the MCP tool            |
| `arguments` | Input parameters                           |
| `response` | Structured JSON output                      |
| `error`   | Optional error payload                       |

---

## H.6 Interaction Summary Diagram

Shows how MCP tools interact with AI agents during hybrid detection.

**Figure H.5 - MCP-Agent Interaction Summary**

```mermaid
sequenceDiagram
    participant DO as DetectionOrchestrator
    participant DA as DetectionAgent
    participant AA as AdvisorAgent
    participant QA as QualityAgent
    participant DET as Detection MCP
    participant ADV as Advisor MCP
    participant QLT as Quality MCP

    DO->>DA: evaluate(logs)
    DA->>DET: getContext()
    DET-->>DA: context

    DO->>AA: generate recommendations
    AA->>ADV: getRecommendations()
    ADV-->>AA: recommendations

    DO->>QA: apply feedback suppression
    QA->>QLT: matchPatterns()
    QLT-->>QA: result
```
