# Starting Up Security Projects

This guide explains how to deploy the AI Security and Log-Transformer projects using Docker.

## Prerequisites

- Docker Desktop installed and running
- Git (to clone repositories)

## Repository Structure

This meta repository contains two main projects as Git submodules:

### ai-security/
The **AI Security Platform** - An intelligent log analysis system that uses AI agents to detect security threats and anomalies in log data. It provides a web-based frontend for visualization and management, and includes MCP (Model Context Protocol) servers for AI-powered analysis.

- **Location:** `security-projects-meta/ai-security/`
- **GitHub:** https://github.com/Elie-Saliba/AI-log-anaomaly-detection
- **Key Components:**
  - Frontend (React/TypeScript)
  - Backend API (Node.js)
  - PostgreSQL Database
  - Three MCP Servers (Detection, Advisor, Quality)

### Log-Transformer/
The **Log-Transformer API** - A .NET application that processes and transforms Windows Event Logs (.evtx files) into structured data. It automatically ingests logs from the uploads directory and stores them in the database for analysis by the AI Security Platform.

- **Location:** `security-projects-meta/Log-Transformer/`
- **GitHub:** https://github.com/Elie-Saliba/log-transformer
- **Key Components:**
  - .NET 8.0 API
  - Windows Event Log Parser
  - Swagger UI for API testing
  - Automatic file processing from uploads directory

## Important: Deployment Order

**AI Security must be deployed first** because it contains the shared PostgreSQL database that Log-Transformer connects to in integrated mode.

## Quick Start

### Method 1: Using Startup Scripts (Recommended)

The easiest way to start both projects with zero manual configuration:

```powershell
# 1. Start AI Security first
cd c:\DEV\security-projects-meta\ai-security
.\start.bat

# 2. Start Log-Transformer (automatically detects AI Security)
cd c:\DEV\security-projects-meta\Log-Transformer
.\start.bat
```

**Linux/Mac:**
```bash
# 1. Start AI Security first
cd /path/to/security-projects-meta/ai-security
./start.sh

# 2. Start Log-Transformer
cd /path/to/security-projects-meta/Log-Transformer
./start.sh
```

### Method 2: Using Docker Compose Directly

If you prefer using Docker Compose commands directly:

```powershell
# 1. Start AI Security first
cd c:\DEV\security-projects-meta\ai-security
docker-compose up --build -d

# 2. Start Log-Transformer
cd c:\DEV\security-projects-meta\Log-Transformer
docker-compose up --build -d
```

## What Happens on First Run

Both startup scripts automatically:
- Create `.env` files from `.env.example` templates if they don't exist
- Create required directories
- Build and start all Docker containers
- Wait for services to become healthy

**No manual configuration required!**

## Accessing the Applications

### AI Security Platform
- **Frontend:** http://localhost:8080
- **Backend API:** http://localhost:3000
- **Database:** localhost:5432
  - Database: `Security`
  - Username: `postgres`
  - Password: `TestDb123`

### Log-Transformer API
- **Swagger UI:** http://localhost:5001/swagger
- **API Base:** http://localhost:5001
- **Uploads Directory:** `Log-Transformer/uploads`

## Initial Configuration

### First-Time Model Provider Setup

When you access the AI Security frontend for the first time, you'll be prompted to configure a model provider. Use the following settings for testing:

- **Model Provider:** OpenAI
- **API Key:** (can be shared on demand)
- **Model:** gpt-5

This configuration enables the AI detection and analysis features.

## Verifying Deployment

Check all containers are running:
```powershell
docker ps
```

You should see 4 containers:
- `ai-security-frontend` - Status: healthy
- `ai-security-backend` - Status: healthy
- `ai-security-db` - Status: healthy
- `log-transformer-api` - Status: healthy

## Stopping the Applications

### Stop all services:
```powershell
# Stop AI Security
cd c:\DEV\security-projects-meta\ai-security
docker-compose down

# Stop Log-Transformer
cd c:\DEV\security-projects-meta\Log-Transformer
docker-compose down
```

### Stop and remove all data (volumes):
```powershell
# AI Security
cd c:\DEV\security-projects-meta\ai-security
docker-compose down -v

# Log-Transformer
cd c:\DEV\security-projects-meta\Log-Transformer
docker-compose down -v
```

## Troubleshooting

### Port Already in Use
If you see port conflict errors:
- Check if another application is using the ports
- Stop any existing containers: `docker ps -a` then `docker stop <container-id>`

### Database Connection Errors
- Ensure AI Security is running before starting Log-Transformer
- Check AI Security database container is healthy: `docker ps`
- Verify network exists: `docker network ls | findstr ai-security`

### Viewing Logs
```powershell
# AI Security logs
cd c:\DEV\security-projects-meta\ai-security
docker-compose logs -f

# Log-Transformer logs
cd c:\DEV\security-projects-meta\Log-Transformer
docker-compose logs -f
```

### Clean Start
To perform a completely fresh deployment:
```powershell
# Remove all containers and volumes
cd c:\DEV\security-projects-meta\ai-security
docker-compose down -v

cd c:\DEV\security-projects-meta\Log-Transformer
docker-compose down -v

# Remove shared network
docker network rm ai-security_ai-security-network

# Optional: Clean Docker cache
docker system prune -f

# Now start fresh
cd c:\DEV\security-projects-meta\ai-security
.\start.bat

cd c:\DEV\security-projects-meta\Log-Transformer
.\start.bat
```

## Architecture Notes

- **Integrated Mode (Default):** Log-Transformer connects to AI Security's PostgreSQL database
- **Shared Network:** Both projects communicate via `ai-security_ai-security-network`
- **Database:** Single PostgreSQL instance (ai-security-db) serves both applications
- **Health Checks:** All services include health checks for reliability

## Additional Resources

- AI Security Documentation: `ai-security/documentation/`
- Log-Transformer Documentation: `Log-Transformer/documentation/`
- Docker Mode Details: `Log-Transformer/DOCKER-MODES.md`
- Quick Start Guide: `security-projects-meta/QUICK-START.md`
