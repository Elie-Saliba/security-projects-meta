# Appendix F-2.5 — Configuration

## Overview

Log-Transformer uses a hierarchical configuration system based on ASP.NET Core's configuration framework. Configuration can be provided through JSON files, environment variables, or command-line arguments, with later sources overriding earlier ones.

## Configuration Hierarchy

Configuration is loaded in the following order (later sources override earlier):

1. `appsettings.json` - Base configuration
2. `appsettings.{Environment}.json` - Environment-specific overrides
3. Environment variables - Container/deployment overrides
4. Command-line arguments - Runtime overrides

## Configuration Files

### appsettings.json (Base Configuration)

Default configuration for all environments:

```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning",
      "Microsoft.EntityFrameworkCore": "Warning"
    }
  },
  "AllowedHosts": "*",
  "Postgres": {
    "ConnectionString": ""
  },
  "Storage": {
    "UploadPath": "uploads"
  },
  "Import": {
    "BatchSize": 1000,
    "PollIntervalSeconds": 5
  }
}
```

### appsettings.Development.json

Local development overrides:

```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Debug",
      "Microsoft.AspNetCore": "Information",
      "Microsoft.EntityFrameworkCore": "Information"
    }
  },
  "Postgres": {
    "ConnectionString": "Host=localhost;Port=5432;Database=Security;Username=postgres;Password=TestDb123;"
  },
  "Storage": {
    "UploadPath": "uploads"
  },
  "Import": {
    "BatchSize": 500,
    "PollIntervalSeconds": 3
  }
}
```

### appsettings.Production.json

Production environment overrides:

```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Warning",
      "Microsoft.AspNetCore": "Warning",
      "Microsoft.EntityFrameworkCore": "Error"
    }
  },
  "Storage": {
    "UploadPath": "/app/uploads"
  },
  "Import": {
    "BatchSize": 2000,
    "PollIntervalSeconds": 5
  }
}
```

## Configuration Sections

### 1. Logging Configuration

Controls application logging behavior.

**Section**: `Logging`

**Properties**:

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| LogLevel.Default | string | Information | Default logging level |
| LogLevel.Microsoft.AspNetCore | string | Warning | ASP.NET Core logs |
| LogLevel.Microsoft.EntityFrameworkCore | string | Warning | Entity Framework logs |

**Log Levels**:
- `Trace` - Most detailed
- `Debug` - Debugging information
- `Information` - General information
- `Warning` - Warning messages
- `Error` - Error messages
- `Critical` - Critical failures
- `None` - Disable logging

**Example**:
```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning",
      "Microsoft.EntityFrameworkCore": "Information",
      "Api.Features.Ingest": "Debug"
    }
  }
}
```

**Environment Variable Override**:
```bash
export Logging__LogLevel__Default=Debug
```

### 2. PostgreSQL Configuration

Database connection settings.

**Section**: `Postgres`

**Properties**:

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| ConnectionString | string | Yes | PostgreSQL connection string |

**Connection String Format**:
```
Host={hostname};Port={port};Database={database};Username={user};Password={password};
```

**Example**:
```json
{
  "Postgres": {
    "ConnectionString": "Host=localhost;Port=5432;Database=Security;Username=postgres;Password=secure_password;"
  }
}
```

**Environment Variable Override**:
```bash
# Docker Compose
environment:
  - Postgres__ConnectionString=Host=ai-security-db;Port=5432;Database=Security;Username=postgres;Password=TestDb123

# Shell Export
export Postgres__ConnectionString="Host=localhost;Port=5432;Database=Security;Username=postgres;Password=TestDb123;"
```

**Advanced Connection Options**:
```
Host=localhost;
Port=5432;
Database=Security;
Username=postgres;
Password=secure_password;
Pooling=true;
Minimum Pool Size=5;
Maximum Pool Size=20;
Connection Lifetime=300;
Timeout=30;
Command Timeout=60;
SSL Mode=Require;
Trust Server Certificate=false;
```

### 3. Storage Configuration

File upload and storage settings.

**Section**: `Storage`

**Properties**:

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| UploadPath | string | uploads | Base directory for uploaded files |

**Path Formats**:
- **Relative Path**: `uploads` (relative to application root)
- **Absolute Path**: `/app/uploads` (absolute path)
- **Windows Path**: `C:\Logs\uploads`

**Example**:
```json
{
  "Storage": {
    "UploadPath": "uploads"
  }
}
```

**File Organization**:
```
{UploadPath}/
  ├── 20251107/
  │   ├── 01HXXX...XXX.evtx
  │   ├── 01HYYY...YYY.jsonl
  │   └── ...
  ├── 20251108/
  │   └── ...
  └── ...
```

**Environment Variable Override**:
```bash
export Storage__UploadPath=/app/uploads
```

**Docker Volume Mapping**:
```yaml
services:
  log-transformer-api:
    volumes:
      - ./uploads:/app/uploads
    environment:
      - Storage__UploadPath=/app/uploads
```

### 4. Import Configuration

Log processing and ingestion settings.

**Section**: `Import`

**Properties**:

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| BatchSize | int | 1000 | Number of records per database batch |
| PollIntervalSeconds | int | 5 | Seconds between job queue polls |

**Batch Size Guidelines**:
- **Small (100-500)**: Lower memory usage, higher latency
- **Medium (1000-2000)**: Balanced performance
- **Large (5000+)**: Higher throughput, more memory

**Poll Interval Guidelines**:
- **Fast (1-3 sec)**: Lower latency, higher CPU usage
- **Medium (5-10 sec)**: Balanced responsiveness
- **Slow (30+ sec)**: Background processing, lower overhead

**Example**:
```json
{
  "Import": {
    "BatchSize": 1000,
    "PollIntervalSeconds": 5
  }
}
```

**Environment Variable Override**:
```bash
export Import__BatchSize=2000
export Import__PollIntervalSeconds=3
```

**Performance Tuning**:

*High Throughput Configuration*:
```json
{
  "Import": {
    "BatchSize": 5000,
    "PollIntervalSeconds": 1
  }
}
```

*Low Resource Configuration*:
```json
{
  "Import": {
    "BatchSize": 500,
    "PollIntervalSeconds": 10
  }
}
```

### 5. ASP.NET Core Configuration

**Properties**:

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| AllowedHosts | string | * | Allowed host headers (security) |
| ASPNETCORE_URLS | string | http://localhost:5132 | Listening URLs |
| ASPNETCORE_ENVIRONMENT | string | Production | Runtime environment |

**Example**:
```json
{
  "AllowedHosts": "*.example.com;localhost"
}
```

**Environment Variables**:
```bash
export ASPNETCORE_URLS=http://+:8080
export ASPNETCORE_ENVIRONMENT=Development
```

## Environment Variables

### Naming Convention

Environment variables use `__` (double underscore) as section delimiters:

```
Section__SubSection__Property
```

**Examples**:
```bash
Postgres__ConnectionString
Storage__UploadPath
Import__BatchSize
Logging__LogLevel__Default
```

### Docker Compose Configuration

**docker-compose.yml**:
```yaml
version: "3.9"

services:
  log-transformer-api:
    image: log-transformer:latest
    environment:
      # Database Configuration
      - Postgres__ConnectionString=Host=ai-security-db;Port=5432;Database=Security;Username=postgres;Password=${DB_PASSWORD}
      
      # Storage Configuration
      - Storage__UploadPath=/app/uploads
      
      # Import Configuration
      - Import__BatchSize=2000
      - Import__PollIntervalSeconds=5
      
      # ASP.NET Core Configuration
      - ASPNETCORE_URLS=http://+:8080
      - ASPNETCORE_ENVIRONMENT=Production
      
      # Logging Configuration
      - Logging__LogLevel__Default=Information
    
    volumes:
      - ./uploads:/app/uploads
    
    ports:
      - "5001:8080"
    
    networks:
      - ai-security-network
```

### Using .env File

Create a `.env` file for sensitive configuration:

**.env**:
```bash
DB_HOST=ai-security-db
DB_PORT=5432
DB_NAME=Security
DB_USER=postgres
DB_PASSWORD=secure_password_here
UPLOAD_PATH=/app/uploads
BATCH_SIZE=1000
```

**docker-compose.yml with .env**:
```yaml
services:
  log-transformer-api:
    environment:
      - Postgres__ConnectionString=Host=${DB_HOST};Port=${DB_PORT};Database=${DB_NAME};Username=${DB_USER};Password=${DB_PASSWORD}
      - Storage__UploadPath=${UPLOAD_PATH}
      - Import__BatchSize=${BATCH_SIZE}
```

## Command-Line Arguments

Override configuration at runtime:

```bash
dotnet run --Postgres:ConnectionString="Host=localhost..." --Import:BatchSize=500
```

## Configuration Validation

### Startup Validation

The application validates configuration on startup:

```csharp
builder.Services.AddOptions<PgOptions>()
    .Bind(configuration.GetSection("Postgres"))
    .ValidateDataAnnotations()
    .ValidateOnStart();
```

**Validation Rules**:
- `Postgres:ConnectionString` must not be empty
- `Import:BatchSize` must be > 0
- `Import:PollIntervalSeconds` must be > 0

### Fallback Behavior

If database configuration is missing:
- Application logs a warning
- Falls back to in-memory database (development only)
- Not suitable for production

## Security Best Practices

### 1. Sensitive Data Protection

**Don't**:
```json
{
  "Postgres": {
    "ConnectionString": "Host=db;Username=postgres;Password=MyPassword123;"
  }
}
```

**Do**:
```bash
# Use environment variables
export Postgres__ConnectionString="Host=db;Username=postgres;Password=${DB_PASSWORD}"
```

### 2. Use Secrets Management

**Docker Secrets**:
```yaml
services:
  log-transformer-api:
    secrets:
      - db_password
    environment:
      - Postgres__ConnectionString=Host=db;Username=postgres;Password_File=/run/secrets/db_password

secrets:
  db_password:
    file: ./secrets/db_password.txt
```

**Azure Key Vault**:
```csharp
builder.Configuration.AddAzureKeyVault(
    new Uri("https://myvault.vault.azure.net/"),
    new DefaultAzureCredential()
);
```

**AWS Secrets Manager**:
```csharp
builder.Configuration.AddSecretsManager(
    configurator: options => {
        options.SecretFilter = entry => entry.Name.StartsWith("LogTransformer");
    }
);
```

### 3. Restrict Allowed Hosts

```json
{
  "AllowedHosts": "log-transformer.example.com;*.internal.example.com"
}
```

## Configuration by Environment

### Development

**Focus**: Debugging and rapid iteration

```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Debug"
    }
  },
  "Postgres": {
    "ConnectionString": "Host=localhost;Database=Security;..."
  },
  "Import": {
    "BatchSize": 500,
    "PollIntervalSeconds": 3
  }
}
```

### Staging

**Focus**: Pre-production testing

```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Information"
    }
  },
  "Import": {
    "BatchSize": 1000,
    "PollIntervalSeconds": 5
  }
}
```

### Production

**Focus**: Performance and stability

```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Warning"
    }
  },
  "Import": {
    "BatchSize": 2000,
    "PollIntervalSeconds": 5
  }
}
```

## Advanced Configuration

### Multiple Database Connections

```json
{
  "Postgres": {
    "ConnectionString": "Host=primary;...",
    "ReadReplicaConnectionString": "Host=replica;..."
  }
}
```

### Source-Specific Configuration

```json
{
  "Import": {
    "BatchSize": 1000,
    "PollIntervalSeconds": 5,
    "Sources": {
      "evtx": {
        "BatchSize": 2000,
        "Enabled": true
      },
      "jsonl": {
        "BatchSize": 1000,
        "Enabled": true
      },
      "syslog": {
        "BatchSize": 500,
        "Enabled": true
      }
    }
  }
}
```

### Health Check Configuration

```json
{
  "HealthChecks": {
    "Database": {
      "Enabled": true,
      "Timeout": 5
    },
    "Storage": {
      "Enabled": true,
      "MinFreeDiskSpace": 1073741824
    }
  }
}
```

### Telemetry Configuration

```json
{
  "OpenTelemetry": {
    "ServiceName": "log-transformer",
    "Endpoint": "http://otel-collector:4317",
    "Enabled": true
  }
}
```

## Troubleshooting Configuration Issues

### View Active Configuration

```csharp
// In Program.cs or a controller
app.MapGet("/debug/config", (IConfiguration config) => 
{
    return new {
        PostgresConnection = config["Postgres:ConnectionString"]?.Substring(0, 20) + "...",
        StoragePath = config["Storage:UploadPath"],
        BatchSize = config["Import:BatchSize"],
        Environment = Environment.GetEnvironmentVariable("ASPNETCORE_ENVIRONMENT")
    };
});
```

### Common Issues

**Issue**: Configuration not loading

**Solution**: Check file naming, environment variable format

**Issue**: Database connection fails

**Solution**: Verify connection string format, network connectivity

**Issue**: Files not uploading

**Solution**: Check storage path exists and is writable

## Configuration Templates

### Minimal Configuration

```json
{
  "Postgres": {
    "ConnectionString": "Host=localhost;Port=5432;Database=Security;Username=postgres;Password=password;"
  }
}
```

### Recommended Production Configuration

```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Warning",
      "Microsoft.AspNetCore": "Warning",
      "Microsoft.EntityFrameworkCore": "Error"
    }
  },
  "AllowedHosts": "log-transformer.example.com",
  "Postgres": {
    "ConnectionString": "Host=db;Port=5432;Database=Security;Username=app_user;Password=${DB_PASSWORD};Pooling=true;Maximum Pool Size=20;"
  },
  "Storage": {
    "UploadPath": "/app/uploads"
  },
  "Import": {
    "BatchSize": 2000,
    "PollIntervalSeconds": 5
  }
}
```
