# Security Projects - Meta Repository

This meta-repository serves as a unified hub for security-related projects and documentation, combining multiple related repositories and shared resources.

## 📁 Repository Structure

```
security-projects-meta/
├── ai-security/              # Git submodule: AI-powered log anomaly detection system
├── Log-Transformer/          # Git submodule: Log transformation and processing system
├── Appendices/              # Shared documentation and appendices
└── README.md               # This file
```

## 🔗 Submodules

### AI Log Anomaly Detection (`ai-security`)
- **Repository**: [AI-log-anaomaly-detection](https://github.com/Elie-Saliba/AI-log-anaomaly-detection)
- **Description**: AI-powered system for detecting security anomalies in log files
- **Technologies**: Node.js, Docker, PostgreSQL, Ollama

### Log Transformer (`Log-Transformer`)
- **Repository**: [log-transformer](https://github.com/Elie-Saliba/log-transformer)
- **Description**: ASP.NET Core-based log transformation and processing system
- **Technologies**: .NET 8, C#, Docker, PostgreSQL

## 📚 Appendices

The `Appendices/` directory contains shared documentation, guides, and reference materials that apply across both projects:

- API documentation
- Architecture diagrams
- Deployment guides
- User guides
- Testing reports
- And more...

## 🚀 Getting Started

### Clone with Submodules

To clone this repository with all submodules:

```bash
git clone --recurse-submodules https://github.com/Elie-Saliba/security-projects-meta.git
```

### Initialize Submodules (if already cloned)

If you've already cloned the repository without submodules:

```bash
git submodule init
git submodule update
```

### Update All Submodules

To pull the latest changes from all submodules:

```bash
git submodule update --remote --merge
```

## 🛠️ Working with Submodules

### Navigate to a Submodule

```bash
cd ai-security
# or
cd Log-Transformer
```

### Make Changes in a Submodule

1. Navigate to the submodule directory
2. Make your changes
3. Commit and push from within the submodule:
   ```bash
   cd ai-security
   git add .
   git commit -m "Your commit message"
   git push
   ```
4. Update the parent repository to reference the new commit:
   ```bash
   cd ..
   git add ai-security
   git commit -m "Update ai-security submodule"
   git push
   ```

## 📖 Documentation

- **AI Security Documentation**: See `ai-security/documentation/`
- **Log Transformer Documentation**: See `Log-Transformer/documentation/`
- **Shared Documentation**: See `Appendices/`

## 🤝 Contributing

Each submodule has its own contribution guidelines. Please refer to the respective repositories for details:

- [AI Security Contributing](https://github.com/Elie-Saliba/AI-log-anaomaly-detection)
- [Log Transformer Contributing](https://github.com/Elie-Saliba/log-transformer)

## 📝 License

Each submodule maintains its own license. Please refer to the individual repositories for licensing information.

## 👤 Author

**Elie Saliba**
- GitHub: [@Elie-Saliba](https://github.com/Elie-Saliba)

## 🔄 Last Updated

December 3, 2025
