# Appendix D - API Endpoint Documentation

This appendix summarizes the REST and WebSocket APIs exposed by the Log-Transformer and AI-Security backend services. These interfaces support log ingestion, detection retrieval, feedback collection, and real-time event streaming.

---

## D.1 Log-Transformer API (Ingestion Service)

Endpoints focus on log submission and background job monitoring. All responses are JSON.

### D.1.1 POST `/ingest/evtx`

Queues a Windows EVTX file for ingestion.

**Figure D.1 - EVTX Ingestion Endpoint**

```http
POST /ingest/evtx
Content-Type: multipart/form-data
file=<evtx_file>
```

**Response**

```json
{
  "jobId": "f2b9872d-65cd-4f91-bd50-e5ce5c6e0890",
  "status": "queued"
}
```

### D.1.2 POST `/ingest/json`

Accepts JSON-formatted logs (single or batched).

**Figure D.2 - JSON Log Ingestion Endpoint**

```http
POST /ingest/json
Content-Type: application/json
[
  { "timestamp": "...", "message": "...", "source_ip": "..." },
  { "timestamp": "...", "message": "...", "source_ip": "..." }
]
```

**Response**

```json
{
  "jobId": "59ab4fa1-aa4d-4dbd-9ac7-2eb0f8169231",
  "recordsAccepted": 128
}
```

### D.1.3 GET `/jobs/{id}`

Retrieves the status of an ingestion job.

**Figure D.3 - Job Status Endpoint**

```http
GET /jobs/f2b9872d-65cd-4f91-bd50-e5ce5c6e0890
```

**Response**

```json
{
  "jobId": "f2b9872d-65cd-4f91-bd50-e5ce5c6e0890",
  "status": "completed",
  "processedCount": 50000,
  "errorCount": 0
}
```

---

## D.2 AI-Security Backend API

Provides access to normalised logs, detections, and analyst feedback.

### D.2.1 GET `/api/logs`

Fetches logs with filtering and pagination.

**Figure D.4 - Logs Retrieval Endpoint**

```http
GET /api/logs?page=1&pageSize=50&source=syslog&severity=high
```

**Response**

```json
{
  "items": [...],
  "page": 1,
  "pageSize": 50,
  "total": 457
}
```

### D.2.2 GET `/api/logs/{id}`

Returns a single normalised log entry.

**Figure D.5 - Log Detail Endpoint**

```http
GET /api/logs/3f91a230-0fb9-4a57-8558-d0e8cd9f03e9
```

**Response**

```json
{
  "id": "...",
  "timestamp": "...",
  "normalized_data": { ... }
}
```

### D.2.3 GET `/api/detections`

Lists detections optionally filtered by severity, source, or status.

**Figure D.6 - Detections Retrieval Endpoint**

```http
GET /api/detections?severity=high&source=windows_evtx
```

**Response**

```json
{
  "detected": 12,
  "items": [
    {
      "id": "...",
      "severity": "high",
      "status": "active",
      "mitre_technique": "T1110"
    }
  ]
}
```

### D.2.4 GET `/api/detections/{id}`

Provides full detection details, including linked logs and AI outputs.

**Figure D.7 - Detection Detail Endpoint**

```http
GET /api/detections/55e38e5a-6979-49b8-a9c8-9dd066357d05
```

**Response**

```json
{
  "id": "55e38e5a-6979-49b8-a9c8-9dd066357d05",
  "severity": "high",
  "description": "...",
  "logs": [...],
  "recommendations": [...]
}
```

### D.2.5 POST `/api/feedback`

Creates or updates a feedback pattern for the QualityAgent.

**Figure D.8 - Feedback Submission Endpoint**

```http
POST /api/feedback
Content-Type: application/json
{
  "pattern_id": "fp-admin-maintenance",
  "reason": "Recurring benign maintenance window activity",
  "attributes": {
    "account_name": "svc_automation",
    "ip_range": "10.0.5.0/24"
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

## D.3 WebSocket API

Enables low-latency streaming of detection events to analyst dashboards.

### D.3.1 `ws://host/ws/detections`

Establishes a WebSocket subscription for new detections.

**Figure D.9 - WebSocket Subscription Endpoint**

```text
ws://localhost:3000/ws/detections
```

**Example Event Payload**

```json
{
  "event": "detection.created",
  "data": {
    "id": "c196f8b7-a27a-4204-a469-5e1c7a7e4352",
    "severity": "high",
    "timestamp": "2025-01-14T08:02:11Z",
    "description": "Repeated failed logons consistent with brute-force activity."
  }
}
```

---

## D.4 Endpoint Summary

| Category  | Endpoint              | Description                   |
| --------- | --------------------- | ----------------------------- |
| Ingestion | `POST /ingest/evtx`   | Upload EVTX log files         |
|           | `POST /ingest/json`   | Batch JSON log intake         |
|           | `GET /jobs/{id}`      | Check job processing status   |
| Log Access| `GET /api/logs`       | Paginated log retrieval       |
|           | `GET /api/logs/{id}`  | View individual log           |
| Detection | `GET /api/detections` | Query detections              |
|           | `GET /api/detections/{id}` | View detection detail    |
| Feedback  | `POST /api/feedback`  | Submit false-positive patterns|
| Realtime  | `WS /ws/detections`   | Live detection events         |
