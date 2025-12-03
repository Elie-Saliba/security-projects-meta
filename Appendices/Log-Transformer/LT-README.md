# Log-Transformer Documentation

Welcome to the comprehensive documentation for the Log-Transformer platform - a universal log ingestion and normalization system designed for scalability, extensibility, and seamless integration with security analysis platforms.

## 📚 Documentation Structure

### [1. Getting Started](./1-getting-started.md)
**Start here if you're new to Log-Transformer**

- Prerequisites and system requirements
- Installation instructions (Docker & Local)
- Database setup and initialization
- Running the application
- Quick start examples
- Integration with AI-Security platform
- Troubleshooting common issues

**Who should read this**: New users, DevOps engineers, anyone setting up Log-Transformer for the first time.

---

### [2. API Reference](./2-api-reference.md)
**Complete API documentation**

- Base URLs and authentication
- File upload endpoints
- Job management endpoints
- Real-time ingestion endpoints (planned)
- Data models and schemas
- Error handling
- Rate limiting
- Swagger/OpenAPI documentation

**Who should read this**: Developers integrating with Log-Transformer, automation engineers, security engineers building workflows.

---

### [3. Architecture](./3-architecture.md)
**System design and architectural patterns**

- High-level system architecture
- Component diagrams and interactions
- Design patterns used
- Processing pipeline flow
- Scalability architecture
- Fault tolerance mechanisms
- Performance characteristics
- Technology stack
- Future enhancements

**Who should read this**: Architects, senior engineers, anyone wanting to understand how the system works internally.

---

### [4. Configuration](./4-configuration.md)
**Complete configuration guide**

- Configuration hierarchy and precedence
- Configuration file structures
- Database configuration
- Storage settings
- Import/processing settings
- Environment variables
- Docker and container configuration
- Security best practices
- Configuration by environment (dev/staging/prod)
- Advanced configuration options

**Who should read this**: DevOps engineers, system administrators, anyone deploying or configuring Log-Transformer.

---

### [5. Core Components](./5-core-components.md)
**Detailed breakdown of each system component**

- API Layer
- Job Queue System
- Background Worker (IngestWorker)
- Parser Registry (Plugin System)
- Normalizer
- Batch Writer
- Data Layer (Entity Framework Core)
- Component interactions
- Testing components
- Dependency relationships

**Who should read this**: Developers, contributors, anyone extending or customizing the platform.

---

### [6. Database](./6-database.md)
**Database schema and optimization**

- Database architecture and design principles
- Table schemas (`normalized_logs`, `ingest_jobs`)
- Enum types
- JSONB storage patterns
- Indexing strategies
- Query optimization
- Partitioning strategy
- Backup and recovery
- Monitoring and maintenance
- Security and access control

**Who should read this**: Database administrators, performance engineers, anyone optimizing queries or managing data.

---

### [7. Scalability and Extensibility](./7-scalability-and-extensibility.md)
**Adding new sources and scaling the platform**

- **Extensibility**:
  - Adding new log sources (step-by-step)
  - Real-world integration examples (FluentBit, Wazuh, Syslog)
  - Parser implementation guide
  - Plugin registration
  
- **Scalability**:
  - Horizontal scaling (API and workers)
  - Vertical scaling (memory, CPU, I/O)
  - Database scaling strategies
  - Performance benchmarks
  - Bottleneck identification
  - Monitoring and observability

**Who should read this**: Anyone planning to add new log sources, scale the platform, or optimize performance.

---

### [8. End-to-End Flow](./8-end-to-end-flow.md)
**Complete data journey from source to analysis**

- System architecture overview
- File-based ingestion flow (detailed walkthrough)
- Real-time stream ingestion flow
- Multi-source correlation example
- Data lifecycle phases
- Integration points
- Performance characteristics
- Monitoring the flow
- Failure scenarios and recovery

**Who should read this**: Everyone! This provides the "big picture" of how everything works together.

---

## 🚀 Quick Navigation

### I want to...

**...get started quickly**
→ [Getting Started](./1-getting-started.md) → Quick Start Example

**...integrate Log-Transformer with my application**
→ [API Reference](./2-api-reference.md) → Endpoints

**...add support for a new log source**
→ [Scalability and Extensibility](./7-scalability-and-extensibility.md) → Adding New Log Sources

**...understand how data flows through the system**
→ [End-to-End Flow](./8-end-to-end-flow.md)

**...configure the application for production**
→ [Configuration](./4-configuration.md) → Production Configuration

**...optimize database performance**
→ [Database](./6-database.md) → Performance Optimization

**...understand the architecture**
→ [Architecture](./3-architecture.md)

**...contribute to the codebase**
→ [Core Components](./5-core-components.md)

**...troubleshoot issues**
→ [Getting Started](./1-getting-started.md) → Troubleshooting

**...scale the platform**
→ [Scalability and Extensibility](./7-scalability-and-extensibility.md) → Scalability

---

## 📖 Reading Paths

### Path 1: New User Getting Started
1. [Getting Started](./1-getting-started.md) - Installation and setup
2. [API Reference](./2-api-reference.md) - Learn the API
3. [End-to-End Flow](./8-end-to-end-flow.md) - Understand the complete system

### Path 2: Developer/Contributor
1. [Architecture](./3-architecture.md) - Understand the design
2. [Core Components](./5-core-components.md) - Learn the internals
3. [Scalability and Extensibility](./7-scalability-and-extensibility.md) - Add features

### Path 3: Operations/DevOps
1. [Getting Started](./1-getting-started.md) - Deploy the application
2. [Configuration](./4-configuration.md) - Configure for your environment
3. [Database](./6-database.md) - Manage data and performance

### Path 4: Architect/Decision Maker
1. [Architecture](./3-architecture.md) - Understand the design
2. [Scalability and Extensibility](./7-scalability-and-extensibility.md) - Evaluate capabilities
3. [End-to-End Flow](./8-end-to-end-flow.md) - See the complete picture

---

## 🎯 Key Features Highlighted in Documentation

### Universal Log Ingestion
- Support for multiple log formats (EVTX, JSONL, Syslog, and more)
- Plugin-based architecture for easy extension
- File upload and real-time streaming support

### Normalization and Standardization
- Unified schema for all log sources
- JSONB flexibility for diverse data
- Consistent severity mapping
- Timezone normalization

### High Performance
- Streaming processing for memory efficiency
- Batch database writes for throughput
- Optimized indexing for fast queries
- 10K-50K events/second processing rate

### Scalability
- Horizontal scaling (multiple API and worker instances)
- Stateless design
- Database partitioning support
- Load balancer ready

### Reliability
- Fault-tolerant job processing
- Transaction-based consistency
- Error recovery mechanisms
- Complete audit trail

### Integration
- REST API with OpenAPI/Swagger
- Shared database with AI-Security platform
- Message queue support (planned)
- SIEM/SOAR integration ready

---

## 🔧 Technology Stack

- **.NET 8.0** - Modern, cross-platform runtime
- **ASP.NET Core** - High-performance web framework
- **Entity Framework Core** - Object-relational mapping
- **PostgreSQL 16** - Robust relational database with JSONB support
- **Docker** - Containerization for consistent deployment
- **Npgsql** - High-performance PostgreSQL driver

---

## 📝 Documentation Conventions

### Code Examples
- **Bash/Shell**: Linux/macOS commands
- **PowerShell**: Windows commands
- **C#**: Application code
- **SQL**: Database queries
- **JSON**: Configuration files
- **YAML**: Docker Compose files

### Terminology
- **Log Source**: System generating logs (e.g., Windows Event Log, Syslog server)
- **Parser**: Component that reads and interprets a specific log format
- **Normalization**: Process of converting diverse formats to unified schema
- **Job**: Unit of work for background processing
- **Worker**: Background service that processes jobs
- **JSONB**: PostgreSQL's binary JSON storage format

---

## 🤝 Contributing to Documentation

If you find errors, unclear sections, or missing information:

1. **Create an issue** describing the problem
2. **Submit a pull request** with improvements
3. **Ask questions** in discussions

Good documentation is essential for a successful project. Your feedback helps make it better!

---

## 📞 Support and Resources

- **GitHub Repository**: [Log-Transformer](https://github.com/your-org/log-transformer)
- **Issue Tracker**: Report bugs and request features
- **Discussions**: Ask questions and share ideas
- **AI-Security Integration**: [AI-Security Documentation](../ai-security/README.md)

---

## 🔄 Documentation Version

**Last Updated**: November 7, 2025  
**Application Version**: 1.0.0  
**Covers**: Core platform, EVTX parser, JSONL parser, basic integrations

---

## 📋 Document Status

| Document | Status | Completeness |
|----------|--------|--------------|
| Getting Started | ✅ Complete | 100% |
| API Reference | ✅ Complete | 100% |
| Architecture | ✅ Complete | 100% |
| Configuration | ✅ Complete | 100% |
| Core Components | ✅ Complete | 100% |
| Database | ✅ Complete | 100% |
| Scalability & Extensibility | ✅ Complete | 100% |
| End-to-End Flow | ✅ Complete | 100% |

---

**Happy logging! 🎉**

Build scalable, extensible log ingestion pipelines with Log-Transformer.
