# Database

## Overview

Log-Transformer uses PostgreSQL as its primary data store, leveraging advanced features like JSONB columns, enum types, and partitioning for optimal performance. The database schema is designed for high-volume log ingestion while maintaining compatibility with the ai-security analysis platform.

## Database Architecture

### Database Design Principles

1. **Shared Schema**: Uses the same database as ai-security for seamless integration
2. **JSONB Storage**: Flexible storage for varying log formats
3. **Enum Types**: Type-safe severity and status values
4. **Indexing Strategy**: Optimized for time-series queries
5. **Partitioning-Ready**: Schema supports future table partitioning

### Database Diagram

```
┌─────────────────────────────────────────────────────┐
│              PostgreSQL Database                     │
│                  "Security"                          │
├─────────────────────────────────────────────────────┤
│                                                      │
│  ┌────────────────────────────────────────────┐    │
│  │         normalized_logs (Main Table)       │    │
│  ├────────────────────────────────────────────┤    │
│  │ id (UUID) PK                               │    │
│  │ timestamp (TIMESTAMPTZ)                    │    │
│  │ source (VARCHAR)                           │    │
│  │ event_type (VARCHAR)                       │    │
│  │ severity (ENUM)                            │    │
│  │ raw_data (JSONB)                           │    │
│  │ normalized_data (JSONB)                    │    │
│  │ message (TEXT)                             │    │
│  │ processed_at (TIMESTAMPTZ)                 │    │
│  │ status (ENUM)                              │    │
│  │ created_at (TIMESTAMPTZ)                   │    │
│  │ updated_at (TIMESTAMPTZ)                   │    │
│  └────────────────────────────────────────────┘    │
│                                                      │
│  ┌────────────────────────────────────────────┐    │
│  │         ingest_jobs (Job Queue)            │    │
│  ├────────────────────────────────────────────┤    │
│  │ id (VARCHAR) PK                            │    │
│  │ source (VARCHAR)                           │    │
│  │ filename (VARCHAR)                         │    │
│  │ path (VARCHAR)                             │    │
│  │ status (VARCHAR)                           │    │
│  │ inserted (INTEGER)                         │    │
│  │ skipped (INTEGER)                          │    │
│  │ error (TEXT)                               │    │
│  │ created_at (TIMESTAMP)                     │    │
│  │ started_at (TIMESTAMP)                     │    │
│  │ finished_at (TIMESTAMP)                    │    │
│  └────────────────────────────────────────────┘    │
│                                                      │
│  ┌────────────────────────────────────────────┐    │
│  │    Enum Types                              │    │
│  ├────────────────────────────────────────────┤    │
│  │ enum_normalized_logs_severity              │    │
│  │   - critical, high, medium, low, info      │    │
│  │                                            │    │
│  │ enum_normalized_logs_status                │    │
│  │   - pending, processed, failed             │    │
│  └────────────────────────────────────────────┘    │
│                                                      │
└─────────────────────────────────────────────────────┘
```

## Tables

### 1. normalized_logs

The primary table for storing transformed log data from all sources.

#### Schema Definition

```sql
CREATE TABLE normalized_logs (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    timestamp TIMESTAMP WITH TIME ZONE NOT NULL,
    source VARCHAR(255) NOT NULL,
    event_type VARCHAR(255) NOT NULL,
    severity enum_normalized_logs_severity NOT NULL DEFAULT 'info',
    raw_data JSONB NOT NULL,
    normalized_data JSONB NOT NULL,
    message TEXT,
    processed_at TIMESTAMP WITH TIME ZONE,
    status enum_normalized_logs_status NOT NULL DEFAULT 'pending',
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);
```

#### Column Descriptions

| Column | Type | Nullable | Description |
|--------|------|----------|-------------|
| id | UUID | No | Unique identifier for each log entry |
| timestamp | TIMESTAMPTZ | No | When the log event occurred (UTC) |
| source | VARCHAR(255) | No | Source system identifier (e.g., "evtx", "syslog", "wazuh") |
| event_type | VARCHAR(255) | No | Type or category of event |
| severity | ENUM | No | Severity level (critical/high/medium/low/info) |
| raw_data | JSONB | No | Original unmodified log data |
| normalized_data | JSONB | No | Standardized extracted fields |
| message | TEXT | Yes | Human-readable message or description |
| processed_at | TIMESTAMPTZ | Yes | When the log was processed by the system |
| status | ENUM | No | Processing status (pending/processed/failed) |
| created_at | TIMESTAMPTZ | No | When the record was created |
| updated_at | TIMESTAMPTZ | No | Last update timestamp |

#### Indexes

```sql
-- Performance indexes for common queries
CREATE INDEX idx_normalized_logs_timestamp 
    ON normalized_logs(timestamp DESC);

CREATE INDEX idx_normalized_logs_source 
    ON normalized_logs(source);

CREATE INDEX idx_normalized_logs_event_type 
    ON normalized_logs(event_type);

CREATE INDEX idx_normalized_logs_severity 
    ON normalized_logs(severity);

CREATE INDEX idx_normalized_logs_status 
    ON normalized_logs(status);

CREATE INDEX idx_normalized_logs_created_at 
    ON normalized_logs(created_at DESC);

-- Composite index for time-range + severity queries
CREATE INDEX idx_normalized_logs_timestamp_severity 
    ON normalized_logs(timestamp DESC, severity);

-- JSONB GIN indexes for field searches
CREATE INDEX idx_normalized_logs_raw_data 
    ON normalized_logs USING GIN(raw_data);

CREATE INDEX idx_normalized_logs_normalized_data 
    ON normalized_logs USING GIN(normalized_data);
```

#### Example Data

```sql
INSERT INTO normalized_logs (
    id,
    timestamp,
    source,
    event_type,
    severity,
    raw_data,
    normalized_data,
    message,
    status
) VALUES (
    'a1b2c3d4-e5f6-4789-a012-b34567890abc',
    '2025-11-07 14:30:25+00',
    'evtx',
    'Security',
    'high',
    '{"EventID": 4625, "Computer": "SERVER01", "TargetUserName": "admin", "IpAddress": "192.168.1.100"}',
    '{"event_id": 4625, "host": "SERVER01", "user": "admin", "source_ip": "192.168.1.100", "action": "login_attempt", "result": "failure"}',
    'Failed login attempt for user admin from 192.168.1.100',
    'processed'
);
```

---

### 2. ingest_jobs

Tracks file ingestion jobs for background processing.

#### Schema Definition

```sql
CREATE TABLE ingest_jobs (
    id VARCHAR(26) PRIMARY KEY,
    source VARCHAR(50) NOT NULL,
    filename VARCHAR(255) NOT NULL,
    path VARCHAR(1024) NOT NULL,
    status VARCHAR(50) NOT NULL,
    inserted INTEGER DEFAULT 0,
    skipped INTEGER DEFAULT 0,
    error TEXT,
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    started_at TIMESTAMP WITH TIME ZONE,
    finished_at TIMESTAMP WITH TIME ZONE,
    
    CONSTRAINT chk_status CHECK (status IN ('queued', 'running', 'done', 'error'))
);
```

#### Column Descriptions

| Column | Type | Nullable | Description |
|--------|------|----------|-------------|
| id | VARCHAR(26) | No | ULID identifier for the job |
| source | VARCHAR(50) | No | Source type (evtx, jsonl, syslog, etc.) |
| filename | VARCHAR(255) | No | Original uploaded filename |
| path | VARCHAR(1024) | No | Relative path to stored file |
| status | VARCHAR(50) | No | Job status (queued/running/done/error) |
| inserted | INTEGER | No | Number of records successfully inserted |
| skipped | INTEGER | No | Number of records skipped |
| error | TEXT | Yes | Error message if status is 'error' |
| created_at | TIMESTAMPTZ | No | When the job was created |
| started_at | TIMESTAMPTZ | Yes | When processing started |
| finished_at | TIMESTAMPTZ | Yes | When processing completed |

#### Indexes

```sql
-- Index for worker polling (queued jobs only)
CREATE INDEX idx_ingest_jobs_status_created 
    ON ingest_jobs(status, created_at) 
    WHERE status = 'queued';

-- Index for status queries
CREATE INDEX idx_ingest_jobs_created_at 
    ON ingest_jobs(created_at DESC);

-- Index for source-based queries
CREATE INDEX idx_ingest_jobs_source 
    ON ingest_jobs(source);
```

#### Example Data

```sql
INSERT INTO ingest_jobs (
    id,
    source,
    filename,
    path,
    status,
    inserted,
    created_at
) VALUES (
    '01HXXXXXXXXXXXXXXXXXXX',
    'evtx',
    'security-logs.evtx',
    'uploads/20251107/01HXXXXXXXXXXXXXXXXXXX.evtx',
    'queued',
    0,
    NOW()
);
```

---

## Enum Types

### enum_normalized_logs_severity

Represents the severity level of log events.

```sql
CREATE TYPE enum_normalized_logs_severity AS ENUM (
    'critical',  -- System failures, security breaches
    'high',      -- Errors requiring attention
    'medium',    -- Warnings
    'low',       -- Notices, minor issues
    'info'       -- Informational events
);
```

**Usage in C#**:
```csharp
public enum SeverityEnum
{
    Info,
    Low,
    Medium,
    High,
    Critical
}
```

### enum_normalized_logs_status

Represents the processing status of log entries.

```sql
CREATE TYPE enum_normalized_logs_status AS ENUM (
    'pending',    -- Not yet analyzed
    'processed',  -- Successfully analyzed
    'failed'      -- Processing failed
);
```

**Usage in C#**:
```csharp
public enum LogStatusEnum
{
    Pending,
    Processed,
    Failed
}
```

---

## JSONB Storage

### raw_data Column

Stores the original log data exactly as received, enabling:
- Forensic analysis
- Reprocessing with updated logic
- Debugging transformation issues

**Example EVTX raw_data**:
```json
{
  "EventID": 4625,
  "TimeCreated": "2025-11-07T14:30:25.123Z",
  "Computer": "SERVER01",
  "Channel": "Security",
  "Level": 2,
  "Task": 12544,
  "Keywords": "0x8010000000000000",
  "TargetUserName": "admin",
  "LogonType": 3,
  "Status": "0xC000006D",
  "SubStatus": "0xC000006A",
  "IpAddress": "192.168.1.100",
  "IpPort": "52341"
}
```

### normalized_data Column

Stores standardized fields extracted from the raw data:

```json
{
  "event_id": 4625,
  "host": "SERVER01",
  "user": "admin",
  "source_ip": "192.168.1.100",
  "action": "login_attempt",
  "result": "failure",
  "logon_type": "network",
  "channel": "Security",
  "additional_fields": {
    "status": "0xC000006D",
    "sub_status": "0xC000006A",
    "port": "52341"
  }
}
```

### JSONB Query Examples

**Find logs with specific IP address**:
```sql
SELECT * FROM normalized_logs
WHERE normalized_data @> '{"source_ip": "192.168.1.100"}';
```

**Find all login failures**:
```sql
SELECT * FROM normalized_logs
WHERE normalized_data @> '{"action": "login_attempt", "result": "failure"}';
```

**Search within nested fields**:
```sql
SELECT * FROM normalized_logs
WHERE normalized_data->'additional_fields'->>'status' = '0xC000006D';
```

**Full-text search in JSONB**:
```sql
SELECT * FROM normalized_logs
WHERE raw_data::text LIKE '%admin%';
```

---

## Database Configuration

### Connection String Format

```
Host={hostname};Port={port};Database={database};Username={user};Password={password};[options]
```

### Example Configurations

**Development**:
```
Host=localhost;Port=5432;Database=Security;Username=postgres;Password=TestDb123;
```

**Production**:
```
Host=ai-security-db;Port=5432;Database=Security;Username=app_user;Password=SecurePass123;Pooling=true;Maximum Pool Size=20;Connection Lifetime=300;
```

**Docker**:
```
Host=ai-security-db;Port=5432;Database=Security;Username=postgres;Password=${DB_PASSWORD};
```

### Connection Pool Settings

```
Pooling=true;
Minimum Pool Size=5;
Maximum Pool Size=20;
Connection Lifetime=300;
Connection Idle Lifetime=60;
```

---

## Database Initialization

### Initialization Script

**Location**: `database/init/01-init-log-transformer.sql`

```sql
-- Create ingest_jobs table if it doesn't exist
CREATE TABLE IF NOT EXISTS ingest_jobs (
    id VARCHAR(26) PRIMARY KEY,
    source VARCHAR(50) NOT NULL,
    filename VARCHAR(255) NOT NULL,
    path VARCHAR(1024) NOT NULL,
    status VARCHAR(50) NOT NULL,
    inserted INTEGER DEFAULT 0,
    skipped INTEGER DEFAULT 0,
    error TEXT,
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    started_at TIMESTAMP WITH TIME ZONE,
    finished_at TIMESTAMP WITH TIME ZONE,
    
    CONSTRAINT chk_status CHECK (status IN ('queued', 'running', 'done', 'error'))
);

-- Create indexes
CREATE INDEX IF NOT EXISTS idx_ingest_jobs_status_created 
    ON ingest_jobs(status, created_at) 
    WHERE status = 'queued';

CREATE INDEX IF NOT EXISTS idx_ingest_jobs_created_at 
    ON ingest_jobs(created_at DESC);

CREATE INDEX IF NOT EXISTS idx_ingest_jobs_source 
    ON ingest_jobs(source);

-- Grant permissions
GRANT SELECT, INSERT, UPDATE ON ingest_jobs TO app_user;
```

### Application Startup Initialization

The application automatically ensures tables exist on startup:

```csharp
// In Program.cs
await DatabaseInitializer.EnsureIngestJobsTableExistsAsync(app.Services);
```

---

## Performance Optimization

### Query Optimization

**Time-Range Queries**:
```sql
-- Use timestamp index
SELECT * FROM normalized_logs
WHERE timestamp >= '2025-11-07 00:00:00+00'
  AND timestamp < '2025-11-08 00:00:00+00'
ORDER BY timestamp DESC;
```

**Severity Filtering**:
```sql
-- Use composite index
SELECT * FROM normalized_logs
WHERE timestamp >= NOW() - INTERVAL '24 hours'
  AND severity IN ('critical', 'high')
ORDER BY timestamp DESC;
```

**Source-Specific Queries**:
```sql
-- Use source index
SELECT COUNT(*), severity
FROM normalized_logs
WHERE source = 'evtx'
GROUP BY severity;
```

### Batch Insert Optimization

```sql
-- Prepared statement with multiple rows
INSERT INTO normalized_logs (id, timestamp, source, event_type, severity, raw_data, normalized_data, status)
VALUES 
    ($1, $2, $3, $4, $5, $6, $7, $8),
    ($9, $10, $11, $12, $13, $14, $15, $16),
    ...
;
```

### JSONB Indexing

```sql
-- GIN index for containment queries
CREATE INDEX idx_normalized_data_gin ON normalized_logs USING GIN(normalized_data);

-- Query using index
SELECT * FROM normalized_logs
WHERE normalized_data @> '{"user": "admin"}';
```

---

## Partitioning Strategy (Future)

### Time-Based Partitioning

For high-volume deployments, implement partitioning by timestamp:

```sql
-- Create partitioned table
CREATE TABLE normalized_logs (
    id UUID NOT NULL,
    timestamp TIMESTAMP WITH TIME ZONE NOT NULL,
    -- ... other columns
) PARTITION BY RANGE (timestamp);

-- Create partitions for each month
CREATE TABLE normalized_logs_2025_11 
    PARTITION OF normalized_logs
    FOR VALUES FROM ('2025-11-01') TO ('2025-12-01');

CREATE TABLE normalized_logs_2025_12 
    PARTITION OF normalized_logs
    FOR VALUES FROM ('2025-12-01') TO ('2026-01-01');
```

### Benefits of Partitioning

- **Faster Queries**: Query only relevant partitions
- **Easier Maintenance**: Drop old partitions instead of DELETE
- **Better Performance**: Smaller indexes per partition
- **Archival**: Move old partitions to archive storage

---

## Backup and Recovery

### Backup Strategy

**Daily Full Backups**:
```bash
pg_dump -U postgres -d Security -F c -f backup_$(date +%Y%m%d).dump
```

**Continuous Archiving**:
```bash
# Enable WAL archiving in postgresql.conf
wal_level = replica
archive_mode = on
archive_command = 'cp %p /backup/archive/%f'
```

### Point-in-Time Recovery

```bash
# Restore base backup
pg_restore -U postgres -d Security -F c backup_20251107.dump

# Recover to specific time
recovery_target_time = '2025-11-07 14:30:00'
```

### Disaster Recovery

1. Regular automated backups
2. Off-site backup storage
3. Test recovery procedures
4. Document recovery steps
5. Monitor backup success

---

## Monitoring and Maintenance

### Key Metrics to Monitor

- **Table Size**: `pg_total_relation_size('normalized_logs')`
- **Index Usage**: `pg_stat_user_indexes`
- **Query Performance**: `pg_stat_statements`
- **Connection Count**: `pg_stat_activity`
- **Bloat**: `pgstattuple`

### Maintenance Tasks

**Weekly**:
```sql
-- Analyze tables for query planner
ANALYZE normalized_logs;
ANALYZE ingest_jobs;
```

**Monthly**:
```sql
-- Vacuum to reclaim space
VACUUM ANALYZE normalized_logs;

-- Reindex if needed
REINDEX TABLE normalized_logs;
```

### Health Check Queries

**Check for long-running queries**:
```sql
SELECT pid, now() - query_start AS duration, query
FROM pg_stat_activity
WHERE state = 'active'
  AND now() - query_start > INTERVAL '5 minutes';
```

**Check database size**:
```sql
SELECT pg_size_pretty(pg_database_size('Security'));
```

**Check table sizes**:
```sql
SELECT 
    schemaname,
    tablename,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) AS size
FROM pg_tables
WHERE schemaname = 'public'
ORDER BY pg_total_relation_size(schemaname||'.'||tablename) DESC;
```

---

## Security Considerations

### Access Control

```sql
-- Create application user with limited privileges
CREATE USER app_user WITH PASSWORD 'secure_password';

-- Grant only necessary permissions
GRANT SELECT, INSERT ON normalized_logs TO app_user;
GRANT SELECT, INSERT, UPDATE ON ingest_jobs TO app_user;
GRANT USAGE ON SCHEMA public TO app_user;
```

### Encryption

- **In Transit**: Use SSL/TLS connections
- **At Rest**: Enable PostgreSQL tablespace encryption
- **Credentials**: Store passwords in secrets management

### Audit Logging

Enable audit logging for compliance:

```sql
-- Install pgaudit extension
CREATE EXTENSION pgaudit;

-- Configure audit settings
ALTER SYSTEM SET pgaudit.log = 'ddl, write';
ALTER SYSTEM SET pgaudit.log_catalog = off;
```
