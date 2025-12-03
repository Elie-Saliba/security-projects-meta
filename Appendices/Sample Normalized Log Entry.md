# Appendix B - Sample Normalised Log Entries

This appendix provides representative examples of the normalised log records produced by the Log-Transformer service. The examples highlight how heterogeneous sources are converted into the unified JSONB schema, enabling consistent downstream analysis.

---

## B.1 Windows EVTX Normalised Log Example

This example illustrates how Windows Security Event Logs are parsed, flattened, and mapped into the unified schema used by the `normalized_logs` table.

**Figure B.1 - Normalised Windows EVTX Record (4625 Failed Logon)**

```json
{
  "id": "b2e5fa17-d78f-4c65-9ed3-71f912c8c221",
  "timestamp": "2025-01-14T02:41:52Z",
  "source": "windows_evtx",
  "event_type": "4625_failed_logon",
  "severity": "medium",
  "raw_data": {
    "EventID": 4625,
    "ProcessID": 780,
    "ThreadID": 920,
    "Channel": "Security"
  },
  "normalized_data": {
    "account_name": "svc_sql",
    "logon_type": "3",
    "ip_address": "185.199.111.22",
    "workstation": "CORP-SQL01",
    "failure_reason": "Unknown username or bad password"
  },
  "status": "processed"
}
```

---

## B.2 Syslog Normalised Log Example

Syslog messages from Linux-based systems and network appliances are parsed into structured key-value fields while preserving the original payload in JSONB format.

**Figure B.2 - Normalised Syslog Record**

```json
{
  "id": "3f91a230-0fb9-4a57-8558-d0e8cd9f03e9",
  "timestamp": "2025-01-14T09:15:33Z",
  "source": "syslog",
  "event_type": "auth_failure",
  "severity": "low",
  "raw_data": {
    "facility": "auth",
    "priority": 3,
    "hostname": "web-gateway-01"
  },
  "normalized_data": {
    "process": "sshd",
    "message": "Failed password for invalid user test from 192.168.10.50",
    "ip_address": "192.168.10.50",
    "username": "test",
    "action": "authentication_failure"
  },
  "status": "processed"
}
```

---

## B.3 Wazuh Alert Normalised Log Example

Wazuh alerts are enriched with rule metadata and converted into structured detection-ready fields.

**Figure B.3 - Normalised Wazuh Alert Record**

```json
{
  "id": "ca928cd0-7c20-4e8d-8add-13b9e1f240e2",
  "timestamp": "2025-01-13T17:08:11Z",
  "source": "wazuh",
  "event_type": "wazuh_rule_match",
  "severity": "high",
  "raw_data": {
    "rule": {
      "id": 5503,
      "level": 8,
      "description": "Repeated SSH authentication failures"
    }
  },
  "normalized_data": {
    "rule_id": 5503,
    "description": "Repeated SSH authentication failures",
    "ip_address": "203.0.113.7",
    "attempts": 12,
    "process": "sshd"
  },
  "status": "processed"
}
```

---

## B.4 JSON / Application Log Normalised Example

JSON logs from applications (e.g., API gateways, microservices) are mapped directly to the normalised schema while preserving all relevant fields.

**Figure B.4 - Normalised Application JSON Log**

```json
{
  "id": "ef71f0ee-61f6-4a7a-96cb-e5ab5a82bdf8",
  "timestamp": "2025-01-12T11:23:44Z",
  "source": "json",
  "event_type": "api_request",
  "severity": "info",
  "raw_data": {
    "path": "/api/v1/orders",
    "method": "POST",
    "status": 500
  },
  "normalized_data": {
    "route": "/api/v1/orders",
    "http_method": "POST",
    "status_code": 500,
    "latency_ms": 348,
    "client_ip": "198.51.100.24"
  },
  "status": "processed"
}
```

---

## B.5 Schema Reference Summary

All normalised logs conform to the schema defined in LT-6-database.md and as-database.md. Essential fields are summarised below.

| Field            | Description                                      |
| ---------------- | ------------------------------------------------ |
| `id`             | UUID primary key                                 |
| `timestamp`      | Event timestamp (UTC)                            |
| `source`         | Origin system (evtx, syslog, wazuh, json)        |
| `event_type`     | Normalised classification of event               |
| `severity`       | Log severity (info/low/medium/high)              |
| `raw_data`       | Unmodified source data in JSONB                  |
| `normalized_data`| Structured fields used by detection              |
| `status`         | Processing status (queued/processed/error)       |
