# Starting Up Security Projects (Local Development)

This guide explains how to build and run the AI Security and Log-Transformer projects locally without Docker.

## Prerequisites

**Required for local development:**

### System Requirements
- **Node.js**: Version 18+ with npm
- **PostgreSQL**: Version 13+ installed and running
- **.NET SDK**: Version 8.0 or higher
- **Git**: For cloning repositories
- **Windows**: For processing .evtx files (Log-Transformer component)

### Optional but Recommended
- **Visual Studio Code** or **Visual Studio 2022** (for .NET development)
- **pgAdmin** or similar PostgreSQL management tool
- **Postman** or similar API testing tool

### Ports Required
- **3000**: Backend API
- **8080**: Frontend development server
- **5001**: Log-Transformer API
- **5432**: PostgreSQL database
- **3100-3102**: MCP (Model Context Protocol) servers

---

## Repository Structure

This meta repository contains two main projects as Git submodules:

### ai-security/
The **AI Security Platform** - An intelligent log analysis system that uses AI agents to detect security threats and anomalies in log data.

- **Location:** `security-projects-meta/ai-security/`
- **GitHub:** https://github.com/Elie-Saliba/AI-log-anaomaly-detection
- **Stack:** Node.js (TypeScript) + React + PostgreSQL

### Log-Transformer/
The **Log-Transformer API** - A .NET application that processes and transforms Windows Event Logs (.evtx files) into structured data.

- **Location:** `security-projects-meta/Log-Transformer/`
- **GitHub:** https://github.com/Elie-Saliba/log-transformer
- **Stack:** .NET 8.0 (C#) + PostgreSQL

---

## Important: Deployment Order

**The database must be set up first**, followed by AI Security, then Log-Transformer.

---

## Step-by-Step Setup

### Step 1: PostgreSQL Database Setup

#### 1.1 Install PostgreSQL
If not already installed, download and install PostgreSQL from https://www.postgresql.org/download/

#### 1.2 Create Database
```powershell
# Connect to PostgreSQL (as postgres user)
psql -U postgres

# Create the database
CREATE DATABASE "Security";

# Exit psql
\q
```

#### 1.3 Run Database Initialization Scripts
```powershell
# Navigate to ai-security database initialization
cd c:\DEV\security-projects-meta\ai-security\database\init

# Run the initialization script
psql -U postgres -d Security -f 01-init.sql
```

**Note:** The initialization script creates all necessary tables including `normalized_logs` which is used by both projects.

#### 1.4 Verify Database Setup
```powershell
# Connect to the database
psql -U postgres -d Security

# List tables
\dt

# Should see tables like: normalized_logs, detections, rules, etc.
\q
```

---

### Step 2: AI Security Platform Setup

#### 2.1 Configure Environment Variables
```powershell
cd c:\DEV\security-projects-meta\ai-security

# Create .env file from template
Copy-Item .env.example .env

# Edit .env file with your settings
notepad .env
```

**Required .env Configuration:**
```env
# Database (for local PostgreSQL)
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

# LLM Provider (choose one)
OPENAI_API_KEY=sk-your-key-here
# OR for Azure OpenAI:
# AZURE_OPENAI_ENDPOINT=https://your-resource.openai.azure.com/openai
# AZURE_OPENAI_API_KEY=your-key

# CORS
CORS_ORIGIN=http://localhost:8080

# MCP Ports
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

#### 2.2 Install Backend Dependencies
```powershell
cd c:\DEV\security-projects-meta\ai-security\backend

# Install all workspace dependencies
npm install
```

#### 2.3 Build Backend
```powershell
# Build the SDK package first
npm run build -w @elie/codex-sdk

# Build the core backend
npm run build -w @security/core
```

#### 2.4 Install Frontend Dependencies
```powershell
cd c:\DEV\security-projects-meta\ai-security\frontend

# Install dependencies
npm install
```

#### 2.5 Build Frontend (Optional for Development)
```powershell
# Build for production
npm run build

# OR just run development server (see Step 3)
```

---

### Step 3: Start AI Security Platform

You need **three terminal windows** to run all components:

#### Terminal 1: Backend API
```powershell
cd c:\DEV\security-projects-meta\ai-security\backend
npm run dev
```

**Expected Output:**
```
[INFO] Server starting on port 3000
[INFO] Database connected
[INFO] MCP servers initialized
[INFO] Backend API ready at http://localhost:3000
```

#### Terminal 2: Frontend Development Server
```powershell
cd c:\DEV\security-projects-meta\ai-security\frontend
npm run dev
```

**Expected Output:**
```
VITE vX.X.X  ready in XXX ms

➜  Local:   http://localhost:8080/
➜  Network: use --host to expose
```

#### Terminal 3: MCP Servers (Optional but Recommended)
The MCP servers provide AI-powered detection capabilities. They should start automatically with the backend, but you can monitor them:

```powershell
cd c:\DEV\security-projects-meta\ai-security\backend
npm run dev -w @security/core
```

**Verify Services:**
- **Frontend:** http://localhost:8080
- **Backend Health:** http://localhost:3000/health
- **Backend API:** http://localhost:3000/api

---

### Step 4: Log-Transformer Setup

#### 4.1 Configure Environment Variables
```powershell
cd c:\DEV\security-projects-meta\Log-Transformer

# Create .env file from template
Copy-Item .env.example .env

# Edit .env file
notepad .env
```

**Required .env Configuration for Local Development:**
```env
# Database - Connect to local PostgreSQL
POSTGRES_DB=Security
POSTGRES_USER=postgres
POSTGRES_PASSWORD=TestDb123
POSTGRES_CONNECTION_STRING=Host=localhost;Port=5432;Database=Security;Username=postgres;Password=TestDb123

# API Configuration
ASPNETCORE_ENVIRONMENT=Development
ASPNETCORE_URLS=http://localhost:5001

# Import Settings
IMPORT_BATCH_SIZE=1000
IMPORT_POLL_INTERVAL=5

# Storage
STORAGE_UPLOAD_PATH=./uploads
```

#### 4.2 Restore .NET Dependencies
```powershell
cd c:\DEV\security-projects-meta\Log-Transformer

# Restore NuGet packages
dotnet restore SecurityAds.sln
```

#### 4.3 Build the Solution
```powershell
# Build the entire solution
dotnet build SecurityAds.sln --configuration Release

# OR build just the API project
dotnet build src/Api/Api.csproj --configuration Release
```

#### 4.4 Create Uploads Directory
```powershell
# Create uploads directory if it doesn't exist
New-Item -ItemType Directory -Force -Path "c:\DEV\security-projects-meta\Log-Transformer\uploads"
```

---

### Step 5: Start Log-Transformer

#### Terminal 4: Log-Transformer API
```powershell
cd c:\DEV\security-projects-meta\Log-Transformer\src\Api

# Run the API
dotnet run --configuration Release
```

**OR run from solution root:**
```powershell
cd c:\DEV\security-projects-meta\Log-Transformer
dotnet run --project src/Api/Api.csproj --configuration Release
```

**Expected Output:**
```
info: Microsoft.Hosting.Lifetime[14]
      Now listening on: http://localhost:5001
info: Microsoft.Hosting.Lifetime[0]
      Application started. Press Ctrl+C to shut down.
```

**Verify Service:**
- **Swagger UI:** http://localhost:5001/swagger
- **Health Check:** http://localhost:5001/health

---

## Initial Configuration

### First-Time Model Provider Setup

When you access the AI Security frontend for the first time at http://localhost:8080, you'll be prompted to configure a model provider:

1. Navigate to **Settings** → **Configuration**
2. Select **Model Provider**: OpenAI or Azure OpenAI
3. Enter your **API Key**
4. Select **Model**: e.g., `gpt-4` or `gpt-3.5-turbo`
5. Save configuration

This enables the AI detection and analysis features.

---

## Verifying the Setup

### 1. Check All Services Are Running
Open a new PowerShell terminal and verify all components:

```powershell
# Check if backend is responding
curl http://localhost:3000/health

# Check if frontend is accessible
curl http://localhost:8080

# Check if Log-Transformer API is responding
curl http://localhost:5001/health
```

### 2. Check Database Connections
```powershell
# Connect to database
psql -U postgres -d Security

# Check if tables exist
\dt

# Check if normalized_logs table has the correct schema
\d normalized_logs

# Exit
\q
```

### 3. Test Log Upload (Optional)
1. Open Swagger UI: http://localhost:5001/swagger
2. Navigate to **POST /api/logs/upload**
3. Upload a sample .evtx file
4. Check the response for job ID
5. Verify logs appear in AI Security frontend

---

## Development Workflow

### Making Changes

#### Backend Changes (AI Security)
1. Edit files in `backend/packages/core/src/` or `backend/packages/codex-sdk/src/`
2. Save changes (TypeScript will recompile automatically in dev mode)
3. Backend will auto-reload
4. Test changes at http://localhost:3000

#### Frontend Changes (AI Security)
1. Edit files in `frontend/src/`
2. Save changes (Vite will hot-reload automatically)
3. Changes appear instantly in browser at http://localhost:8080

#### Log-Transformer Changes
1. Edit files in `src/Api/`, `src/Shared/`, or `src/Importer/`
2. Stop the running API (Ctrl+C)
3. Rebuild: `dotnet build`
4. Restart: `dotnet run --project src/Api/Api.csproj`

### Running Tests

#### AI Security Tests
```powershell
cd c:\DEV\security-projects-meta\ai-security\backend
npm run test
```

#### Log-Transformer Tests
```powershell
cd c:\DEV\security-projects-meta\Log-Transformer
dotnet test SecurityAds.sln
```

---

## Troubleshooting

### Database Connection Issues

**Problem:** Cannot connect to PostgreSQL
```
Error: connect ECONNREFUSED 127.0.0.1:5432
```

**Solutions:**
1. Verify PostgreSQL is running:
   ```powershell
   # Check PostgreSQL service
   Get-Service postgresql*
   
   # If not running, start it
   Start-Service postgresql-x64-XX
   ```

2. Verify connection string in `.env` files
3. Check PostgreSQL is listening on port 5432:
   ```powershell
   netstat -an | findstr :5432
   ```

### Port Already in Use

**Problem:** Port 3000, 5001, or 8080 already in use

**Solutions:**
1. Find process using the port:
   ```powershell
   netstat -ano | findstr :3000
   ```

2. Kill the process:
   ```powershell
   taskkill /PID <PID> /F
   ```

3. OR change port in `.env` files

### Backend Build Errors

**Problem:** TypeScript compilation errors

**Solutions:**
1. Clear node_modules and reinstall:
   ```powershell
   cd backend
   Remove-Item -Recurse -Force node_modules
   npm install
   ```

2. Clear build cache:
   ```powershell
   npm run build -w @elie/codex-sdk
   npm run build -w @security/core
   ```

### Frontend Not Loading

**Problem:** Blank page or connection errors

**Solutions:**
1. Check backend is running at http://localhost:3000
2. Verify `VITE_API_BASE_URL` in frontend `.env`
3. Clear browser cache or use incognito mode
4. Check browser console for errors (F12)

### Log-Transformer Build Errors

**Problem:** .NET build fails

**Solutions:**
1. Verify .NET SDK version:
   ```powershell
   dotnet --version
   # Should be 8.0 or higher
   ```

2. Clean and rebuild:
   ```powershell
   dotnet clean
   dotnet restore
   dotnet build
   ```

3. Check NuGet sources:
   ```powershell
   dotnet nuget list source
   ```

### Missing Environment Variables

**Problem:** Application starts but features don't work

**Solutions:**
1. Verify `.env` file exists in each project root
2. Check all required variables are set
3. Restart services after changing `.env` files
4. Ensure API keys are valid (OpenAI, Azure, etc.)

### Database Schema Issues

**Problem:** Tables not found or schema mismatch

**Solutions:**
1. Re-run initialization script:
   ```powershell
   cd c:\DEV\security-projects-meta\ai-security\database\init
   psql -U postgres -d Security -f 01-init.sql
   ```

2. Check table schema:
   ```powershell
   psql -U postgres -d Security -c "\d normalized_logs"
   ```

3. If needed, drop and recreate database:
   ```powershell
   psql -U postgres -c "DROP DATABASE \"Security\";"
   psql -U postgres -c "CREATE DATABASE \"Security\";"
   # Then re-run init script
   ```

---

## Stopping the Applications

### Stop All Services
Press `Ctrl+C` in each terminal window running a service.

### Proper Shutdown Order
1. Stop Log-Transformer API (Terminal 4)
2. Stop Frontend Dev Server (Terminal 2)
3. Stop Backend API (Terminal 1)
4. PostgreSQL can remain running for next session

### Complete Cleanup
```powershell
# Stop all Node.js processes
Get-Process node | Stop-Process -Force

# Stop all .NET processes
Get-Process dotnet | Stop-Process -Force

# Optional: Stop PostgreSQL service
Stop-Service postgresql-x64-XX
```

---

## Production Build

### Building for Production

#### AI Security Platform
```powershell
# Build backend
cd c:\DEV\security-projects-meta\ai-security\backend
npm run build

# Build frontend
cd c:\DEV\security-projects-meta\ai-security\frontend
npm run build
# Output will be in frontend/dist/
```

#### Log-Transformer
```powershell
cd c:\DEV\security-projects-meta\Log-Transformer
dotnet publish src/Api/Api.csproj -c Release -o ./publish
# Output will be in publish/
```

### Running Production Builds

#### AI Security Backend (Production)
```powershell
cd c:\DEV\security-projects-meta\ai-security\backend
set NODE_ENV=production
node packages/core/dist/index.js
```

#### AI Security Frontend (Production)
```powershell
cd c:\DEV\security-projects-meta\ai-security\frontend
npm run preview
# Or serve the dist/ folder with any static file server
```

#### Log-Transformer (Production)
```powershell
cd c:\DEV\security-projects-meta\Log-Transformer\publish
.\Api.exe
```

---

## Additional Resources

- **AI Security Documentation:** `ai-security/documentation/`
- **Log-Transformer Documentation:** `Log-Transformer/documentation/`
- **Docker Deployment Guide:** `Appendices/Starting-Up-Projects-Docker.md`
- **API Reference:** 
  - AI Security API: `documentation/as-api-reference.md`
  - Log-Transformer API: http://localhost:5001/swagger
- **Architecture Diagrams:**
  - AI Security: `documentation/as-architecture.md`
  - Log-Transformer: `Log-Transformer/documentation/LT-3-architecture.md`

---

## Quick Reference Commands

### Start Everything (4 Terminals)
```powershell
# Terminal 1: Backend
cd c:\DEV\security-projects-meta\ai-security\backend; npm run dev

# Terminal 2: Frontend
cd c:\DEV\security-projects-meta\ai-security\frontend; npm run dev

# Terminal 3: Database (if not running)
Start-Service postgresql-x64-XX

# Terminal 4: Log-Transformer
cd c:\DEV\security-projects-meta\Log-Transformer\src\Api; dotnet run
```

### Access Points
- **Frontend:** http://localhost:8080
- **Backend API:** http://localhost:3000
- **Backend Health:** http://localhost:3000/health
- **Log-Transformer Swagger:** http://localhost:5001/swagger
- **Log-Transformer API:** http://localhost:5001

### Database Access
```powershell
psql -U postgres -d Security
```

### View Logs
- **Backend:** Check terminal output or `./logs/` directory
- **Frontend:** Browser console (F12)
- **Log-Transformer:** Check terminal output
- **PostgreSQL:** Check PostgreSQL logs directory
