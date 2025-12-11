# Appendix F-1.7 — Deployment Guide

# AI-Security Platform - Deployment Guide

Complete guide for setting up and running the AI-Security log analysis platform.

---

## Table of Contents

- [System Requirements](#system-requirements)
- [Prerequisites](#prerequisites)
- [Quick Start (Docker - Recommended)](#quick-start-docker---recommended)
- [Local Development Setup](#local-development-setup)
- [Configuration](#configuration)
- [Running the Application](#running-the-application)
- [Verification & Testing](#verification--testing)
- [Troubleshooting](#troubleshooting)

---

## System Requirements

### Minimum Requirements
- **CPU**: 2 cores
- **RAM**: 4 GB
- **Disk Space**: 10 GB free
- **OS**: Windows 10/11, macOS 10.15+, Linux (Ubuntu 20.04+)

### Recommended Requirements
- **CPU**: 4+ cores
- **RAM**: 8+ GB
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

2. **OpenAI API Key** (or Azure OpenAI credentials)
   - Required for AI-powered detection
   - Get from: https://platform.openai.com/api-keys

### For Local Development

1. **Node.js 18+** (LTS recommended)
   ```bash
   node --version  # Should show v18.0.0 or higher
   npm --version   # Should show 9.0.0 or higher
   ```
   Download from: https://nodejs.org/

2. **PostgreSQL 15+**
   ```bash
   psql --version  # Should show PostgreSQL 15+
   ```
   Download from: https://www.postgresql.org/download/

3. **Git**
   ```bash
   git --version
   ```
   Download from: https://git-scm.com/downloads

4. **OpenAI API Key** (or Azure OpenAI credentials)

---

## Quick Start (Docker - Recommended)

Docker deployment is the fastest way to get started. All services are containerized and configured automatically.

### Step 1: Clone the Repository

```bash
git clone <repository-url>
cd ai-security
```

### Step 2: Configure Environment

```bash
# Windows
copy .env.example .env

# Linux/macOS
cp .env.example .env
```

### Step 3: Set Required Credentials

Edit the `.env` file and configure:

```bash
# Required: Add your OpenAI API key
OPENAI_API_KEY=sk-your-actual-openai-key-here

# Optional: If using Azure OpenAI instead, uncomment and configure:
# AZURE_OPENAI_ENDPOINT=https://your-resource.openai.azure.com/openai
# AZURE_OPENAI_API_KEY=your-azure-key-here

# Model Configuration
CODEX_MODEL=gpt-4o
CODEX_MODEL_PROVIDER=openai
```

### Step 4: Start the Platform

```bash
# Windows
start.bat

# Linux/macOS
./start.sh
```

The script will:
- Create necessary directories
- Build Docker images
- Start all services (database, backend, frontend)
- Wait for services to become healthy
- Display access URLs

### Step 5: Access the Application

Once started, access:

- **Frontend Dashboard**: http://localhost:8080
- **Backend API**: http://localhost:3000
- **API Health Check**: http://localhost:3000/health
- **API Documentation**: http://localhost:3000/docs (if enabled)

### Docker Management Commands

```bash
# View logs
docker-compose logs -f

# View specific service logs
docker-compose logs -f backend
docker-compose logs -f frontend
docker-compose logs -f database

# Stop all services
docker-compose down

# Stop and remove volumes (clean slate)
docker-compose down -v

# Restart services
docker-compose restart

# Rebuild after code changes
docker-compose up --build -d

# Check service status
docker-compose ps
```

---

## Local Development Setup

For development without Docker, follow these steps.

### Step 1: Clone the Repository

```bash
git clone <repository-url>
cd ai-security
```

### Step 2: Database Setup

#### 2.1 Install PostgreSQL

Ensure PostgreSQL 15+ is installed and running.

#### 2.2 Create Database

```bash
# Connect to PostgreSQL
psql -U postgres

# Create database
CREATE DATABASE "Security";

# Exit
\q
```

#### 2.3 Initialize Schema

The schema will be automatically initialized when the backend starts for the first time.

### Step 3: Backend Setup

```bash
cd backend

# Install dependencies
npm install

# Build the codex-sdk package (required)
npm run build -w @elie/codex-sdk

# Build the core package (optional, can run in dev mode)
npm run build -w @security/core
```

### Step 4: Frontend Setup

```bash
cd frontend

# Install dependencies
npm install

# Build (optional for development)
npm run build
```

### Step 5: Configure Environment

Create `.env` file in the root directory:

```bash
# Database
POSTGRES_DB=Security
POSTGRES_USER=postgres
POSTGRES_PASSWORD=TestDb123
DATABASE_URL=postgresql://postgres:TestDb123@localhost:5432/Security

# Backend
NODE_ENV=development
PORT=3000
API_BASE_URL=http://localhost:3000

# Frontend
VITE_API_BASE_URL=http://localhost:3000
VITE_WS_URL=ws://localhost:3000

# Required: OpenAI Configuration
OPENAI_API_KEY=sk-your-actual-openai-key-here

# Model Configuration
CODEX_MODEL=gpt-4o
CODEX_MODEL_PROVIDER=openai

# CORS
CORS_ORIGIN=http://localhost:8080

# MCP Server Ports
MCP_DETECTION_PORT=3100
MCP_ADVISOR_PORT=3101
MCP_QUALITY_PORT=3102

# Detection Configuration
DETECTION_MODE=ai-only
POLLING_INTERVAL_MS=600000
BATCH_SIZE=10
GROUPING_MODE=host
GROUPING_ENABLED=true
GROUPING_TIME_WINDOW_MINUTES=5
GROUPING_INCLUDE_PROCESSED=true

# Logging
LOG_LEVEL=info
LOG_DIR=./logs

# Agent Configuration
AGENT_WORKSPACE_PATH=./data/agent-workspace
AGENT_WORKSPACE_TEMP_PATH=./data/agent-workspace/temp
MCP_LOG_LEVEL=info
```

---

## Configuration

### Environment Variables Reference

#### Database Configuration
| Variable | Description | Default | Required |
|----------|-------------|---------|----------|
| `DATABASE_URL` | PostgreSQL connection string | - | ✅ |
| `POSTGRES_DB` | Database name | Security | ✅ |
| `POSTGRES_USER` | Database user | postgres | ✅ |
| `POSTGRES_PASSWORD` | Database password | - | ✅ |

#### API Configuration
| Variable | Description | Default | Required |
|----------|-------------|---------|----------|
| `PORT` | Backend API port | 3000 | ✅ |
| `API_BASE_URL` | Backend API URL | http://localhost:3000 | ✅ |
| `CORS_ORIGIN` | Allowed CORS origin | http://localhost:8080 | ✅ |

#### LLM Provider Configuration
| Variable | Description | Default | Required |
|----------|-------------|---------|----------|
| `OPENAI_API_KEY` | OpenAI API key | - | ✅ (if using OpenAI) |
| `AZURE_OPENAI_ENDPOINT` | Azure OpenAI endpoint | - | ✅ (if using Azure) |
| `AZURE_OPENAI_API_KEY` | Azure OpenAI key | - | ✅ (if using Azure) |
| `CODEX_MODEL` | Model to use | gpt-4o | ✅ |
| `CODEX_MODEL_PROVIDER` | Provider (openai/azure) | openai | ✅ |

#### Detection Configuration
| Variable | Description | Default | Required |
|----------|-------------|---------|----------|
| `DETECTION_MODE` | Detection mode (ai-only/rule-only/hybrid) | ai-only | ✅ |
| `POLLING_INTERVAL_MS` | Detection polling interval | 600000 | No |
| `BATCH_SIZE` | Logs per batch | 10 | No |
| `GROUPING_MODE` | Grouping mode (host/host_user_ip) | host | No |
| `GROUPING_ENABLED` | Enable log grouping | true | No |
| `GROUPING_TIME_WINDOW_MINUTES` | Time window for grouping | 5 | No |

#### MCP Server Configuration
| Variable | Description | Default | Required |
|----------|-------------|---------|----------|
| `MCP_DETECTION_PORT` | Detection server port | 3100 | No |
| `MCP_ADVISOR_PORT` | Advisor server port | 3101 | No |
| `MCP_QUALITY_PORT` | Quality server port | 3102 | No |
| `MCP_LOG_LEVEL` | MCP logging level | info | No |

### Frontend Build Arguments

When building the frontend Docker image, you can override these:

```bash
docker build \
  --build-arg VITE_API_BASE_URL=http://your-api:3000 \
  --build-arg VITE_WS_URL=ws://your-api:3000 \
  -t ai-security-frontend \
  ./frontend
```

---

## Running the Application

### Docker Mode (Production-like)

```bash
# Start all services
docker-compose up -d

# View logs in real-time
docker-compose logs -f

# Stop services
docker-compose down
```

### Local Development Mode

You need **3 terminal windows**:

#### Terminal 1: Backend (API + MCP Servers)

```bash
cd backend/packages/core
npm run dev
```

This starts:
- Main API server on port 3000
- MCP Detection server on port 3100
- MCP Advisor server on port 3101
- MCP Quality server on port 3102

#### Terminal 2: Frontend

```bash
cd frontend
npm run dev
```

Frontend starts on: http://localhost:5173 (Vite dev server)

#### Terminal 3: Database (if not using Docker)

Ensure PostgreSQL is running:

```bash
# Windows (if installed as service)
# PostgreSQL should auto-start

# Linux
sudo systemctl start postgresql

# macOS
brew services start postgresql@15
```

### Alternative: Run Individual Services

```bash
# Backend API only
cd backend/packages/core
npm run dev:api

# Individual MCP servers
npm run mcp:http:detection  # Port 3100
npm run mcp:http:advisor    # Port 3101
npm run mcp:http:quality    # Port 3102

# Frontend
cd frontend
npm run dev
```

---

## Verification & Testing

### 1. Check Service Health

```bash
# Backend health
curl http://localhost:3000/health

# Expected response:
# {"status":"ok","timestamp":"2024-..."}
```

### 2. Check Frontend

Open browser: http://localhost:8080 (Docker) or http://localhost:5173 (dev)

You should see the AI-Security dashboard.

### 3. Check Database Connection

```bash
# Using Docker
docker exec -it ai-security-db psql -U postgres -d Security -c "SELECT COUNT(*) FROM normalized_logs;"

# Local PostgreSQL
psql -U postgres -d Security -c "SELECT COUNT(*) FROM normalized_logs;"
```

### 4. Test API Endpoints

```bash
# List logs
curl http://localhost:3000/api/logs

# Get detections
curl http://localhost:3000/api/detections

# Get rules
curl http://localhost:3000/api/rules
```

### 5. Check Docker Services

```bash
docker-compose ps

# All services should show "Up" status:
# ai-security-backend    Up (healthy)
# ai-security-frontend   Up (healthy)
# ai-security-db         Up (healthy)
```

---

## Troubleshooting

### Common Issues

#### 1. Docker Build Fails

**Problem**: Build errors during `docker-compose up --build`

**Solutions**:
```bash
# Clean Docker cache and rebuild
docker-compose down -v
docker system prune -a
docker-compose up --build
```

#### 2. Backend Won't Start

**Problem**: Backend container exits immediately

**Check logs**:
```bash
docker-compose logs backend
```

**Common causes**:
- Missing `OPENAI_API_KEY` in `.env`
- Database not ready (wait 30 seconds and restart)
- Port 3000 already in use

**Solutions**:
```bash
# Verify .env file exists and has API key
cat .env | grep OPENAI_API_KEY

# Check if port is in use
netstat -ano | findstr :3000  # Windows
lsof -i :3000                 # Linux/macOS

# Restart backend only
docker-compose restart backend
```

#### 3. Database Connection Errors

**Problem**: "Connection refused" or "ECONNREFUSED"

**Solutions**:
```bash
# Check database health
docker-compose ps database

# Restart database
docker-compose restart database

# Wait for healthy status
docker-compose logs database | grep "ready to accept connections"

# Verify connection string in .env
# Docker: database:5432
# Local: localhost:5432
```

#### 4. Frontend Shows Blank Page

**Problem**: Frontend loads but shows blank screen

**Solutions**:
```bash
# Check browser console for errors
# Common issue: API URL misconfigured

# Verify environment variables
docker-compose exec frontend printenv | grep VITE

# Rebuild frontend
docker-compose up -d --build frontend
```

#### 5. Permission Errors (Linux)

**Problem**: Permission denied errors

**Solutions**:
```bash
# Fix directory permissions
sudo chown -R $USER:$USER backend/packages/core/data
sudo chown -R $USER:$USER backend/packages/core/logs

# Or run Docker with correct user
docker-compose down
export USER_ID=$(id -u)
export GROUP_ID=$(id -g)
docker-compose up -d
```

#### 6. MCP Servers Not Starting

**Problem**: MCP server ports not accessible

**Check ports**:
```bash
# Verify MCP servers are running
curl http://localhost:3100/health  # Detection
curl http://localhost:3101/health  # Advisor
curl http://localhost:3102/health  # Quality
```

**Solutions**:
```bash
# Check backend logs for MCP errors
docker-compose logs backend | grep MCP

# Verify ports in .env
grep MCP .env
```

#### 7. Out of Memory

**Problem**: Services crash with memory errors

**Solutions**:
```bash
# Increase Docker memory (Docker Desktop > Settings > Resources)
# Minimum 4GB, recommended 8GB

# For local development, increase Node.js memory
export NODE_OPTIONS="--max-old-space-size=4096"
```

### Getting Help

If issues persist:

1. **Check logs**:
   ```bash
   docker-compose logs -f
   ```

2. **Verify configuration**:
   ```bash
   cat .env
   docker-compose config
   ```

3. **Clean restart**:
   ```bash
   docker-compose down -v
   rm -rf backend/packages/core/data/*
   rm -rf backend/packages/core/logs/*
   docker-compose up --build
   ```

4. **Check system resources**:
   ```bash
   docker stats
   ```

---

## Advanced Topics

### Custom Configuration

To use custom configuration:

1. Edit `backend/packages/core/config.yaml`
2. Rebuild Docker image or restart local service

### Production Deployment

For production deployment:

1. Use proper secrets management (not `.env` files)
2. Set `NODE_ENV=production`
3. Use reverse proxy (nginx/Traefik) for SSL
4. Configure proper database backups
5. Set up monitoring and logging
6. Use managed PostgreSQL service
7. Scale services using Docker Swarm or Kubernetes

### Database Migrations

Schema changes are automatically applied on startup. For manual migration:

```bash
cd backend/packages/core
npm run db:init
```

### Development Tools

```bash
# Run tests
cd backend/packages/core
npm test

# Run linting
npm run lint

# Fix linting issues
npm run lint:fix

# Build for production
npm run build
```

---

## Next Steps

- Read [Architecture Documentation](./as-architecture.md)
- Review [API Reference](./as-api-reference.md)
- Explore [Rule System](./as-rule-system.md)
- Configure [MCP Integration](./as-mcp-integration.md)

---

**Last Updated**: November 2024  
**Version**: 1.0.0
