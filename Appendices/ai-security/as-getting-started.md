# Appendix F-1.8 — Getting Started

This guide will help you set up and run the AI Security Backend for the first time.

## Table of Contents

- [Prerequisites](#prerequisites)
- [Installation](#installation)
- [Configuration](#configuration)
- [Database Setup](#database-setup)
- [Running the Application](#running-the-application)
- [Verification](#verification)
- [Common Commands](#common-commands)
- [Development vs Production](#development-vs-production)
- [Troubleshooting](#troubleshooting)

## Prerequisites

### Required

- **Node.js** 18 or higher
  ```bash
  node --version
  ```

- **PostgreSQL** 12 or higher
  ```bash
  psql --version
  ```

- **npm** or **yarn**
  ```bash
  npm --version
  ```

## Installation

### 1. Clone the Repository

```bash
git clone <repository-url>
cd ai-security
```

### Backend

Navigate to backend root and install all packages:

```bash
cd backend
npm install
```

This will install dependencies for both `core` and `codex-sdk` packages.

Build packages:

```bash
npm run build
```

## Configuration

### 1. Create Environment File

```bash
cd ai-security-backend/packages/core
touch .env
```

### 2. Edit `.env` File

Open `.env` and configure the following critical variables:

```env
DATABASE_URL=postgresql://username:password@localhost:5432/security_db
NODE_ENV=development
LOG_LEVEL=debug

DETECTION_MODE=polling
POLLING_INTERVAL_MS=600000
BATCH_SIZE=100

API_PORT=3000
API_HOST=0.0.0.0
CORS_ORIGIN=http://localhost:5173

OPENAI_API_KEY=sk-your-openai-api-key-here

AGENT_WORKSPACE_PATH=./data/agent-workspace
```

**Key Variables:**

| Variable | Description | Default | Required |
|----------|-------------|---------|----------|
| `DATABASE_URL` | PostgreSQL connection string | - | ✅ Yes |
| `DETECTION_MODE` | Log ingestion mode: `realtime`, `polling`, or `ai-only` | `polling` | No |
| `POLLING_INTERVAL_MS` | Polling interval in milliseconds | `600000` (10 min) | No |
| `API_PORT` | API server port | `3000` | No |
| `API_HOST` | API server host | `0.0.0.0` | No |

See [Configuration Guide](./configuration.md) for complete reference.

### 3. Optional: Create `config.yaml`

For more advanced configuration (optional):

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
  model: gpt-5-high
  modelProvider: openai
  apiKey: ${OPENAI_API_KEY}
```

This file takes precedence over environment variables for detection and AI configuration.

## Database Setup

### 1. Configure Database Connection

Ensure `DATABASE_URL` in `.env` points to your database:

```env
DATABASE_URL=postgresql://username:password@localhost:5432/security_db
```

### 2. Run Migrations

```bash
cd ai-security-backend/packages/core
npx sequelize-cli db:migrate
```

This creates the following tables:
- `normalized_logs`
- `detections`
- `feedbacks`
- `feedback_patterns`
- `detection_logs` (junction table)

### 3. Verify Tables

```bash
psql security_db -c "\\dt"
```

You should see:
```
                List of relations
 Schema |       Name        | Type  |  Owner
--------+-------------------+-------+---------
 public | detections        | table | username
 public | detection_logs    | table | username
 public | feedback_patterns | table | username
 public | feedbacks         | table | username
 public | normalized_logs   | table | username
```

## Running the Application

### Development Mode

From the monorepo root:

```bash
npm run dev
```

Or from `packages/core`:

```bash
npm run dev -w @security/core
```

This starts:
1. **API Server** on port 3000
2. **MCP Detection Server** on port 3100
3. **MCP Advisor Server** on port 3101
4. **MCP Quality Server** on port 3102

### Production Mode

```bash
npm run build
npm run start
```

### Running Individual Components

```bash
npm run dev:api                # API server only
npm run mcp:http:detection     # MCP Detection server only
npm run mcp:http:advisor       # MCP Advisor server only
npm run mcp:http:quality       # MCP Quality server only
```

## Verification

### 1. Check API Health

```bash
curl http://localhost:3000/
```

Expected response:

```json
{
  "name": "Security API",
  "version": "0.1.0",
  "status": "running",
  "endpoints": {
    "detections": "/api/detections",
    "logs": "/api/logs",
    "stats": "/api/stats",
    "feedback": "/api/feedback",
    "search": "/api/search",
    "config": "/api/config",
    "rules": "/api/rules",
    "health": "/api/health",
    "websocket": "/ws/detections"
  }
}
```

### 2. Check MCP Servers

```bash
curl http://127.0.0.1:3100/health
curl http://127.0.0.1:3101/health
curl http://127.0.0.1:3102/health
```

Expected response for each:

```json
{
  "status": "ok",
  "agentType": "detection",
  "activeSessions": 0
}
```

### 3. Check Logs

Watch the console output for:

```
API Server: http://0.0.0.0:3000
WebSocket: ws://0.0.0.0:3000/ws/detections
Detection Mode: polling
Polling Interval: 600000ms
Rules Loaded: 9
Batch Size: 100
```

### 4. Test WebSocket Connection

Using `wscat`:

```bash
npm install -g wscat
wscat -c ws://localhost:3000/ws/detections
```

You should see:
```
Connected (press CTRL+C to quit)
```

## Common Commands

### Development

```bash
npm run dev                 # Run in development mode (all services)
npm run dev:api             # Run API server only
npm run build:watch         # Build in watch mode
```

### Building

```bash
npm run build               # Build for production
```

### Testing

```bash
npm run test                # Run all tests
npm run test:watch          # Run tests in watch mode
npm run test:coverage       # Run tests with coverage report
```

### Database

```bash
npx sequelize-cli migration:generate --name migration-name
npx sequelize-cli db:migrate
npx sequelize-cli db:migrate:undo
npx sequelize-cli db:migrate:undo:all
```

### Linting

```bash
npm run lint                # Check for linting issues
npm run lint:fix            # Auto-fix linting issues
```

### MCP Tools

```bash
npm run mcp:inspect:detection    # Inspect Detection MCP server
npm run mcp:inspect:advisor      # Inspect Advisor MCP server
npm run mcp:inspect:quality      # Inspect Quality MCP server
```

## Development vs Production

### Development Mode

- Hot reload enabled
- Detailed logging (debug level)
- Source maps available
- MCP servers run separately for debugging
- CORS enabled for frontend development

### Production Mode

- Compiled JavaScript (no source maps)
- Optimized logging (info/warn/error levels)
- All services run in single process
- Environment-based configuration
- CORS configured for production domains

### Production Deployment Checklist

- [ ] Build the application: `npm run build`
- [ ] Set `NODE_ENV=production`
- [ ] Configure production database
- [ ] Configure CORS origins for production domains
- [ ] Set up process manager (PM2, systemd)
- [ ] Configure logging to files
- [ ] Set up monitoring and alerts
- [ ] Run database migrations

## Troubleshooting

### Database Connection Issues

**Error**: `ECONNREFUSED` or `connection refused`

**Solution**:
1. Verify PostgreSQL is running:
   ```bash
   pg_isready
   ```

2. Check `DATABASE_URL` format:
   ```env
   DATABASE_URL=postgresql://username:password@localhost:5432/dbname
   ```

3. Test connection manually:
   ```bash
   psql $DATABASE_URL -c "SELECT 1;"
   ```

### Port Already in Use

**Error**: `EADDRINUSE` on port 3000/3100/3101/3102

**Solution**:
1. Find process using port:
   ```bash
   lsof -i :3000
   ```

2. Kill process or change port in `.env`:
   ```env
   API_PORT=3001
   ```

### Missing OpenAI API Key

**Error**: AI agents fail with authentication error

**Solution**:
1. Verify `OPENAI_API_KEY` is set in `.env`
2. Check API key validity at OpenAI dashboard
3. For local models, configure model provider in `config.yaml`

### Module Not Found Errors

**Error**: `Cannot find module` or `ERR_MODULE_NOT_FOUND`

**Solution**:
1. Ensure all imports use `.js` extensions (ES modules):
   ```typescript
   import { something } from './module.js';
   ```

2. Reinstall dependencies:
   ```bash
   rm -rf node_modules package-lock.json
   npm install
   ```

### Migration Failures

**Error**: Sequelize migration errors

**Solution**:
1. Check database connection
2. Verify Sequelize CLI configuration in `.sequelizerc`
3. Run migrations from correct directory:
   ```bash
   cd packages/core
   npx sequelize-cli db:migrate
   ```

### MCP Server Not Starting

**Error**: MCP servers fail to start

**Solution**:
1. Check if ports 3100, 3101, 3102 are available
2. Verify database connection (MCP servers need DB access)
3. Check logs for specific error messages

For more help, see the inline code documentation and relevant documentation sections.

## Next Steps

After successful setup:

1. **Review Architecture**: Read [Architecture Overview](./architecture.md)
2. **Understand Components**: See [Core Components](./core-components.md)
3. **Configure Rules**: Learn about [Rule System](./rule-system.md)
4. **Test API**: Explore [API Reference](./api-reference.md)
5. **Customize Agents**: Check [AI Agents](./ai-agents.md)

---