# Appendix F-2.2 — Getting Started

## Overview

Log-Transformer is a universal log ingestion and normalization platform built on .NET 8.0. It accepts logs from multiple sources (file uploads, real-time streams, API endpoints) and transforms them into a standardized format for security analysis and monitoring.

## Prerequisites

### System Requirements
- **Operating System**: Windows, Linux, or macOS
- **.NET 8.0 SDK** or later
- **Docker** and **Docker Compose** (for containerized deployment)
- **PostgreSQL 16** or later (if running without Docker)

### Minimum Hardware
- **CPU**: 2 cores
- **RAM**: 4 GB
- **Disk Space**: 10 GB (varies based on log volume)

### Network Requirements
- Access to PostgreSQL database server
- Outbound connectivity for NuGet package restoration (development only)
- Inbound port availability (default: 5001 for API)

## Installation

### Option 1: Docker Deployment (Recommended)

Docker deployment is the easiest way to get started and ensures consistency across environments.

1. **Clone the repository**:
   ```bash
   git clone https://github.com/your-org/log-transformer.git
   cd log-transformer
   ```

2. **Navigate to the docker directory**:
   ```bash
   cd docker
   ```

3. **Start the services**:
   ```bash
   docker-compose up -d
   ```

4. **Verify the deployment**:
   ```bash
   docker-compose logs -f log-transformer-api
   ```

The API will be available at `http://localhost:5001`

### Option 2: Local Development Setup

For development or customization, you can run the application locally.

1. **Clone the repository**:
   ```bash
   git clone https://github.com/your-org/log-transformer.git
   cd log-transformer
   ```

2. **Restore dependencies**:
   ```bash
   cd src/Api
   dotnet restore
   ```

3. **Configure database connection** (see Configuration section):
   Edit `appsettings.Development.json`:
   ```json
   {
     "Postgres": {
       "ConnectionString": "Host=localhost;Port=5432;Database=Security;Username=postgres;Password=yourpassword;"
     }
   }
   ```

4. **Run the application**:
   ```bash
   dotnet run
   ```

The API will start on `http://localhost:5132` by default.

## Database Setup

### Using Docker (Automatic)

If you're using the Docker deployment with the ai-security project, the database is automatically configured. The Log-Transformer connects to the shared PostgreSQL instance.

### Manual Database Setup

1. **Create the database**:
   ```sql
   CREATE DATABASE Security;
   ```

2. **Run the initialization script**:
   ```bash
   psql -U postgres -d Security -f database/init/01-init-log-transformer.sql
   ```

   This creates:
   - `ingest_jobs` table for tracking ingestion jobs
   - Required indexes for performance

3. **Verify the setup**:
   ```sql
   \c Security
   \dt
   ```

   You should see `ingest_jobs` and `normalized_logs` tables.

### Schema Requirements

The Log-Transformer requires the `normalized_logs` table with the following structure:

```sql
CREATE TABLE normalized_logs (
    id UUID PRIMARY KEY,
    timestamp TIMESTAMP WITH TIME ZONE NOT NULL,
    source VARCHAR(255) NOT NULL,
    event_type VARCHAR(255) NOT NULL,
    severity enum_normalized_logs_severity NOT NULL,
    raw_data JSONB NOT NULL,
    normalized_data JSONB NOT NULL,
    message TEXT,
    processed_at TIMESTAMP WITH TIME ZONE,
    status enum_normalized_logs_status NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);
```

## Running the Application

### Development Mode

```bash
cd src/Api
dotnet run --environment Development
```

Features in development mode:
- Detailed error messages
- Swagger UI at `/swagger`
- Hot reload support
- Verbose logging

### Production Mode (Docker)

```bash
cd docker
docker-compose up -d
```

Features in production mode:
- Optimized performance
- Container isolation
- Automatic restarts
- Network integration with ai-security

### Verifying the Installation

1. **Check API health**:
   ```bash
   curl http://localhost:5001/swagger/index.html
   ```

2. **Upload a test file**:
   ```bash
   curl -X POST http://localhost:5001/ingest/evtx \
     -F "file=@/path/to/test-log.evtx"
   ```

3. **Check job status**:
   ```bash
   curl http://localhost:5001/ingest/jobs/{job_id}
   ```

## Quick Start Example

### 1. Upload a Log File

```bash
curl -X POST http://localhost:5001/ingest/evtx \
  -F "file=@security-logs.evtx" \
  -o response.json
```

Response:
```json
{
  "job_id": "01HXXX...",
  "status_url": "/ingest/jobs/01HXXX..."
}
```

### 2. Monitor Processing Status

```bash
curl http://localhost:5001/ingest/jobs/01HXXX...
```

Response:
```json
{
  "id": "01HXXX...",
  "source": "evtx",
  "filename": "security-logs.evtx",
  "status": "running",
  "inserted": 1250,
  "skipped": 0,
  "created_at": "2025-11-07T10:30:00Z",
  "started_at": "2025-11-07T10:30:05Z"
}
```

### 3. Query Normalized Data

Once processing is complete, query the data through your analysis platform (e.g., ai-security):

```sql
SELECT * FROM normalized_logs 
WHERE source = 'evtx' 
  AND severity IN ('critical', 'high')
ORDER BY timestamp DESC 
LIMIT 100;
```

## Integration with AI-Security Platform

The Log-Transformer is designed to work seamlessly with the ai-security analysis platform:

1. **Shared Network**: Both services run on the same Docker network
2. **Common Database**: Writes to the same PostgreSQL instance
3. **Standardized Schema**: Uses the same `normalized_logs` table
4. **Real-time Analysis**: Data is immediately available for analysis

### Integration Steps

1. Ensure both applications are on the same Docker network:
   ```yaml
   networks:
     - ai-security_ai-security-network
   ```

2. Configure the same database connection in both applications

3. Start both services:
   ```bash
   # Start ai-security
   cd ai-security
   docker-compose up -d
   
   # Start log-transformer
   cd log-transformer/docker
   docker-compose up -d
   ```

## Troubleshooting

### Common Issues

#### Database Connection Errors

**Problem**: `Npgsql.NpgsqlException: Connection refused`

**Solution**: 
- Verify PostgreSQL is running: `docker ps | grep postgres`
- Check connection string in configuration
- Ensure database network is accessible

#### Enum Type Errors

**Problem**: `column "severity" is of type enum_normalized_logs_severity but expression is of type character varying`

**Solution**:
- Verify PostgreSQL enums exist: `\dT+ enum_normalized_logs_*`
- Ensure ai-security database is initialized first
- Check that enum types are registered in `AppDbContext.cs`

#### File Upload Fails

**Problem**: File upload returns 500 error

**Solution**:
- Check upload directory permissions
- Verify `Storage__UploadPath` configuration
- Ensure sufficient disk space

#### Background Worker Not Processing

**Problem**: Jobs stuck in "queued" status

**Solution**:
- Check worker logs: `docker-compose logs -f log-transformer-api`
- Verify `Import__PollIntervalSeconds` configuration
- Ensure database is accessible from worker

### Viewing Logs

**Docker deployment**:
```bash
docker-compose logs -f log-transformer-api
```

**Local development**:
Logs are written to console and `logs/` directory.

### Getting Help

- **Documentation**: Check other documentation files in this directory
- **Issues**: Report bugs on GitHub Issues
- **Logs**: Always include relevant log output when reporting issues

## Next Steps

- [Configuration Guide](./3-configuration.md) - Learn about all configuration options
- [Architecture Overview](./2-architecture.md) - Understand the system design
- [API Reference](./1-api-reference.md) - Explore all available endpoints
- [Adding New Sources](./7-scalability-and-extensibility.md) - Extend the platform
