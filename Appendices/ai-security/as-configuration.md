# Appendix F-1.4 — Configuration

# Configuration Guide

Complete reference for configuring the AI Security Backend.

## Table of Contents

- [Configuration Methods](#configuration-methods)
- [Environment Variables](#environment-variables)
- [Config YAML File](#config-yaml-file)
- [Configuration Precedence](#configuration-precedence)
- [Detection Modes](#detection-modes)
- [Log Grouping](#log-grouping)
- [Codex SDK Configuration](#codex-sdk-configuration)
- [Model Providers](#model-providers)
- [Database Configuration](#database-configuration)
- [API Configuration](#api-configuration)
- [Logging Configuration](#logging-configuration)
- [Agent Configuration](#agent-configuration)
- [Runtime Configuration Updates](#runtime-configuration-updates)
- [Environment-Specific Configs](#environment-specific-configs)

## Configuration Methods

The backend supports two configuration methods:

1. **Environment Variables** (`.env` file)
   - Base configuration
   - Always loaded
   - Simple key-value pairs

2. **YAML Configuration** (`config.yaml`)
   - Advanced configuration
   - Takes precedence for detection and AI settings
   - Optional, falls back to environment variables

## Environment Variables

### Required Variables

```env
DATABASE_URL=postgresql://username:password@localhost:5432/security_db
```

### All Environment Variables

#### Database

```env
# Database
DATABASE_URL=postgresql://securityai:securepassword@localhost:5432/ai_security
DATABASE_POOL_MAX=10
DATABASE_POOL_MIN=2
DATABASE_POOL_ACQUIRE=30000
DATABASE_POOL_IDLE=10000
```

| Variable | Description | Default | Type |
|----------|-------------|---------|------|
| `DATABASE_URL` | PostgreSQL connection string | - | String (Required) |
| `DATABASE_POOL_SIZE` | Maximum database connection pool size | `10` | Number |

#### Detection Settings

```env
DETECTION_MODE=polling
POLLING_INTERVAL_MS=600000
BATCH_SIZE=100
```

| Variable | Description | Default | Type |
|----------|-------------|---------|------|
| `DETECTION_MODE` | Log ingestion mode: `realtime`, `polling`, or `ai-only` | `polling` | Enum |
| `POLLING_INTERVAL_MS` | Polling interval in milliseconds (polling mode only) | `600000` (10 min) | Number |
| `BATCH_SIZE` | Number of logs to process per batch | `100` | Number |

#### Log Grouping

```env
GROUPING_ENABLED=true
GROUPING_TIME_WINDOW_MINUTES=5
GROUPING_MATCH_FIELDS=normalized_data.source_ip,normalized_data.host,normalized_data.user
GROUPING_INCLUDE_PROCESSED=true
```

| Variable | Description | Default | Type |
|----------|-------------|---------|------|
| `GROUPING_ENABLED` | Enable log grouping for correlated analysis | `true` | Boolean |
| `GROUPING_TIME_WINDOW_MINUTES` | Time window for grouping logs (minutes) | `5` | Number |
| `GROUPING_MATCH_FIELDS` | Comma-separated fields to match for grouping | See below | String |
| `GROUPING_INCLUDE_PROCESSED` | Include previously processed logs in groups | `true` | Boolean |

**Default Match Fields**:
- `normalized_data.source_ip`
- `normalized_data.host`
- `normalized_data.user`

#### API Server

```env
API_PORT=3000
API_HOST=0.0.0.0
CORS_ORIGIN=http://localhost:5173,http://localhost:3000
CORS_CREDENTIALS=true
```

| Variable | Description | Default | Type |
|----------|-------------|---------|------|
| `API_PORT` | API server port | `3000` | Number |
| `API_HOST` | API server host | `0.0.0.0` | String |
| `CORS_ORIGIN` | Allowed CORS origins (comma-separated) | `*` | String |
| `CORS_CREDENTIALS` | Enable CORS credentials | `false` | Boolean |

#### Logging

```env
LOG_LEVEL=info
LOG_DIR=./logs
```

| Variable | Description | Default | Type |
|----------|-------------|---------|------|
| `LOG_LEVEL` | Logging level: `error`, `warn`, `info`, `debug` | `info` | Enum |
| `LOG_DIR` | Directory for log files (empty = console only) | Empty | String |

#### AI Agents

```env
AGENT_WORKSPACE_PATH=./data/agent-workspace
AGENT_WORKSPACE_TEMP_PATH=./data/agent-workspace/temp
MCP_LOG_LEVEL=info
```

| Variable | Description | Default | Type |
|----------|-------------|---------|------|
| `AGENT_WORKSPACE_PATH` | Base directory for agent workspaces | `./data/agent-workspace` | String |
| `AGENT_WORKSPACE_TEMP_PATH` | Temporary directory for agent operations | `./data/agent-workspace/temp` | String |
| `MCP_LOG_LEVEL` | MCP server log level | `info` | Enum |

#### Codex SDK

```env
CODEX_MODEL=gpt-5-mini
CODEX_MODEL_PROVIDER=openai
CODEX_API_KEY=sk-your-key-here
CODEX_BASE_URL=https://api.openai.com/v1

OPENAI_API_KEY=sk-your-key-here
OPENAI_BASE_URL=https://api.openai.com/v1
```

| Variable | Description | Default | Type |
|----------|-------------|---------|------|
| `CODEX_MODEL` | Model identifier (e.g., `gpt-5-high`, `gpt-5-nano`) | - | String |
| `CODEX_MODEL_PROVIDER` | Model provider name (e.g., `mistral`, `openai`, `ollama`) | - | String |
| `CODEX_API_KEY` | API key for Codex SDK | Falls back to `OPENAI_API_KEY` | String |
| `CODEX_BASE_URL` | Base URL for Codex API | Falls back to `OPENAI_BASE_URL` | String |
| `OPENAI_API_KEY` | OpenAI API key (fallback for `CODEX_API_KEY`) | - | String (Required) |
| `OPENAI_BASE_URL` | OpenAI base URL (fallback for `CODEX_BASE_URL`) | - | String |

#### Feedback System

```env
FEEDBACK_UPDATE_THRESHOLD=1
```

| Variable | Description | Default | Type |
|----------|-------------|---------|------|
| `FEEDBACK_UPDATE_THRESHOLD` | Minimum feedback count to update patterns | `1` | Number |

#### Environment

```env
NODE_ENV=development
```

| Variable | Description | Default | Type |
|----------|-------------|---------|------|
| `NODE_ENV` | Environment: `development`, `production`, `test` | `development` | Enum |

### Model Provider Configuration (Advanced)

Configure custom model providers using environment variables:

```env
CODEX_MODEL_PROVIDER_OLLAMA_NAME=ollama
CODEX_MODEL_PROVIDER_OLLAMA_BASE_URL=http://localhost:11434/v1
CODEX_MODEL_PROVIDER_OLLAMA_ENV_KEY=OLLAMA_API_KEY
CODEX_MODEL_PROVIDER_OLLAMA_WIRE_API=chat

CODEX_MODEL_PROVIDER_AZURE_NAME=azure
CODEX_MODEL_PROVIDER_AZURE_BASE_URL=https://your-resource.openai.azure.com
CODEX_MODEL_PROVIDER_AZURE_ENV_KEY=AZURE_OPENAI_API_KEY
CODEX_MODEL_PROVIDER_AZURE_WIRE_API=chat
```

Format: `CODEX_MODEL_PROVIDER_<NAME>_<PROPERTY>=value`

Properties:
- `NAME` - Provider name (e.g., `OLLAMA`, `AZURE`)
- `BASE_URL` - API endpoint
- `ENV_KEY` - Environment variable name for API key
- `WIRE_API` - API protocol: `chat` or `responses`
- `QUERY_PARAMS` - JSON object of query parameters
- `HTTP_HEADERS` - JSON object of HTTP headers
- `ENV_HTTP_HEADERS` - JSON object mapping header names to env variable names

## Config YAML File

Create `config.yaml` in the root directory for advanced configuration:

```yaml
detection:
  mode: polling
  pollingIntervalMs: 600000
  batchSize: 100
  grouping:
    enabled: true
    timeWindowMinutes: 5
    matchFields:
      - normalized_data.source_ip
      - normalized_data.host
      - normalized_data.user
    includeProcessedLogs: true

codex:
  model: gpt-oss
  modelProvider: ollama
  modelProviders:
    ollama:
      name: ollama
      baseUrl: http://localhost:11434/v1
      envKey: OLLAMA_API_KEY
      wireApi: chat

prompts:
  detection:
    systemPrompt: "Custom system prompt for detection agent..."
    userPromptTemplate: "Custom user prompt template..."

  remediation:
    systemPrompt: "Custom system prompt for advisor agent..."
    userPromptTemplate: "Custom user prompt template..."

  quality:
    systemPrompt: "Custom system prompt for quality agent..."
    userPromptTemplate: "Custom user prompt template..."
```

### YAML Schema Reference

#### Detection Configuration

```yaml
detection:
  mode: polling                    # realtime | polling | ai-only
  pollingIntervalMs: 600000        # Number (milliseconds)
  batchSize: 100                   # Number
  grouping:
    enabled: true                  # Boolean
    timeWindowMinutes: 5           # Number
    matchFields:                   # String[]
      - field1
      - field2
    includeProcessedLogs: true     # Boolean
```

#### Codex Configuration

```yaml
codex:
  model: string                    # Model identifier
  modelProvider: string            # Provider name
  apiKey: string                   # API key (supports ${ENV_VAR} syntax)
  baseUrl: string                  # API base URL
  modelProviders:                  # Record<string, ModelProviderConfig>
    providerName:
      name: string                 # Provider name
      baseUrl: string              # Required
      envKey: string               # Optional
      wireApi: chat | responses    # Optional
      queryParams:                 # Optional
        key: value
      httpHeaders:                 # Optional
        key: value
      envHttpHeaders:              # Optional
        headerName: ENV_VAR_NAME
```

#### Prompts Configuration

```yaml
prompts:
  detection:
    systemPrompt: string
    userPromptTemplate: string
  remediation:
    systemPrompt: string
    userPromptTemplate: string
  quality:
    systemPrompt: string
    userPromptTemplate: string
```

## Configuration Precedence

Configuration is loaded in the following order (later overrides earlier):

1. **Default Values** (hardcoded in `src/utils/config.ts`)
2. **Environment Variables** (`.env` file)
3. **YAML File** (`config.yaml`) - only for `detection`, `codex`, and `prompts`

Example:

```
DATABASE_URL → Always from .env
API_PORT → Always from .env
DETECTION_MODE → config.yaml if exists, otherwise .env
CODEX_MODEL → config.yaml if exists, otherwise .env
```

## Detection Modes

### Polling Mode (Default)

Periodically queries database for unprocessed logs.

```yaml
detection:
  mode: polling
  pollingIntervalMs: 600000  # 10 minutes
  batchSize: 100
```

**Pros**:
- Reliable and predictable
- No persistent database connections
- Easy to debug

**Cons**:
- Higher latency (up to polling interval)
- Regular database queries

**Use When**:
- Reliability is critical
- Real-time processing not required
- Simpler deployment

### Realtime Mode

Uses PostgreSQL LISTEN/NOTIFY for instant log processing.

```yaml
detection:
  mode: realtime
```

**Pros**:
- Immediate processing
- Lower latency
- Event-driven

**Cons**:
- Requires persistent database connection
- More complex error handling
- Potential connection drops

**Use When**:
- Low latency is critical
- High-frequency log ingestion
- Real-time alerting required

**Requirements**:
- PostgreSQL trigger for NOTIFY
- Stable database connection

### AI-Only Mode

Processes logs using only AI agents, bypassing rule engine.

```yaml
detection:
  mode: ai-only
  pollingIntervalMs: 600000
  batchSize: 100
```

**Pros**:
- No rule maintenance
- Flexible threat detection
- Adapts to new threats

**Cons**:
- Higher AI API costs
- Slower processing
- Less predictable

**Use When**:
- Exploring new threat patterns
- Rule development/testing
- Advanced threat hunting

## Log Grouping

Groups related logs for correlated analysis (attack chains, brute force, etc.).

### Configuration

```yaml
detection:
  grouping:
    enabled: true
    timeWindowMinutes: 5
    matchFields:
      - normalized_data.source_ip
      - normalized_data.host
      - normalized_data.user
    includeProcessedLogs: true
```

### How It Works

1. Logs are grouped based on matching values in specified fields
2. Groups are formed within the time window
3. Optionally includes previously processed logs
4. Entire group is analyzed together for context

### Example

With `matchFields: [normalized_data.source_ip]`:

```
Log 1: source_ip=192.168.1.100, timestamp=10:00:00
Log 2: source_ip=192.168.1.100, timestamp=10:02:00
Log 3: source_ip=192.168.1.101, timestamp=10:01:00
```

Creates 2 groups:
- Group A: [Log 1, Log 2] (same source IP)
- Group B: [Log 3] (different source IP)

### Best Practices

- Use fields that correlate attacks: `source_ip`, `user`, `host`
- Adjust `timeWindowMinutes` based on attack patterns
  - Brute force: 5-10 minutes
  - Lateral movement: 30-60 minutes
- Set `includeProcessedLogs: true` for attack chain detection

## Codex SDK Configuration

### Using OpenAI

```yaml
codex:
  model: gpt-4
  modelProvider: openai
  apiKey: ${OPENAI_API_KEY}
```

```env
OPENAI_API_KEY=sk-your-key-here
```

### Using Ollama (Local)

```yaml
codex:
  model: llama3:latest
  modelProvider: ollama
  apiKey: ollama
  modelProviders:
    ollama:
      name: ollama
      baseUrl: http://localhost:11434/v1
      envKey: OLLAMA_API_KEY
      wireApi: chat
```

```env
OLLAMA_API_KEY=ollama
```

### Using Azure OpenAI

```yaml
codex:
  model: gpt-4
  modelProvider: azure
  apiKey: ${AZURE_OPENAI_API_KEY}
  modelProviders:
    azure:
      name: azure
      baseUrl: https://your-resource.openai.azure.com
      envKey: AZURE_OPENAI_API_KEY
      wireApi: chat
      queryParams:
        api-version: "2024-02-15-preview"
```

```env
AZURE_OPENAI_API_KEY=your-azure-key
```

### Multiple Model Providers

Configure multiple providers and switch via `modelProvider`:

```yaml
codex:
  model: gpt-ss
  modelProvider: ollama
  apiKey: ${OPENAI_API_KEY}
  modelProviders:
    ollama:
      name: ollama
      baseUrl: http://localhost:11434/v1
      envKey: OLLAMA_API_KEY
    openai:
      name: openai
      baseUrl: https://api.openai.com/v1
      envKey: OPENAI_API_KEY
```

Switch provider at runtime via API (see [Runtime Configuration](#runtime-configuration-updates)).

## Model Providers

### Provider Configuration

```yaml
codex:
  modelProviders:
    providerName:
      name: string              # Provider identifier
      baseUrl: string           # API endpoint (required)
      envKey: string            # Environment variable for API key
      wireApi: chat | responses # API protocol
      queryParams:              # URL query parameters
        key: value
      httpHeaders:              # HTTP headers
        Authorization: "Bearer token"
      envHttpHeaders:           # Headers from env vars
        X-API-Key: API_KEY_ENV_VAR
```

## Database Configuration

```env
DATABASE_URL=postgresql://username:password@host:port/database?sslmode=require
DATABASE_POOL_SIZE=10
```

### Connection String Format

```
postgresql://[user[:password]@][host][:port][/dbname][?param1=value1&...]
```

### SSL/TLS

For production, enable SSL:

```env
DATABASE_URL=postgresql://user:pass@host:5432/db?sslmode=require
```

SSL Modes:
- `disable` - No SSL
- `prefer` - SSL if available (default)
- `require` - Require SSL
- `verify-ca` - Verify CA certificate
- `verify-full` - Verify CA and hostname

### Connection Pool

```env
DATABASE_POOL_SIZE=20
```

Adjust based on:
- Number of concurrent requests
- Database connection limit
- System resources

## API Configuration

```env
API_PORT=3000
API_HOST=0.0.0.0
CORS_ORIGIN=http://localhost:5173,https://app.example.com
CORS_CREDENTIALS=true
```

### CORS Configuration

**Development (allow all)**:
```env
CORS_ORIGIN=*
CORS_CREDENTIALS=false
```

**Production (specific origins)**:
```env
CORS_ORIGIN=https://app.example.com,https://dashboard.example.com
CORS_CREDENTIALS=true
```

## Logging Configuration

```env
LOG_LEVEL=info
LOG_DIR=./logs
```

### Log Levels

- `error` - Only errors
- `warn` - Warnings and errors
- `info` - Info, warnings, and errors (default)
- `debug` - All logs including debug

### File Logging

Leave `LOG_DIR` empty for console-only logging:

```env
LOG_DIR=
```

Set `LOG_DIR` for file logging:

```env
LOG_DIR=./logs
```

Creates:
- `logs/combined.log` - All logs
- `logs/error.log` - Error logs only

## Agent Configuration

```env
AGENT_WORKSPACE_PATH=./data/agent-workspace
AGENT_WORKSPACE_TEMP_PATH=./data/agent-workspace/temp
MCP_LOG_LEVEL=info
```

### Workspace Structure

```
data/agent-workspace/
├── detection/     # DetectionAgent workspace
├── advisor/       # AdvisorAgent workspace
├── quality/       # QualityAgent workspace
└── temp/          # Temporary files
```

Each agent gets isolated workspace for Codex SDK operations.

## Runtime Configuration Updates

Update configuration at runtime via API:

```bash
curl -X PATCH http://localhost:3000/api/config \
  -H "Content-Type: application/json" \
  -d '{
    "detection": {
      "mode": "polling",
      "pollingIntervalMs": 300000
    },
    "codex": {
      "model": "gpt-4",
      "modelProvider": "openai"
    }
  }'
```

Changes are:
1. Applied immediately
2. Saved to `config.yaml`
3. Persisted across restarts
4. LogListener is restarted automatically

See [API Reference](./api-reference.md#configuration-endpoints) for details.

## Environment-Specific Configs

### Development

```env
NODE_ENV=development
LOG_LEVEL=debug
CORS_ORIGIN=*
DETECTION_MODE=polling
POLLING_INTERVAL_MS=60000
```

### Production

```env
NODE_ENV=production
LOG_LEVEL=info
LOG_DIR=./logs
CORS_ORIGIN=https://app.example.com
DETECTION_MODE=realtime
DATABASE_URL=postgresql://user:pass@prod-db:5432/security?sslmode=require
```

### Testing

```env
NODE_ENV=test
LOG_LEVEL=error
DATABASE_URL=postgresql://user:pass@localhost:5432/security_test
```

## Configuration Validation

The application validates configuration on startup:

- Required variables present
- Valid enum values (mode, log level)
- Valid URLs and ports
- Database connection successful
- Model provider compatibility

Check startup logs for validation errors.

## Best Practices

1. **Use `.env` for secrets** - Never commit sensitive values
2. **Use `config.yaml` for settings** - Version control safe configuration
3. **Environment variables for deployment** - Override for different environments
4. **Document custom configs** - Comment your `config.yaml`
5. **Validate before deploy** - Test configuration changes locally
6. **Monitor after changes** - Watch logs after runtime updates

---

For more help, see:
- [Getting Started](./getting-started.md)
- [API Reference](./api-reference.md#configuration-endpoints)
