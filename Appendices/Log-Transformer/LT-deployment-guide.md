# Log-Transformer - Deployment Guide

Complete guide for setting up and running the Log-Transformer EVTX file processing API.

---

## Table of Contents

- [Overview](#overview)
- [System Requirements](#system-requirements)
- [Prerequisites](#prerequisites)
- [Quick Start (Docker - Recommended)](#quick-start-docker---recommended)
- [Local Development Setup](#local-development-setup)
- [Configuration](#configuration)
- [Running the Application](#running-the-application)
- [API Usage](#api-usage)
- [Integration with AI-Security](#integration-with-ai-security)
- [Troubleshooting](#troubleshooting)

---

## Overview

Log-Transformer is a .NET 8.0 API that processes Windows Event Log (EVTX) files and transforms them into normalized format compatible with the AI-Security platform. It provides:

- RESTful API for uploading EVTX files
- Background processing of log files
- Automatic normalization to AI-Security schema
- Job tracking and status monitoring

---

## System Requirements

### Minimum Requirements
- **CPU**: 2 cores
- **RAM**: 2 GB
- **Disk Space**: 5 GB free (more for large EVTX files)
- **OS**: Windows 10/11, Linux (Ubuntu 20.04+), macOS 10.15+

### Recommended Requirements
- **CPU**: 4+ cores
- **RAM**: 4+ GB
- **Disk Space**: 20+ GB free

---

## Prerequisites

### For Docker Deployment (Recommended)

1. **Docker Desktop** (includes Docker Engine + Docker Compose)
   - Windows: [Download Docker Desktop](https://www.docker.com/products/docker-desktop)
   - macOS: [Download Docker Desktop](https://www.docker.com/products/docker-desktop)
   - Linux: Install Docker Engine + Docker Compose separately
   
   ```bash
   # Verify installation
   docker --version        # Should show Docker version 20.10+
   docker-compose --version # Should show version 2.0+
   ```

2. **AI-Security Platform Running**
   - Log-Transformer requires the AI-Security database to be running
   - Follow the AI-Security deployment guide first

### For Local Development

1. **.NET 8.0 SDK**
   ```bash
   dotnet --version  # Should show 8.0.x
   ```
   Download from: https://dotnet.microsoft.com/download/dotnet/8.0

2. **PostgreSQL 15+** (or access to AI-Security database)
   ```bash
   psql --version  # Should show PostgreSQL 15+
   ```

3. **Git**
   ```bash
   git --version
   ```

---

## Quick Start (Docker - Recommended)

### Prerequisites
**IMPORTANT**: AI-Security platform must be running before starting Log-Transformer, as it shares the same database.

```bash
# Start AI-Security first
cd ai-security
docker-compose up -d

# Verify AI-Security database is running
docker ps | grep ai-security-db
```

### Step 1: Clone the Repository

```bash
git clone <repository-url>
cd Log-Transformer
```

### Step 2: Build and Start with Docker

```bash
cd docker

# Build and start the Log-Transformer API
docker-compose up --build -d
```

The Log-Transformer service will:
- Build the .NET application
- Connect to the AI-Security database
- Start the API on port 5001
- Create the uploads directory for file storage

### Step 3: Verify Service is Running

```bash
# Check service status
docker-compose ps

# Should show:
# log-transformer-api   Up   0.0.0.0:5001->8080/tcp

# View logs
docker-compose logs -f

# Test API health (if health endpoint exists)
curl http://localhost:5001/swagger
```

### Step 4: Access the Application

- **API Base URL**: http://localhost:5001
- **Swagger Documentation**: http://localhost:5001/swagger

### Docker Management Commands

```bash
# View logs
docker-compose logs -f log-transformer-api

# Stop service
docker-compose down

# Restart service
docker-compose restart

# Rebuild after code changes
docker-compose up --build -d

# Remove with volumes
docker-compose down -v
```

---

## Local Development Setup

For development without Docker.

### Step 1: Clone the Repository

```bash
git clone <repository-url>
cd Log-Transformer
```

### Step 2: Database Setup

Log-Transformer uses the same database as AI-Security. Ensure the database exists:

```bash
# Connect to PostgreSQL
psql -U postgres

# Verify Security database exists
\l Security

# If not, create it
CREATE DATABASE "Security";

# Exit
\q
```

### Step 3: Initialize Log-Transformer Tables

```bash
# Run the initialization script
psql -U postgres -d Security -f database/init/01-init-log-transformer.sql
```

This creates the `ingest_jobs` table for tracking file processing jobs.

### Step 4: Build the Application

```bash
cd src/Api

# Restore dependencies
dotnet restore

# Build the application
dotnet build

# Run (development mode)
dotnet run
```

The API will start on: http://localhost:5132 (default .NET dev port)

### Step 5: Verify Build

```bash
# Should show compilation success
# API should be listening on http://localhost:5132
```

---

## Configuration

### Environment Variables (Docker)

When running with Docker, configure in `docker/docker-compose.yml`:

```yaml
environment:
  # Database connection (connects to AI-Security database)
  - Postgres__ConnectionString=Host=ai-security-db;Port=5432;Database=Security;Username=postgres;Password=TestDb123
  
  # File storage location
  - Storage__UploadPath=/app/uploads
  
  # Processing configuration
  - Import__BatchSize=1000
  - Import__PollIntervalSeconds=5
  
  # ASP.NET Core configuration
  - ASPNETCORE_URLS=http://+:8080
  - ASPNETCORE_ENVIRONMENT=Production
```

### Configuration Files (Local Development)

#### appsettings.json

Located at: `src/Api/appsettings.json`

```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning"
    }
  },
  "AllowedHosts": "*",
  "Postgres": {
    "ConnectionString": "Host=localhost;Port=5432;Database=Security;Username=postgres;Password=TestDb123;"
  },
  "Storage": {
    "UploadPath": "./uploads"
  },
  "Import": {
    "BatchSize": 1000
  }
}
```

#### appsettings.Development.json

For development-specific settings:

```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Debug",
      "Microsoft.AspNetCore": "Information"
    }
  },
  "Postgres": {
    "ConnectionString": "Host=localhost;Port=5432;Database=Security;Username=postgres;Password=YourLocalPassword;"
  }
}
```

### Configuration Reference

| Setting | Description | Default | Required |
|---------|-------------|---------|----------|
| `Postgres__ConnectionString` | PostgreSQL connection string | - | ✅ |
| `Storage__UploadPath` | Directory for uploaded files | /app/uploads | ✅ |
| `Import__BatchSize` | Batch size for processing | 1000 | No |
| `Import__PollIntervalSeconds` | Job polling interval | 5 | No |
| `ASPNETCORE_URLS` | API listening URL | http://+:8080 | No |
| `ASPNETCORE_ENVIRONMENT` | Environment name | Production | No |

---

## Running the Application

### Docker Mode (Production-like)

```bash
# Ensure AI-Security is running first
cd ../ai-security
docker-compose up -d

# Start Log-Transformer
cd ../Log-Transformer/docker
docker-compose up -d

# View logs
docker-compose logs -f
```

### Local Development Mode

```bash
cd src/Api

# Run with hot reload
dotnet watch run

# Or standard run
dotnet run
```

The API starts on http://localhost:5132

### Running with Specific Configuration

```bash
# Use Development environment
dotnet run --environment Development

# Use Production configuration
dotnet run --environment Production
```

---

## API Usage

### Upload EVTX File

**Endpoint**: `POST /upload/evtx`

**cURL Example**:
```bash
curl -X POST http://localhost:5001/upload/evtx \
  -F "file=@path/to/your/file.evtx" \
  -H "Content-Type: multipart/form-data"
```

**Response**:
```json
{
  "jobId": "12345678-1234-1234-1234-123456789abc",
  "status": "queued",
  "message": "File uploaded successfully and queued for processing"
}
```

**PowerShell Example**:
```powershell
$filePath = "C:\Logs\Security.evtx"
$uri = "http://localhost:5001/upload/evtx"

$form = @{
    file = Get-Item -Path $filePath
}

Invoke-RestMethod -Uri $uri -Method Post -Form $form
```

### Check Job Status

**Endpoint**: `GET /jobs/{jobId}/status`

**cURL Example**:
```bash
curl http://localhost:5001/jobs/12345678-1234-1234-1234-123456789abc/status
```

**Response**:
```json
{
  "jobId": "12345678-1234-1234-1234-123456789abc",
  "status": "processing",
  "progress": 45,
  "message": "Processing 4500 of 10000 events",
  "createdAt": "2024-11-17T10:30:00Z",
  "completedAt": null
}
```

**Status Values**:
- `queued` - Job is waiting to be processed
- `processing` - Job is currently being processed
- `completed` - Job completed successfully
- `failed` - Job failed with errors

### API Documentation

Access interactive Swagger documentation:

**URL**: http://localhost:5001/swagger

This provides:
- Complete API endpoint documentation
- Request/response schemas
- Interactive API testing interface

---

## Integration with AI-Security

### Network Configuration

Log-Transformer connects to AI-Security's database via Docker network:

```yaml
networks:
  ai-security_ai-security-network:
    external: true
```

### Data Flow

1. **Upload**: User uploads EVTX file to Log-Transformer
2. **Processing**: Background worker processes file
3. **Normalization**: Events are normalized to AI-Security schema
4. **Storage**: Normalized logs are written to `normalized_logs` table
5. **Detection**: AI-Security detects anomalies in new logs

### Database Schema Compatibility

Log-Transformer writes to the same `normalized_logs` table used by AI-Security:

```sql
-- Table structure (managed by AI-Security)
CREATE TABLE normalized_logs (
    id UUID PRIMARY KEY,
    timestamp TIMESTAMP WITH TIME ZONE NOT NULL,
    source VARCHAR(255) NOT NULL,           -- 'evtx'
    event_type VARCHAR(255),                -- Channel name
    severity enum_normalized_logs_severity, -- 'critical', 'high', 'medium', 'low', 'info'
    raw_data JSONB,                         -- Original event data
    normalized_data JSONB,                  -- Normalized fields
    processed BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);
```

### Normalized Data Structure

Log-Transformer produces this structure in `normalized_data`:

```json
{
  "host": "DESKTOP-ABC123",
  "user": "DOMAIN\\username",
  "source_ip": "192.168.1.100",
  "destination_ip": null,
  "process": "services.exe",
  "file_path": "C:\\Windows\\System32\\services.exe",
  "action": "Process Started",
  "result": "Success",
  "channel": "Security",
  "event_id": 4688,
  "additional_fields": {
    "processId": "1234",
    "parentProcessId": "568",
    "tokenElevationType": "%%1936"
  }
}
```

---

## Verification & Testing

### 1. Check Service Health

```bash
# Docker
docker-compose ps

# Should show:
# log-transformer-api   Up (healthy)

# Check logs
docker-compose logs log-transformer-api
```

### 2. Test File Upload

```bash
# Using a sample EVTX file
curl -X POST http://localhost:5001/upload/evtx \
  -F "file=@./test-files/Security.evtx"

# Should return job ID and status
```

### 3. Verify Database Connection

```bash
# Connect to database
docker exec -it ai-security-db psql -U postgres -d Security

# Check if ingest_jobs table exists
\dt ingest_jobs

# Check for processed logs
SELECT COUNT(*) FROM normalized_logs WHERE source = 'evtx';

# Exit
\q
```

### 4. Check Uploaded Files

```bash
# Docker
docker exec -it log-transformer-api ls -la /app/uploads

# Local
ls -la uploads/
```

---

## Troubleshooting

### Common Issues

#### 1. Cannot Connect to Database

**Problem**: `Npgsql.NpgsqlException: Connection refused`

**Solutions**:
```bash
# Verify AI-Security database is running
docker ps | grep ai-security-db

# Check if database is accessible
docker exec -it ai-security-db psql -U postgres -c "\l"

# Verify connection string in docker-compose.yml
# Should be: Host=ai-security-db;Port=5432;...

# Check network connectivity
docker network ls
docker network inspect ai-security_ai-security-network
```

#### 2. Docker Network Not Found

**Problem**: `network ai-security_ai-security-network not found`

**Solution**:
```bash
# Start AI-Security first to create the network
cd ../ai-security
docker-compose up -d

# Verify network exists
docker network ls | grep ai-security

# Then start Log-Transformer
cd ../Log-Transformer/docker
docker-compose up -d
```

#### 3. Enum Type Compatibility Error

**Problem**: `column "severity" is of type enum_normalized_logs_severity but expression is of type character varying`

**Status**: Known issue being resolved

**Workaround**:
```sql
-- Temporarily cast severity values
-- Fix is being implemented in the application code
```

#### 4. File Upload Fails

**Problem**: Upload returns 400 or 500 error

**Check**:
```bash
# View API logs
docker-compose logs -f log-transformer-api

# Common causes:
# - File too large (check maxFileSize settings)
# - Invalid EVTX format
# - Disk space full
```

**Solutions**:
```bash
# Check disk space
df -h

# Check upload directory permissions (local)
ls -la uploads/

# Increase file size limit in appsettings.json
```

#### 5. Jobs Stuck in "Queued" Status

**Problem**: Jobs never move to "processing"

**Solutions**:
```bash
# Check background worker is running
docker-compose logs log-transformer-api | grep "IngestWorker"

# Verify database connection
docker-compose logs log-transformer-api | grep "Connected to database"

# Restart service
docker-compose restart log-transformer-api
```

#### 6. Permission Errors (Linux)

**Problem**: Permission denied when accessing uploads directory

**Solutions**:
```bash
# Fix upload directory permissions
chmod 755 uploads/
chown -R $USER:$USER uploads/

# Or for Docker
docker-compose down
sudo chown -R 1000:1000 uploads/
docker-compose up -d
```

#### 7. Port Already in Use

**Problem**: `Address already in use: 5001`

**Solutions**:
```bash
# Find process using port 5001
# Windows
netstat -ano | findstr :5001

# Linux/macOS
lsof -i :5001

# Kill the process or change port in docker-compose.yml
ports:
  - "5002:8080"  # Use different external port
```

### Debugging Tips

#### Enable Debug Logging

Edit `appsettings.json`:
```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Debug",
      "Microsoft.EntityFrameworkCore": "Information"
    }
  }
}
```

#### View Detailed Logs

```bash
# All logs
docker-compose logs -f

# Last 100 lines
docker-compose logs --tail=100 log-transformer-api

# Follow specific pattern
docker-compose logs -f | grep ERROR
```

#### Database Query Logging

Enable in `appsettings.Development.json`:
```json
{
  "Logging": {
    "LogLevel": {
      "Microsoft.EntityFrameworkCore.Database.Command": "Information"
    }
  }
}
```

#### Test Database Connection

```bash
# From host machine
psql -h localhost -p 5432 -U postgres -d Security

# From Docker
docker exec -it ai-security-db psql -U postgres -d Security
```

---

## Advanced Topics

### Custom Batch Processing

Configure batch size and polling interval:

```json
{
  "Import": {
    "BatchSize": 2000,
    "PollIntervalSeconds": 10
  }
}
```

### File Storage Management

Configure storage location:

```json
{
  "Storage": {
    "UploadPath": "/custom/path/uploads"
  }
}
```

Ensure the path exists and has proper permissions.

### Production Deployment

For production:

1. **Use proper connection strings**:
   - Store credentials securely (Azure Key Vault, AWS Secrets Manager)
   - Use connection pooling

2. **Configure logging**:
   - Use structured logging (Serilog)
   - Send logs to centralized system

3. **Set resource limits**:
   ```yaml
   deploy:
     resources:
       limits:
         cpus: '2'
         memory: 2G
   ```

4. **Enable health checks**:
   ```csharp
   builder.Services.AddHealthChecks()
       .AddNpgSql(connectionString);
   ```

5. **Use reverse proxy**:
   - nginx or Traefik for SSL termination
   - Rate limiting
   - Load balancing

### Development Tools

```bash
# Run tests (if available)
dotnet test

# Format code
dotnet format

# Create migration (if using EF migrations)
dotnet ef migrations add MigrationName

# Update database schema
dotnet ef database update

# Build for release
dotnet build -c Release

# Publish application
dotnet publish -c Release -o ./publish
```

---

## API Reference Quick Guide

### Endpoints

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/upload/evtx` | Upload EVTX file |
| GET | `/jobs/{id}/status` | Get job status |
| GET | `/swagger` | API documentation |

### Status Codes

| Code | Description |
|------|-------------|
| 200 | Success |
| 201 | Created |
| 400 | Bad Request (invalid file/format) |
| 404 | Not Found (job doesn't exist) |
| 500 | Internal Server Error |

---

## Performance Tuning

### Database Performance

```sql
-- Add index for faster job lookups
CREATE INDEX idx_ingest_jobs_status ON ingest_jobs(status);

-- Add index for log queries
CREATE INDEX idx_normalized_logs_source ON normalized_logs(source);
```

### Memory Configuration

For large EVTX files, increase memory:

```yaml
environment:
  - DOTNET_GCHeapCount=4
  - DOTNET_GCServer=1
```

### Parallel Processing

Increase worker threads in configuration:

```json
{
  "Import": {
    "MaxConcurrentJobs": 3
  }
}
```

---

## Next Steps

- Review [API Reference](./LT-2-api-reference.md)
- Explore [Architecture](./LT-3-architecture.md)
- Configure [Integration](./LT-README.md)
- Check [AI-Security Documentation](../../ai-security/documentation/)

---

**Last Updated**: November 2024  
**Version**: 1.0.0
