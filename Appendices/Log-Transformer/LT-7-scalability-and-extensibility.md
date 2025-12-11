# Appendix F-2.8 — Scalability and Extensibility

## Overview

Log-Transformer is architected from the ground up for scalability and extensibility. The plugin-based design, stateless architecture, and modular components enable the system to grow from handling thousands to millions of logs per day, while easily adapting to new log sources and formats.

## Extensibility

### Adding New Log Sources

The parser system uses a plugin architecture that makes adding new sources trivial.

#### Step 1: Implement the Parser Interface

```csharp
public class CustomSourceParser : ILogIngestParser
{
    public string SourceSystem => "custom-source";
    
    public async IAsyncEnumerable<NormalizedLog> ParseAsync(
        IngestContext context,
        [EnumeratorCancellation] CancellationToken ct = default)
    {
        // Your parsing logic here
        await foreach (var log in ParseSourceDataAsync(context.InputPath, ct))
        {
            yield return log;
        }
    }
    
    private async IAsyncEnumerable<NormalizedLog> ParseSourceDataAsync(
        string path,
        [EnumeratorCancellation] CancellationToken ct)
    {
        // Implementation specific to your log format
        // Read file, parse entries, convert to NormalizedLog
    }
}
```

#### Step 2: Register the Parser

```csharp
// In IngestWorker.cs - BuildRegistry method
private SourceAdapterRegistry BuildRegistry()
{
    var parsers = new List<ILogIngestParser>
    {
        new EvtxIngestParser(),
        new JsonlIngestParser(),
        new SyslogParser(),          // Add your parser
        new CustomSourceParser(),     // Add your parser
    };
    return new SourceAdapterRegistry(parsers);
}
```

#### Step 3: Configure Source Detection

```csharp
// In SourceAdapterRegistry.cs constructor
switch (parser.SourceSystem)
{
    case "jsonl":
        _byExtension[".jsonl"] = parser;
        break;
    case "windows":
        _byExtension[".evtx"] = parser;
        break;
    case "custom-source":
        _byExtension[".custom"] = parser;  // Add your extension
        break;
}
```

**That's it!** The new source is now fully integrated.

---

### Real-World Examples

#### Example 1: FluentBit Integration

FluentBit sends logs via HTTP POST in JSON format.

**1. Create FluentBit Parser**:
```csharp
public class FluentBitParser : ILogIngestParser
{
    public string SourceSystem => "fluentbit";
    
    public async IAsyncEnumerable<NormalizedLog> ParseAsync(
        IngestContext context,
        [EnumeratorCancellation] CancellationToken ct = default)
    {
        // FluentBit sends JSON array of log entries
        var json = await File.ReadAllTextAsync(context.InputPath, ct);
        var entries = JsonSerializer.Deserialize<FluentBitEntry[]>(json);
        
        foreach (var entry in entries)
        {
            yield return new NormalizedLog
            {
                Id = Guid.NewGuid().ToString(),
                Timestamp = entry.Timestamp,
                Source = "fluentbit",
                EventType = entry.Tag ?? "unknown",
                Severity = MapFluentBitSeverity(entry.Level),
                RawData = JsonSerializer.Serialize(entry),
                NormalizedData = JsonSerializer.Serialize(new NormalizedData
                {
                    host = entry.Host,
                    message = entry.Log,
                    process = entry.Container,
                    additional_fields = entry.Metadata
                }),
                Status = LogStatusEnum.Pending,
                CreatedAt = DateTime.UtcNow,
                UpdatedAt = DateTime.UtcNow
            };
        }
    }
    
    private SeverityEnum MapFluentBitSeverity(string level)
    {
        return level?.ToLower() switch
        {
            "error" => SeverityEnum.High,
            "warn" or "warning" => SeverityEnum.Medium,
            "info" => SeverityEnum.Info,
            "debug" => SeverityEnum.Info,
            _ => SeverityEnum.Info
        };
    }
}
```

**2. Add Real-Time HTTP Endpoint**:
```csharp
// In Program.cs
app.MapPost("/ingest/fluentbit", async (
    FluentBitPayload payload,
    AppDbContext dbContext,
    BatchWriter batchWriter,
    CancellationToken ct) =>
{
    var logs = new List<NormalizedLog>();
    
    foreach (var entry in payload.Records)
    {
        logs.Add(new NormalizedLog
        {
            Id = Guid.NewGuid().ToString(),
            Timestamp = entry.Timestamp,
            Source = "fluentbit",
            EventType = payload.Tag ?? "unknown",
            Severity = MapSeverity(entry.Level),
            RawData = JsonSerializer.Serialize(entry),
            NormalizedData = JsonSerializer.Serialize(ExtractFields(entry)),
            Status = LogStatusEnum.Pending,
            CreatedAt = DateTime.UtcNow
        });
    }
    
    await batchWriter.WriteAsync(logs, ct);
    
    return Results.Ok(new { inserted = logs.Count });
});
```

**3. Configure FluentBit to Send Logs**:
```conf
[OUTPUT]
    Name  http
    Match *
    Host  log-transformer-api
    Port  5001
    URI   /ingest/fluentbit
    Format json
```

---

#### Example 2: Wazuh Alert Integration

**1. Create Wazuh Parser**:
```csharp
public class WazuhParser : ILogIngestParser
{
    public string SourceSystem => "wazuh";
    
    public async IAsyncEnumerable<NormalizedLog> ParseAsync(
        IngestContext context,
        [EnumeratorCancellation] CancellationToken ct = default)
    {
        await using var stream = File.OpenRead(context.InputPath);
        using var reader = new StreamReader(stream);
        
        while (!reader.EndOfStream)
        {
            ct.ThrowIfCancellationRequested();
            
            var line = await reader.ReadLineAsync();
            if (string.IsNullOrWhiteSpace(line)) continue;
            
            var alert = JsonSerializer.Deserialize<WazuhAlert>(line);
            
            yield return new NormalizedLog
            {
                Id = Guid.NewGuid().ToString(),
                Timestamp = alert.Timestamp,
                Source = "wazuh",
                EventType = alert.Rule.Description,
                Severity = MapWazuhLevel(alert.Rule.Level),
                RawData = line,
                NormalizedData = JsonSerializer.Serialize(new NormalizedData
                {
                    source_ip = alert.Data?.SrcIp,
                    destination_ip = alert.Data?.DstIp,
                    user = alert.Data?.DstUser,
                    host = alert.Agent.Name,
                    action = alert.Rule.Description,
                    result = alert.Rule.Level >= 7 ? "failure" : "success",
                    event_id = alert.Rule.Id,
                    additional_fields = new Dictionary<string, string>
                    {
                        ["rule_level"] = alert.Rule.Level.ToString(),
                        ["rule_groups"] = string.Join(",", alert.Rule.Groups),
                        ["agent_id"] = alert.Agent.Id
                    }
                }),
                Status = LogStatusEnum.Pending,
                CreatedAt = DateTime.UtcNow
            };
        }
    }
    
    private SeverityEnum MapWazuhLevel(int level)
    {
        return level switch
        {
            >= 12 => SeverityEnum.Critical,
            >= 7 => SeverityEnum.High,
            >= 4 => SeverityEnum.Medium,
            >= 2 => SeverityEnum.Low,
            _ => SeverityEnum.Info
        };
    }
}
```

---

#### Example 3: Syslog Integration

**1. Create Syslog Parser**:
```csharp
public class SyslogParser : ILogIngestParser
{
    public string SourceSystem => "syslog";
    
    private readonly Regex _rfc3164Pattern = new Regex(
        @"^<(\d+)>(\w+\s+\d+\s+\d+:\d+:\d+)\s+(\S+)\s+(.+)$"
    );
    
    public async IAsyncEnumerable<NormalizedLog> ParseAsync(
        IngestContext context,
        [EnumeratorCancellation] CancellationToken ct = default)
    {
        await using var stream = File.OpenRead(context.InputPath);
        using var reader = new StreamReader(stream);
        
        while (!reader.EndOfStream)
        {
            ct.ThrowIfCancellationRequested();
            
            var line = await reader.ReadLineAsync();
            if (string.IsNullOrWhiteSpace(line)) continue;
            
            var match = _rfc3164Pattern.Match(line);
            if (!match.Success) continue;
            
            var priority = int.Parse(match.Groups[1].Value);
            var severity = priority & 0x07;
            var facility = priority >> 3;
            var timestamp = DateTime.Parse(match.Groups[2].Value);
            var hostname = match.Groups[3].Value;
            var message = match.Groups[4].Value;
            
            yield return new NormalizedLog
            {
                Id = Guid.NewGuid().ToString(),
                Timestamp = timestamp,
                Source = "syslog",
                EventType = GetFacilityName(facility),
                Severity = MapSyslogSeverity(severity),
                RawData = JsonSerializer.Serialize(new { line }),
                NormalizedData = JsonSerializer.Serialize(new NormalizedData
                {
                    host = hostname,
                    message = message,
                    additional_fields = new Dictionary<string, string>
                    {
                        ["facility"] = facility.ToString(),
                        ["severity"] = severity.ToString(),
                        ["priority"] = priority.ToString()
                    }
                }),
                Message = message,
                Status = LogStatusEnum.Pending,
                CreatedAt = DateTime.UtcNow
            };
        }
    }
    
    private SeverityEnum MapSyslogSeverity(int severity)
    {
        return severity switch
        {
            0 or 1 => SeverityEnum.Critical,  // Emergency, Alert
            2 => SeverityEnum.Critical,        // Critical
            3 => SeverityEnum.High,            // Error
            4 => SeverityEnum.Medium,          // Warning
            5 => SeverityEnum.Low,             // Notice
            6 => SeverityEnum.Info,            // Informational
            7 => SeverityEnum.Info,            // Debug
            _ => SeverityEnum.Info
        };
    }
}
```

---

### Additional Source Examples

The following sources can be added with similar patterns:

| Source | Complexity | Time Estimate | Description |
|--------|------------|---------------|-------------|
| **CloudWatch Logs** | Easy | 2-4 hours | AWS log aggregation service |
| **Azure Monitor** | Easy | 2-4 hours | Azure logging and monitoring |
| **Firewall Logs** | Medium | 4-8 hours | Various firewall formats (Palo Alto, Fortinet) |
| **IIS Logs** | Easy | 2-3 hours | W3C Extended Log Format |
| **Apache Logs** | Easy | 2-3 hours | Common/Combined log formats |
| **Nginx Logs** | Easy | 2-3 hours | Access and error logs |
| **SQL Server Audit** | Medium | 4-6 hours | Database audit logs |
| **Active Directory** | Medium | 4-6 hours | Domain controller logs |
| **CEF/LEEF** | Medium | 3-5 hours | Common Event Format / Log Event Extended Format |
| **Kubernetes** | Medium | 4-6 hours | Container orchestration logs |
| **Elasticsearch** | Easy | 2-4 hours | Pull logs from ES |
| **Splunk** | Medium | 4-6 hours | Splunk forwarder integration |

---

## Scalability

### Horizontal Scaling

The stateless architecture enables running multiple instances for increased throughput.

#### API Layer Scaling

**Docker Compose Example**:
```yaml
services:
  log-transformer-api:
    image: log-transformer:latest
    deploy:
      replicas: 3  # Run 3 instances
    ports:
      - "5001-5003:8080"
    environment:
      - Postgres__ConnectionString=${DB_CONNECTION}
    networks:
      - ai-security-network
  
  load-balancer:
    image: nginx:latest
    ports:
      - "5001:80"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf
    depends_on:
      - log-transformer-api
```

**Nginx Load Balancer Config**:
```nginx
upstream log_transformer {
    least_conn;  # Route to least busy instance
    server log-transformer-api-1:8080;
    server log-transformer-api-2:8080;
    server log-transformer-api-3:8080;
}

server {
    listen 80;
    
    location / {
        proxy_pass http://log_transformer;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

#### Worker Scaling

Multiple workers can process jobs concurrently:

```yaml
services:
  log-transformer-worker:
    image: log-transformer:latest
    deploy:
      replicas: 5  # Run 5 worker instances
    environment:
      - Postgres__ConnectionString=${DB_CONNECTION}
      - Import__BatchSize=2000
      - Import__PollIntervalSeconds=3
    volumes:
      - ./uploads:/app/uploads
    networks:
      - ai-security-network
```

**Database Row-Level Locking** prevents duplicate processing:
```sql
-- Worker queries use FOR UPDATE SKIP LOCKED
SELECT * FROM ingest_jobs
WHERE status = 'queued'
ORDER BY created_at
LIMIT 1
FOR UPDATE SKIP LOCKED;
```

---

### Vertical Scaling

#### Memory Optimization

**Adjust batch size based on available memory**:
```json
{
  "Import": {
    "BatchSize": 5000  // Higher = more memory, faster throughput
  }
}
```

**Memory Calculation**:
```
Estimated Memory = BatchSize × Average Log Size × Safety Factor
                 = 5000 × 2 KB × 2
                 = ~20 MB per batch
```

#### CPU Optimization

**Parallel Processing** (future enhancement):
```csharp
await Parallel.ForEachAsync(jobs, async (job, ct) =>
{
    await ProcessJobAsync(job, ct);
});
```

#### I/O Optimization

**SSD Storage** for uploads directory:
- Faster file reads
- Better IOPS for concurrent operations

**Database Connection Pooling**:
```
Pooling=true;
Minimum Pool Size=10;
Maximum Pool Size=50;
Connection Lifetime=300;
```

---

### Database Scaling

#### Read Replicas

For query-heavy workloads:

```csharp
builder.Services.AddDbContext<ReadOnlyDbContext>(options =>
{
    options.UseNpgsql(readReplicaConnectionString);
});
```

#### Partitioning

For high-volume scenarios, partition by time:

```sql
-- Create partitioned table
CREATE TABLE normalized_logs (
    id UUID NOT NULL,
    timestamp TIMESTAMPTZ NOT NULL,
    -- ...
) PARTITION BY RANGE (timestamp);

-- Monthly partitions
CREATE TABLE normalized_logs_2025_11 PARTITION OF normalized_logs
    FOR VALUES FROM ('2025-11-01') TO ('2025-12-01');

CREATE TABLE normalized_logs_2025_12 PARTITION OF normalized_logs
    FOR VALUES FROM ('2025-12-01') TO ('2026-01-01');
```

**Benefits**:
- Query only relevant partitions (10-100x faster)
- Easy archival (drop old partitions)
- Better index performance

#### Indexing Strategy

```sql
-- Composite index for common query patterns
CREATE INDEX idx_timestamp_severity_source 
    ON normalized_logs(timestamp DESC, severity, source);

-- Partial index for active data
CREATE INDEX idx_recent_critical 
    ON normalized_logs(timestamp DESC)
    WHERE severity = 'critical' 
      AND timestamp > NOW() - INTERVAL '30 days';
```

---

### Performance Benchmarks

#### Single Instance Performance

| Metric | Value | Configuration |
|--------|-------|---------------|
| **Upload Throughput** | 20-50 MB/s | Network limited |
| **Parsing Rate** | 10K-50K events/sec | Format dependent |
| **Database Writes** | 5K-10K inserts/sec | BatchSize=1000 |
| **API Latency** | <100ms | File upload response |
| **Memory Usage** | 512MB-2GB | Per instance |
| **CPU Usage** | 0.5-2 cores | Per instance |

#### Multi-Instance Performance

| Configuration | Throughput | Latency | Notes |
|---------------|------------|---------|-------|
| **3 API + 1 Worker** | 150 MB/s | <100ms | Balanced |
| **3 API + 5 Workers** | 150 MB/s | <50ms | Fast processing |
| **5 API + 10 Workers** | 250 MB/s | <30ms | High volume |

---

### Scalability Patterns

#### Pattern 1: Geographic Distribution

Deploy instances in multiple regions:

```yaml
# US East Region
services:
  log-transformer-us-east:
    image: log-transformer:latest
    environment:
      - Postgres__ConnectionString=${US_EAST_DB}
      - Region=us-east

# EU Region
services:
  log-transformer-eu:
    image: log-transformer:latest
    environment:
      - Postgres__ConnectionString=${EU_DB}
      - Region=eu
```

#### Pattern 2: Source-Specific Instances

Dedicate instances to specific log sources:

```yaml
services:
  log-transformer-evtx:
    image: log-transformer:latest
    environment:
      - Import__Sources__Enabled=evtx
  
  log-transformer-syslog:
    image: log-transformer:latest
    environment:
      - Import__Sources__Enabled=syslog
```

#### Pattern 3: Hot/Warm/Cold Architecture

```sql
-- Hot: Recent data on fast storage
CREATE TABLE normalized_logs_hot PARTITION OF normalized_logs
    FOR VALUES FROM (NOW() - INTERVAL '7 days') TO (MAXVALUE)
    TABLESPACE fast_ssd;

-- Warm: 7-30 days on standard storage
CREATE TABLE normalized_logs_warm PARTITION OF normalized_logs
    FOR VALUES FROM (NOW() - INTERVAL '30 days') TO (NOW() - INTERVAL '7 days')
    TABLESPACE standard;

-- Cold: Archive to S3/Glacier
-- (Automated export job)
```

---

### Monitoring and Observability

#### Key Metrics

**System Metrics**:
- Request rate (requests/second)
- Error rate (errors/second)
- Latency (p50, p95, p99)
- Throughput (MB/second, events/second)

**Application Metrics**:
- Jobs queued
- Jobs processing
- Jobs completed
- Jobs failed
- Average processing time
- Batch write performance

**Infrastructure Metrics**:
- CPU utilization
- Memory usage
- Disk I/O
- Network bandwidth
- Database connections

#### Prometheus Metrics Example

```csharp
// Add to Program.cs
builder.Services.AddOpenTelemetry()
    .WithMetrics(metrics =>
    {
        metrics.AddPrometheusExporter();
        metrics.AddMeter("LogTransformer");
    });

// In worker
private readonly Meter _meter = new Meter("LogTransformer");
private readonly Counter<long> _jobsProcessed;
private readonly Histogram<double> _processingTime;

_jobsProcessed = _meter.CreateCounter<long>("jobs_processed_total");
_processingTime = _meter.CreateHistogram<double>("job_processing_seconds");

// Record metrics
_jobsProcessed.Add(1, new KeyValuePair<string, object>("status", "success"));
_processingTime.Record(duration.TotalSeconds);
```

---

### Bottleneck Identification and Solutions

| Bottleneck | Symptoms | Solution |
|------------|----------|----------|
| **Database Write** | High CPU on DB, slow inserts | Increase batch size, add write replicas |
| **File I/O** | High disk wait time | Use SSD, increase file cache |
| **Network** | High latency, slow uploads | Load balancer, CDN, regional deployment |
| **Parser CPU** | High CPU on workers | Add more workers, optimize parsing |
| **Memory** | OOM errors, swapping | Reduce batch size, add more RAM |
| **Connection Pool** | Connection timeouts | Increase pool size, optimize queries |

---

## Future Enhancements

### Planned Features for Scalability

1. **Message Queue Integration**
   - RabbitMQ/Kafka for reliable message delivery
   - Distributed job queue
   - Better fault tolerance

2. **Caching Layer**
   - Redis for frequently accessed data
   - Reduce database load
   - Faster status queries

3. **Auto-Scaling**
   - Kubernetes HPA (Horizontal Pod Autoscaler)
   - Scale based on queue depth
   - Cost optimization

4. **Stream Processing**
   - Apache Flink/Spark Streaming
   - Real-time analytics
   - Complex event processing

5. **Data Lake Integration**
   - S3/Azure Blob for long-term storage
   - Parquet format for analytics
   - Athena/Synapse queries

---

## Best Practices

### Development

1. **Test parsers independently** before integration
2. **Use streaming** (`IAsyncEnumerable`) for large files
3. **Implement cancellation token support** for graceful shutdown
4. **Validate input** before processing
5. **Log errors** with contextual information

### Deployment

1. **Start small** and scale gradually
2. **Monitor metrics** continuously
3. **Load test** before production
4. **Plan for failures** (circuit breakers, retries)
5. **Document** scaling decisions

### Operations

1. **Regular backups** of configuration and data
2. **Health checks** for all components
3. **Alerts** for critical issues
4. **Capacity planning** based on growth
5. **Performance testing** with production-like data

---

## Conclusion

Log-Transformer's architecture provides:

- **Extensibility**: Add new sources in hours, not weeks
- **Scalability**: Handle thousands to millions of logs per day
- **Reliability**: Fault-tolerant job processing
- **Performance**: Optimized for high throughput
- **Flexibility**: Adapt to changing requirements

The platform grows with your needs, from a single instance processing a few sources to a distributed system handling dozens of log sources at scale.
