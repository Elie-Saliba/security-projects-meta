# Appendix C - Security Rule Examples

This appendix presents representative YAML-based detection rules executed by the AI-Security backend. These deterministic rules form the first stage of the hybrid pipeline, running within the `RuleEngine` before detections are enriched by the AI agents.

---

## C.1 Windows Brute-Force Authentication Rule

Identifies repeated failed logon attempts (Event ID 4625) from the same IP address within a short time window.

**Figure C.1 - Windows Failed Logon Threshold Rule**

```yaml
id: WIN-4625-BRUTEFORCE
name: "Windows Failed Logon Threshold"
description: "Detects repeated authentication failures from the same IP address."
severity: medium

match:
  event_type: "4625_failed_logon"

conditions:
  - type: threshold
    field: "normalized_data.ip_address"
    count: 5
    window_minutes: 3

metadata:
  mitre_technique: "T1110"
  category: "Brute Force"
```

---

## C.2 Suspicious Process Spawn (Sysmon) Rule

Captures cmd.exe spawning net.exe or sc.exe, which is a common privilege escalation pattern.

**Figure C.2 - Sysmon Privilege Escalation Rule**

```yaml
id: SYSMON-PRIV-ESC
name: "Sysmon Privilege Escalation Pattern"
description: "Detects high-risk process creation chains consistent with privilege escalation."
severity: high

match:
  event_type: "sysmon_process_create"

conditions:
  - type: equals
    field: "normalized_data.parent_process"
    value: "cmd.exe"

  - type: any_of
    field: "normalized_data.process_name"
    values:
      - "net.exe"
      - "sc.exe"

metadata:
  mitre_technique: "T1068"
  category: "Privilege Escalation"
```

---

## C.3 Web Application SQL Injection Attempt Rule (OWASP)

Flags SQLi indicators originating from WAF telemetry or API gateway logs.

**Figure C.3 - Web Application SQL Injection Rule**

```yaml
id: WEB-SQLI-API
name: "SQL Injection Attempt"
description: "Flags potential SQL injection attempts based on WAF rule hits or payload analysis."
severity: high

match:
  event_type: "waf_alert"

conditions:
  - type: equals
    field: "normalized_data.rule_id"
    value: 942101   # OWASP CRS SQL Injection

  - type: contains
    field: "normalized_data.request_payload"
    value: "' OR 1=1 --"

metadata:
  mitre_technique: "T1190"
  category: "Injection"
  owasp: "A03:2021"
```

---

## C.4 Suspicious Login Outside Maintenance Window

Detects unusual logins by service accounts outside their allowed operational periods.

**Figure C.4 - After-Hours Service Account Logon Rule**

```yaml
id: WIN-SVC-AFTERHOURS
name: "Service Account Login Outside Maintenance Window"
description: "Detects unexpected authentication activity by service accounts outside the approved maintenance timeframe."
severity: medium

match:
  event_type: "4624_successful_logon"

conditions:
  - type: in
    field: "normalized_data.account_name"
    values:
      - "svc_automation"
      - "svc_backup"

  - type: time_not_between
    field: "timestamp"
    start: "02:00"
    end: "03:00"

metadata:
  mitre_technique: "T1078"
  category: "Valid Accounts"
```

---

## C.5 SSH Brute-Force Rule (Syslog / Wazuh)

Aggregates multiple failed SSH password attempts observed in Linux telemetry streams.

**Figure C.5 - SSH Brute-Force Detection Rule**

```yaml
id: SSH-BRUTEFORCE
name: "SSH Brute-Force Attempt"
description: "Identifies high-frequency SSH authentication failures from the same IP address."
severity: high

match:
  event_type: "auth_failure"

conditions:
  - type: threshold
    field: "normalized_data.ip_address"
    count: 10
    window_minutes: 5

metadata:
  mitre_technique: "T1110"
  category: "Brute Force"
```

---

## C.6 Rule Schema Reference

All rules conform to the structure defined in `as-rule-system.md`. Essential sections are summarised below.

| Section    | Purpose                                      |
| ---------- | -------------------------------------------- |
| `id`       | Unique rule identifier                       |
| `name`     | Human-readable title                         |
| `description` | Explanation of detection logic            |
| `severity` | Alert criticality (low/medium/high)          |
| `match`    | Event type to evaluate                       |
| `conditions` | One or more logical checks                 |
| `metadata` | Mapping to MITRE/OWASP or internal taxonomy  |
