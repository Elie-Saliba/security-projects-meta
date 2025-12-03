# API Reference

## Overview

The Log-Transformer API provides RESTful endpoints for log ingestion, job management, and system monitoring. All endpoints accept and return JSON (except file uploads which use `multipart/form-data`).

## Base URL

- **Development**: `http://localhost:5132`
- **Production (Docker)**: `http://localhost:5001`
- **API Documentation**: Available at `/swagger` in all environments

## Authentication

Currently, the API does not require authentication. For production deployments, consider implementing:
- API keys
- OAuth 2.0
- JWT tokens
- IP whitelisting

## Endpoints

### File Upload Endpoints

#### Upload Log File

Accepts log files from various sources for asynchronous processing.

**Endpoint**: `POST /ingest/evtx`

**Note**: Despite the endpoint name, this accepts multiple file types (EVTX, JSONL, and any registered parser type).

**Content Type**: `multipart/form-data`

**Parameters**:
| Name | Type | Required | Description |
|------|------|----------|-------------|
| file | File | Yes | The log file to upload |

**Supported File Types**:
- `.evtx` - Windows Event Log files
- `.jsonl` - JSON Lines format
- Additional types can be added via parser plugins

**Request Example** (cURL):
```bash
curl -X POST http://localhost:5001/ingest/evtx \
  -H "Content-Type: multipart/form-data" \
  -F "file=@/path/to/logfile.evtx"
```

**Request Example** (PowerShell):
```powershell
$file = Get-Item "C:\logs\security.evtx"
$uri = "http://localhost:5001/ingest/evtx"

$form = @{
    file = $file
}

Invoke-RestMethod -Uri $uri -Method Post -Form $form
```

**Success Response** (202 Accepted):
```json
{
  "job_id": "01HXXXXXXXXXXXXXXXXXXX",
  "status_url": "/ingest/jobs/01HXXXXXXXXXXXXXXXXXXX"
}
```

**Error Responses**:

*400 Bad Request - No file provided*:
```json
{
  "error": "A non-empty file must be provided using the 'file' form field."
}
```

*400 Bad Request - Invalid file type*:
```json
{
  "error": "Only .evtx and .jsonl uploads are supported."
}
```

*500 Internal Server Error - Storage failure*:
```json
{
  "error": "Unable to store the uploaded file."
}
```

---

### Job Management Endpoints

#### Get Job Status

Retrieves the current status and statistics for an ingestion job.

**Endpoint**: `GET /ingest/jobs/{jobId}`

**Parameters**:
| Name | Type | Required | Description |
|------|------|----------|-------------|
| jobId | string | Yes | The unique job identifier (ULID format) |

**Request Example**:
```bash
curl http://localhost:5001/ingest/jobs/01HXXXXXXXXXXXXXXXXXXX
```

**Success Response** (200 OK):

*Queued Job*:
```json
{
  "id": "01HXXXXXXXXXXXXXXXXXXX",
  "source": "evtx",
  "filename": "security-events.evtx",
  "path": "uploads/20251107/01HXXXXXXXXXXXXXXXXXXX.evtx",
  "status": "queued",
  "inserted": 0,
  "skipped": 0,
  "error": null,
  "created_at": "2025-11-07T10:30:00.000Z",
  "started_at": null,
  "finished_at": null
}
```

*Running Job*:
```json
{
  "id": "01HXXXXXXXXXXXXXXXXXXX",
  "source": "evtx",
  "filename": "security-events.evtx",
  "path": "uploads/20251107/01HXXXXXXXXXXXXXXXXXXX.evtx",
  "status": "running",
  "inserted": 1250,
  "skipped": 0,
  "error": null,
  "created_at": "2025-11-07T10:30:00.000Z",
  "started_at": "2025-11-07T10:30:05.000Z",
  "finished_at": null
}
```

*Completed Job*:
```json
{
  "id": "01HXXXXXXXXXXXXXXXXXXX",
  "source": "evtx",
  "filename": "security-events.evtx",
  "path": "uploads/20251107/01HXXXXXXXXXXXXXXXXXXX.evtx",
  "status": "done",
  "inserted": 5432,
  "skipped": 12,
  "error": null,
  "created_at": "2025-11-07T10:30:00.000Z",
  "started_at": "2025-11-07T10:30:05.000Z",
  "finished_at": "2025-11-07T10:32:18.000Z"
}
```

*Failed Job*:
```json
{
  "id": "01HXXXXXXXXXXXXXXXXXXX",
  "source": "evtx",
  "filename": "corrupted-log.evtx",
  "path": "uploads/20251107/01HXXXXXXXXXXXXXXXXXXX.evtx",
  "status": "error",
  "inserted": 234,
  "skipped": 0,
  "error": "Unable to parse EVTX file: Invalid file format",
  "created_at": "2025-11-07T10:30:00.000Z",
  "started_at": "2025-11-07T10:30:05.000Z",
  "finished_at": "2025-11-07T10:30:45.000Z"
}
```

**Status Values**:
- `queued` - Job is waiting to be processed
- `running` - Job is currently being processed
- `done` - Job completed successfully
- `error` - Job failed with an error

**Error Responses**:

*404 Not Found*:
```json
{
  "error": "Job not found"
}
```

---

## Future Endpoints (Planned)

### Real-Time Ingestion Endpoints

These endpoints will accept real-time log streams from various sources:

#### FluentBit Endpoint

**Endpoint**: `POST /ingest/fluentbit`

**Request Body**:
```json
{
  "timestamp": "2025-11-07T10:30:00.000Z",
  "log": "User authentication successful",
  "level": "info",
  "source": "auth-service",
  "metadata": {
    "user_id": "12345",
    "ip_address": "192.168.1.100"
  }
}
```

#### Wazuh Endpoint

**Endpoint**: `POST /ingest/wazuh`

**Request Body**:
```json
{
  "timestamp": "2025-11-07T10:30:00.000Z",
  "rule": {
    "id": 5710,
    "level": 5,
    "description": "Attempt to login using a non-existent user"
  },
  "agent": {
    "id": "001",
    "name": "server01"
  },
  "data": {
    "srcip": "192.168.1.100",
    "dstuser": "admin"
  }
}
```

#### Syslog Endpoint

**Endpoint**: `POST /ingest/syslog`

**Request Body**:
```json
{
  "timestamp": "2025-11-07T10:30:00.000Z",
  "hostname": "web-server-01",
  "facility": "auth",
  "severity": "warning",
  "message": "Failed password for invalid user admin from 192.168.1.100 port 52312 ssh2"
}
```

---

## Data Models

### IngestJob

Represents a background processing job for uploaded files.

| Field | Type | Description |
|-------|------|-------------|
| id | string | Unique job identifier (ULID format) |
| source | string | Source system type (e.g., "evtx", "jsonl", "syslog") |
| filename | string | Original filename |
| path | string | Relative storage path |
| status | string | Current job status (queued/running/done/error) |
| inserted | number | Count of records successfully inserted |
| skipped | number | Count of records skipped (duplicates, invalid) |
| error | string? | Error message if status is "error" |
| created_at | datetime | Job creation timestamp |
| started_at | datetime? | Processing start timestamp |
| finished_at | datetime? | Processing completion timestamp |

### NormalizedLog

The standardized log format that all sources are transformed into.

| Field | Type | Description |
|-------|------|-------------|
| id | uuid | Unique log entry identifier |
| timestamp | datetime | Log event timestamp |
| source | string | Source system identifier |
| event_type | string | Type/category of event |
| severity | enum | Severity level (critical/high/medium/low/info) |
| raw_data | jsonb | Original unmodified log data |
| normalized_data | jsonb | Standardized fields extracted from log |
| message | string? | Human-readable message |
| processed_at | datetime? | When the log was processed |
| status | enum | Processing status (pending/processed/failed) |
| created_at | datetime | Record creation timestamp |
| updated_at | datetime | Last update timestamp |

### NormalizedData Structure

The `normalized_data` JSONB field contains standardized fields:

```json
{
  "source_ip": "192.168.1.100",
  "destination_ip": "10.0.0.50",
  "user": "jdoe",
  "host": "workstation-01",
  "process": "sshd",
  "file_path": "/var/log/auth.log",
  "action": "login_attempt",
  "result": "failure",
  "channel": "security",
  "event_id": 4625,
  "additional_fields": {
    "logon_type": "3",
    "failure_reason": "Unknown user name or bad password"
  }
}
```

---

## Error Handling

All API endpoints follow consistent error response patterns:

### Error Response Format

```json
{
  "error": "Description of what went wrong",
  "details": "Additional context (optional)",
  "code": "ERROR_CODE (optional)"
}
```

### HTTP Status Codes

| Code | Meaning | When Used |
|------|---------|-----------|
| 200 | OK | Successful GET requests |
| 202 | Accepted | File upload accepted for processing |
| 400 | Bad Request | Invalid input, missing parameters |
| 404 | Not Found | Resource not found |
| 500 | Internal Server Error | Server-side processing error |
| 503 | Service Unavailable | Database or dependency unavailable |

---

## Rate Limiting

Currently, no rate limiting is implemented. For production deployments, consider:

- **File Uploads**: Limit to 10 requests per minute per IP
- **Job Status**: Limit to 100 requests per minute per IP
- **Real-time Endpoints**: Limit to 1000 requests per minute per source

---

## Pagination

Job listing endpoints (when implemented) will support pagination:

**Query Parameters**:
- `page` - Page number (default: 1)
- `page_size` - Items per page (default: 50, max: 200)

**Response Headers**:
- `X-Total-Count` - Total number of items
- `X-Page` - Current page number
- `X-Page-Size` - Items per page
- `Link` - Pagination links (first, prev, next, last)

---

## Swagger/OpenAPI

Interactive API documentation is available at `/swagger` when the application is running:

```
http://localhost:5001/swagger/index.html
```

Features:
- Test endpoints directly from the browser
- View request/response schemas
- Download OpenAPI specification
- Generate client code

---

## SDK/Client Libraries

Currently, client libraries are not provided. You can:

1. **Generate from OpenAPI**: Use the Swagger endpoint to generate clients
   ```bash
   # Generate TypeScript client
   npx @openapitools/openapi-generator-cli generate \
     -i http://localhost:5001/swagger/v1/swagger.json \
     -g typescript-fetch \
     -o ./generated-client
   ```

2. **Use HTTP libraries directly**: Standard HTTP libraries work with all endpoints
   - C#: `HttpClient`
   - JavaScript: `fetch`, `axios`
   - Python: `requests`
   - PowerShell: `Invoke-RestMethod`

---

## Versioning

The API currently does not use versioning. Future versions will use URL-based versioning:

- `/v1/ingest/evtx`
- `/v2/ingest/evtx`

---

## Security Considerations

### Production Recommendations

1. **Enable HTTPS**: Use TLS certificates
2. **Add Authentication**: Implement API key or OAuth
3. **Validate Input**: Additional file type validation
4. **Rate Limiting**: Prevent abuse
5. **CORS Configuration**: Restrict allowed origins
6. **File Size Limits**: Set maximum upload sizes
7. **Audit Logging**: Track all API requests

### Current Security Features

- Input validation on file types
- SQL injection protection via Entity Framework
- Path traversal prevention
- Atomic file operations
