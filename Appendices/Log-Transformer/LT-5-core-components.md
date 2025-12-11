# Appendix F-2.6 — Core Components

## Overview

Log-Transformer is built on a modular architecture with clearly separated concerns. This document details each core component, its responsibilities, and how it interacts with other parts of the system.

## Component Diagram

```
┌─────────────────────────────────────────────────────────┐
│                    Core Components                       │
├─────────────────────────────────────────────────────────┤
│                                                          │
│  ┌────────────────┐    ┌─────────────────┐             │
│  │  API Layer     │───▶│  Job Queue      │             │
│  │  (Endpoints)   │    │  (Database)     │             │
│  └────────────────┘    └─────────────────┘             │
│           │                      │                      │
│           │                      ▼                      │
│           │            ┌─────────────────┐             │
│           │            │ Background      │             │
│           │            │ Worker          │             │
│           │            └────────┬────────┘             │
│           │                     │                      │
│           ▼                     ▼                      │
│  ┌────────────────┐    ┌─────────────────┐             │
│  │  Real-time     │───▶│ Parser Registry │             │
│  │  Processor     │    │ (Plugin System) │             │
│  └────────────────┘    └────────┬────────┘             │
│                                  │                      │
│                                  ▼                      │
│                        ┌─────────────────┐             │
│                        │ Normalizer      │             │
│                        └────────┬────────┘             │
│                                  │                      │
│                                  ▼                      │
│                        ┌─────────────────┐             │
│                        │ Batch Writer    │             │
│                        └────────┬────────┘             │
│                                  │                      │
│                                  ▼                      │
│                        ┌─────────────────┐             │
│                        │ Data Layer      │             │
│                        │ (EF Core)       │             │
│                        └─────────────────┘             │
│                                                          │
└─────────────────────────────────────────────────────────┘
```

## 1. API Layer

### Purpose
Provides HTTP endpoints for external systems to interact with the log ingestion platform.

### Technology
- ASP.NET Core 8.0 Minimal APIs
- Swagger/OpenAPI for documentation

### Key Files
- `src/Api/Program.cs` - Application entry point and service configuration
- `src/Api/Features/Ingest/EvtxUploadEndpoint.cs` - File upload endpoint
- `src/Api/Features/Ingest/JobStatusEndpoint.cs` - Job status query endpoint

### Responsibilities

#### File Upload Handling
- Accept multipart/form-data file uploads
- Validate file types and sizes
- Store files in organized directory structure
- Create job records for background processing
- Return job IDs for status tracking

#### Job Status Queries
- Retrieve job information by ID
- Provide real-time progress updates
- Report errors and completion status

#### API Documentation
- Serve OpenAPI/Swagger documentation
- Interactive API testing interface
- Schema definitions

### Implementation Details

**Upload Endpoint**:
```csharp
public static void MapEvtxUploadEndpoint(this IEndpointRouteBuilder endpoints)
{
    endpoints.MapPost("/ingest/evtx", HandleAsync)
             .Accepts<IFormFile>("multipart/form-data")
             .Produces(StatusCodes.Status202Accepted)
             .Produces(StatusCodes.Status400BadRequest)
             .DisableAntiforgery();
}
```

**Key Features**:
- Asynchronous file handling
- Atomic file operations
- Transaction-based job creation
- Error handling and cleanup
- ULID-based unique identifiers

**File Organization**:
```
uploads/
  └── 20251107/           # Date-based partitioning
      ├── {ULID}.evtx     # Unique file identifiers
      ├── {ULID}.jsonl
      └── ...
```

### Configuration
```json
{
  "Storage": {
    "UploadPath": "uploads"
  }
}
```

---

## 2. Job Queue System

### Purpose
Decouples file upload from processing to provide asynchronous, fault-tolerant log ingestion.

### Technology
- PostgreSQL table-based queue
- Entity Framework Core for data access

### Key Files
- `src/Shared/Models/IngestJob.cs` - Job entity model
- `database/init/01-init-log-transformer.sql` - Job table schema

### Responsibilities

#### Job Lifecycle Management
- Create new jobs on file upload
- Track job status transitions
- Store processing statistics
- Record error information

#### Status Tracking
- `queued` - Job created, waiting for processing
- `running` - Currently being processed
- `done` - Successfully completed
- `error` - Failed with error message

### Schema

```sql
CREATE TABLE ingest_jobs (
    id VARCHAR(26) PRIMARY KEY,           -- ULID identifier
    source VARCHAR(50) NOT NULL,          -- Source type (evtx, jsonl, etc.)
    filename VARCHAR(255) NOT NULL,       -- Original filename
    path VARCHAR(1024) NOT NULL,          -- Relative storage path
    status VARCHAR(50) NOT NULL,          -- Current status
    inserted INTEGER DEFAULT 0,           -- Records inserted
    skipped INTEGER DEFAULT 0,            -- Records skipped
    error TEXT,                           -- Error message if failed
    created_at TIMESTAMP NOT NULL,        -- Job creation time
    started_at TIMESTAMP,                 -- Processing start time
    finished_at TIMESTAMP,                -- Processing end time
    
    CHECK (status IN ('queued', 'running', 'done', 'error'))
);

CREATE INDEX idx_ingest_jobs_status_created 
    ON ingest_jobs(status, created_at) 
    WHERE status = 'queued';
```

### Job Model

```csharp
public class IngestJob
{
    public string Id { get; set; }
    public string Source { get; set; }
    public string Filename { get; set; }
    public string Path { get; set; }
    public string Status { get; set; }
    public int Inserted { get; set; }
    public int Skipped { get; set; }
    public string? Error { get; set; }
    public DateTime CreatedAt { get; set; }
    public DateTime? StartedAt { get; set; }
    public DateTime? FinishedAt { get; set; }
}
```

---

## 3. Background Worker (IngestWorker)

### Purpose
Continuously processes queued ingestion jobs in the background.

### Technology
- .NET `BackgroundService` (Hosted Service)
- Runs as long-lived background task

### Key Files
- `src/Api/Features/Ingest/IngestWorker.cs` - Main worker implementation

### Responsibilities

#### Job Processing Loop
1. Poll database for queued jobs
2. Claim oldest job (update to "running")
3. Resolve appropriate parser
4. Process log file
5. Update job status and statistics
6. Repeat indefinitely

#### Error Handling
- Graceful error recovery
- Detailed error logging
- Job marked as "error" with message
- Continues processing other jobs

#### Resource Management
- Configurable poll interval
- Scoped service lifetime
- Proper disposal of resources
- Cancellation token support

### Implementation

```csharp
public class IngestWorker : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            try
            {
                await TryProcessNextJobAsync(stoppingToken);
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Unhandled error in ingest worker.");
            }
            
            await Task.Delay(pollInterval, stoppingToken);
        }
    }
}
```

### Processing Flow

```
1. Query for queued jobs
2. Lock job (update to running)
3. Load file from storage
4. Resolve parser by extension/source
5. Stream parse file
6. Batch normalize records
7. Bulk insert to database
8. Update job statistics
9. Mark job as done/error
```

### Configuration
```json
{
  "Import": {
    "PollIntervalSeconds": 5
  }
}
```

---

## 4. Parser Registry

### Purpose
Plugin system for extensible log format support.

### Technology
- Interface-based polymorphism
- Factory pattern for parser resolution

### Key Files
- `src/Api/Features/Ingest/Parsers/ILogIngestParser.cs` - Parser interface
- `src/Api/Features/Ingest/Parsers/SourceAdapterRegistry.cs` - Registry implementation
- `src/Api/Features/Ingest/Parsers/EvtxIngestParser.cs` - EVTX parser
- `src/Api/Features/Ingest/Parsers/JsonlIngestParser.cs` - JSONL parser

### Responsibilities

#### Parser Resolution
- Resolve by file extension (`.evtx`, `.jsonl`)
- Resolve by source hint (`evtx`, `jsonl`)
- Fallback to default parser

#### Parser Management
- Register parsers at startup
- Maintain parser catalog
- Support custom parsers

### Parser Interface

```csharp
public interface ILogIngestParser
{
    /// <summary>
    /// Logical source system identifier
    /// </summary>
    string SourceSystem { get; }
    
    /// <summary>
    /// Parse input asynchronously producing normalized logs
    /// </summary>
    IAsyncEnumerable<NormalizedLog> ParseAsync(
        IngestContext context, 
        CancellationToken cancellationToken = default
    );
}
```

### Registry Implementation

```csharp
public class SourceAdapterRegistry
{
    private readonly Dictionary<string, ILogIngestParser> _byExtension;
    private readonly Dictionary<string, ILogIngestParser> _bySource;
    
    public ILogIngestParser? Resolve(string filePath, string? hintedSource = null)
    {
        // Try source hint first
        if (!string.IsNullOrWhiteSpace(hintedSource) && 
            _bySource.TryGetValue(hintedSource, out var byHint))
            return byHint;
        
        // Fall back to extension
        var ext = Path.GetExtension(filePath);
        if (_byExtension.TryGetValue(ext, out var parser))
            return parser;
        
        return null;
    }
}
```

### Built-in Parsers

#### EVTX Parser
- **Source**: `windows`
- **Extensions**: `.evtx`
- **Platform**: Windows (native) and Linux (library-based)
- **Features**: Streaming, event filtering, field extraction

#### JSONL Parser
- **Source**: `jsonl`
- **Extensions**: `.jsonl`
- **Platform**: Cross-platform
- **Features**: Line-by-line streaming, flexible schema

### Adding Custom Parsers

```csharp
public class SyslogParser : ILogIngestParser
{
    public string SourceSystem => "syslog";
    
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
            
            var log = ParseSyslogLine(line);
            yield return log;
        }
    }
    
    private NormalizedLog ParseSyslogLine(string line)
    {
        // Parse RFC3164 or RFC5424 format
        // Extract timestamp, hostname, facility, severity, message
        // Return normalized log
    }
}
```

**Register the parser**:
```csharp
// In IngestWorker.cs
var parsers = new List<ILogIngestParser>
{
    new EvtxIngestParser(),
    new JsonlIngestParser(),
    new SyslogParser(),      // Add new parser
};
var registry = new SourceAdapterRegistry(parsers);
```

---

## 5. Normalizer

### Purpose
Transform diverse log formats into a unified schema for analysis.

### Technology
- Pure C# transformation logic
- JSON serialization for flexible data storage

### Key Files
- `src/Api/Features/Ingest/Normalizer.cs` - Normalization logic
- `src/Shared/Models/NormalizedData.cs` - Normalized schema

### Responsibilities

#### Field Extraction
Extract common fields from various log formats:
- Source/destination IPs
- User names
- Hostnames
- Process names
- File paths
- Actions and results

#### Severity Mapping
Convert various severity systems to standard enum:
```csharp
public enum SeverityEnum
{
    Info,       // Informational events
    Low,        // Low-priority events
    Medium,     // Warnings
    High,       // Errors
    Critical    // Critical failures
}
```

**Mapping Examples**:

*Windows Event Logs*:
```
0 (LogAlways)    → info
1 (Critical)     → critical
2 (Error)        → high
3 (Warning)      → medium
4 (Information)  → info
5 (Verbose)      → info
```

*Syslog*:
```
0-1 (Emerg/Alert) → critical
2 (Critical)      → critical
3 (Error)         → high
4 (Warning)       → medium
5-6 (Notice/Info) → low/info
7 (Debug)         → info
```

#### Timestamp Normalization
- Convert all timestamps to UTC
- Parse various timestamp formats
- Handle timezone conversions

#### Data Structuring
```csharp
public class NormalizedData
{
    public string? source_ip { get; set; }
    public string? destination_ip { get; set; }
    public string? user { get; set; }
    public string? host { get; set; }
    public string? process { get; set; }
    public string? file_path { get; set; }
    public string? action { get; set; }
    public string? result { get; set; }
    public string? channel { get; set; }
    public int? event_id { get; set; }
    public Dictionary<string, string> AdditionalFields { get; set; }
}
```

---

## 6. Batch Writer

### Purpose
Optimize database writes through bulk operations and transaction management.

### Technology
- Entity Framework Core bulk operations
- PostgreSQL batch inserts

### Key Files
- `src/Api/Features/Ingest/BatchWriter.cs` - Batch writing logic

### Responsibilities

#### Bulk Insert Operations
- Accumulate records into batches
- Execute bulk INSERT statements
- Minimize database round trips

#### Transaction Management
- Ensure atomicity of batch writes
- Rollback on errors
- Maintain data consistency

#### Performance Optimization
- Configurable batch sizes
- Connection pooling
- Prepared statements

### Implementation

```csharp
public class BatchWriter
{
    private readonly AppDbContext _context;
    
    public async Task<int> WriteAsync(
        List<NormalizedLog> logs, 
        CancellationToken ct)
    {
        if (logs.Count == 0) return 0;
        
        // Ensure timestamps are UTC
        foreach (var log in logs)
        {
            log.Timestamp = DateTime.SpecifyKind(
                log.Timestamp, 
                DateTimeKind.Utc
            );
            log.CreatedAt = DateTime.UtcNow;
            log.UpdatedAt = DateTime.UtcNow;
        }
        
        // Bulk insert
        _context.NormalizedLogs.AddRange(logs);
        await _context.SaveChangesAsync(ct);
        
        return logs.Count;
    }
}
```

### Configuration
```json
{
  "Import": {
    "BatchSize": 1000
  }
}
```

### Performance Characteristics

| Batch Size | Throughput | Memory | Latency |
|------------|------------|--------|---------|
| 100 | Low | Low | Low |
| 1000 | Medium | Medium | Medium |
| 5000 | High | High | High |

---

## 7. Data Layer (Entity Framework Core)

### Purpose
Provide object-relational mapping and database abstraction.

### Technology
- Entity Framework Core 8.0
- Npgsql PostgreSQL provider

### Key Files
- `src/Api/Data/AppDbContext.cs` - Database context
- `src/Shared/Models/NormalizedLog.cs` - Log entity model
- `src/Shared/Models/IngestJob.cs` - Job entity model

### Responsibilities

#### Entity Mapping
- Map C# classes to database tables
- Handle type conversions
- Configure relationships

#### Query Generation
- Translate LINQ to SQL
- Optimize query performance
- Parameter binding

#### Schema Management
- Register PostgreSQL enums
- Configure indexes
- Define constraints

### Database Context

```csharp
public class AppDbContext : DbContext
{
    public DbSet<IngestJob> IngestJobs { get; set; }
    public DbSet<NormalizedLog> NormalizedLogs { get; set; }
    
    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        // Register PostgreSQL enums
        modelBuilder.HasPostgresEnum<SeverityEnum>("enum_normalized_logs_severity");
        modelBuilder.HasPostgresEnum<LogStatusEnum>("enum_normalized_logs_status");
        
        // Configure entities
        ConfigureIngestJobs(modelBuilder);
        ConfigureNormalizedLogs(modelBuilder);
    }
}
```

### Entity Configuration

```csharp
private void ConfigureNormalizedLogs(ModelBuilder modelBuilder)
{
    var entity = modelBuilder.Entity<NormalizedLog>();
    
    entity.ToTable("normalized_logs");
    entity.HasKey(log => log.Id);
    
    // Column mappings
    entity.Property(log => log.Id)
        .HasColumnName("id")
        .HasColumnType("uuid");
    
    entity.Property(log => log.Severity)
        .HasColumnName("severity")
        .HasColumnType("enum_normalized_logs_severity");
    
    entity.Property(log => log.RawData)
        .HasColumnName("raw_data")
        .HasColumnType("jsonb");
    
    // Indexes
    entity.HasIndex(log => log.Timestamp);
    entity.HasIndex(log => log.Source);
    entity.HasIndex(log => log.Severity);
}
```

---

## Component Interactions

### File Upload Flow

```
1. Client uploads file
   → API Endpoint receives request
   
2. API validates file
   → Check file type, size
   
3. API stores file
   → Save to uploads/{date}/{ulid}.{ext}
   
4. API creates job
   → Insert into ingest_jobs table
   
5. API returns job ID
   → Client can track status
```

### Background Processing Flow

```
1. Worker polls for jobs
   → Query ingest_jobs WHERE status='queued'
   
2. Worker claims job
   → UPDATE status='running'
   
3. Worker resolves parser
   → Registry.Resolve(filePath, source)
   
4. Parser streams records
   → IAsyncEnumerable<NormalizedLog>
   
5. Normalizer transforms data
   → Extract fields, map severity
   
6. Batch Writer inserts records
   → Bulk INSERT into normalized_logs
   
7. Worker updates job
   → UPDATE status='done', inserted=N
```

### Real-time Processing Flow

```
1. External system POSTs data
   → API endpoint receives payload
   
2. Normalizer transforms data
   → Direct transformation (no file)
   
3. Batch Writer inserts record
   → Immediate database write
   
4. API responds
   → Success/failure response
```

## Testing Components

Each component should be independently testable:

```csharp
// Test parser
var parser = new EvtxParser();
var context = new IngestContext { InputPath = "test.evtx", ... };
var logs = await parser.ParseAsync(context).ToListAsync();
Assert.Equal(expectedCount, logs.Count);

// Test normalizer
var raw = new RawLogData { ... };
var normalized = Normalizer.Normalize(raw);
Assert.Equal("critical", normalized.Severity);

// Test batch writer
var logs = new List<NormalizedLog> { ... };
var count = await batchWriter.WriteAsync(logs);
Assert.Equal(logs.Count, count);
```

## Component Dependencies

```
API Layer
  ├─ Job Queue (writes)
  └─ Data Layer (queries)

Background Worker
  ├─ Job Queue (reads/writes)
  ├─ Parser Registry (uses)
  ├─ Normalizer (uses)
  └─ Batch Writer (uses)

Parser Registry
  └─ Parsers (manages)

Batch Writer
  └─ Data Layer (uses)

Data Layer
  └─ PostgreSQL (connects)
```
