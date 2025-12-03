# API Reference

Complete REST API documentation for the AI Security Backend.

## Base URL

```
http://localhost:3000
```

## Endpoints

### Detections

#### GET /api/detections

Get all detections with optional filtering.

**Query Parameters**:
```
?severity=high,critical
&status=active
&limit=50
&offset=0
&sort=created_at
&order=desc
```

**Response**:
```json
{
  "detections": [
    {
      "id": "uuid",
      "rule_id": "SEC-AUTH-002",
      "title": "Brute Force Attack Detected",
      "severity": "high",
      "status": "active",
      "confidence": 0.95,
      "created_at": "2025-01-01T10:00:00.000Z"
    }
  ],
  "total": 100,
  "limit": 50,
  "offset": 0
}
```

#### GET /api/detections/:id

Get single detection with full details.

**Response**:
```json
{
  "id": "uuid",
  "rule_id": "SEC-AUTH-002",
  "category": "AUTHENTICATION_ATTACK",
  "severity": "high",
  "title": "Brute Force Attack Detected",
  "description": "Multiple failed login attempts...",
  "confidence": 0.95,
  "mitre_tactics": ["TA0006"],
  "mitre_techniques": ["T1110.001"],
  "owasp_category": "A07:2021",
  "remediation_plan": {
    "summary": "...",
    "steps": [...],
    "references": [...]
  },
  "status": "active",
  "notes": null,
  "logs": [
    {
      "id": "log-uuid",
      "source": "windows",
      "event_type": "failed_login",
      "timestamp": "2025-01-01T09:59:00.000Z"
    }
  ],
  "created_at": "2025-01-01T10:00:00.000Z"
}
```

#### PATCH /api/detections/:id

Update detection status or notes.

**Request**:
```json
{
  "status": "investigating",
  "notes": "Assigned to security team"
}
```

**Response**: Updated detection object

#### DELETE /api/detections/:id

Delete detection.

**Response**: `204 No Content`

### Logs

#### GET /api/logs

Get normalized logs.

**Query Parameters**:
```
?source=windows
&event_type=failed_login
&limit=100
&since=2025-01-01T00:00:00Z
```

**Response**:
```json
{
  "logs": [
    {
      "id": "uuid",
      "source": "windows",
      "event_type": "failed_login",
      "normalized_data": {
        "source_ip": "192.168.1.100",
        "user": "admin",
        "result": "failure"
      },
      "timestamp": "2025-01-01T10:00:00.000Z",
      "processed_at": "2025-01-01T10:01:00.000Z"
    }
  ],
  "total": 500
}
```

#### GET /api/logs/:id

Get single log with full raw data.

### Stats

#### GET /api/stats

Get system statistics.

**Response**:
```json
{
  "detections": {
    "total": 150,
    "by_severity": {
      "critical": 10,
      "high": 40,
      "medium": 70,
      "low": 30
    },
    "by_status": {
      "active": 100,
      "investigating": 30,
      "resolved": 15,
      "false_positive": 5
    },
    "last_24h": 45
  },
  "logs": {
    "total": 10000,
    "processed": 9500,
    "pending": 500
  },
  "rules": {
    "total": 9,
    "top_triggered": [
      {
        "rule_id": "SEC-AUTH-002",
        "count": 50
      }
    ]
  }
}
```

### Feedback

#### POST /api/feedback

Submit feedback on detection.

**Request**:
```json
{
  "detection_id": "uuid",
  "is_helpful": false,
  "comment": "False positive - legitimate admin activity"
}
```

**Response**:
```json
{
  "id": "feedback-uuid",
  "detection_id": "uuid",
  "is_helpful": false,
  "created_at": "2025-01-01T10:00:00.000Z"
}
```

#### GET /api/feedback/patterns

Get false positive patterns.

**Response**:
```json
{
  "patterns": [
    {
      "pattern_id": "fp-auth-001",
      "rule_id": "SEC-AUTH-002",
      "description": "Admin password reset attempts",
      "confidence": 0.85,
      "occurrences": 5
    }
  ]
}
```

### Search

#### POST /api/search

Search detections and logs.

**Request**:
```json
{
  "query": "brute force",
  "types": ["detections", "logs"],
  "filters": {
    "severity": ["high", "critical"],
    "time_range": "24h"
  }
}
```

### Configuration

#### GET /api/config

Get current configuration.

**Response**:
```json
{
  "detection": {
    "mode": "polling",
    "pollingIntervalMs": 600000,
    "batchSize": 100
  },
  "codex": {
    "model": "gpt-5-mini",
    "modelProvider": "azure"
  }
}
```

#### PATCH /api/config

Update runtime configuration.

**Request**:
```json
{
  "detection": {
    "mode": "polling",
    "pollingIntervalMs": 300000
  }
}
```

**Note**: Restarts LogListener automatically

### Rules

#### GET /api/rules

Get all loaded security rules.

**Response**:
```json
{
  "rules": [
    {
      "ruleId": "SEC-AUTH-002",
      "name": "Windows Failed Login Attempts",
      "category": "AUTHENTICATION_ATTACK",
      "severity": "high"
    }
  ],
  "count": 9
}
```

#### GET /api/rules/:ruleId

Get single rule details.

## WebSocket

### Connect

```javascript
const ws = new WebSocket('ws://localhost:3000/ws/detections');
```

### Events

**detection.created**:
```json
{
  "type": "detection.created",
  "data": {
    "id": "uuid",
    "title": "Brute Force Attack",
    "severity": "high"
  },
  "timestamp": "2025-01-01T10:00:00.000Z"
}
```

See: [WebSocket Documentation](./websocket.md)

## Error Responses

All endpoints return consistent error format:

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Invalid severity value",
    "details": [...]
  }
}
```

**Status Codes**:
- `200` - Success
- `201` - Created
- `204` - No Content
- `400` - Bad Request
- `404` - Not Found
- `500` - Internal Server Error

## Authentication

Currently no authentication is implemented. This system is designed for internal deployment within a secure network.

**Security Recommendations:**
- Deploy behind a VPN or firewall
- Use network segmentation to restrict access
- Consider adding API authentication for production deployments

**Future Considerations:**
- API key authentication
- JWT token-based auth
- Rate limiting per client
- Role-based access control (RBAC)

---

For more details:
- [WebSocket](./websocket.md)
- [Database](./database.md)
