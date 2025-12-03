# Quick Start Guide

## 🎯 Repository URLs

- **Meta Repository**: https://github.com/Elie-Saliba/security-projects-meta
- **AI Security**: https://github.com/Elie-Saliba/AI-log-anaomaly-detection
- **Log Transformer**: https://github.com/Elie-Saliba/log-transformer

## 📥 Clone Everything

```bash
git clone --recurse-submodules https://github.com/Elie-Saliba/security-projects-meta.git
cd security-projects-meta
```

## 🔄 Update Submodules

Pull latest changes from all submodules:
```bash
git submodule update --remote --merge
```

## 📁 What's Inside

```
security-projects-meta/
├── ai-security/         → AI-powered log anomaly detection
├── Log-Transformer/     → ASP.NET log transformation system  
├── Appendices/         → Shared documentation (84 files)
├── README.md           → Full documentation
└── .gitmodules         → Submodule configuration
```

## 🚀 Running Projects

### AI Security (Node.js)
```bash
cd ai-security
docker-compose up -d
# Access at http://localhost:3000
```

### Log Transformer (.NET)
```bash
cd Log-Transformer
docker-compose up -d
# Access at http://localhost:5000
```

## 📚 Documentation

- **AI Security Docs**: `Appendices/ai-security/`
- **Log Transformer Docs**: `Appendices/Log-Transformer/`
- **Deployment Guides**: Check individual `Appendices/Docker Deployment Instruction*.md`
- **Screenshots**: `Appendices/Documentation Screenshots/`

## 🔧 Working with Submodules

### Make changes in a submodule:
```bash
cd ai-security  # or Log-Transformer
# Make your changes
git add .
git commit -m "Your changes"
git push

# Update parent repo
cd ..
git add ai-security  # or Log-Transformer
git commit -m "Update submodule reference"
git push
```

## 💡 Tips

- Always use `--recurse-submodules` when cloning
- Submodules point to specific commits (not branches)
- Update submodules regularly with `git submodule update --remote`
- Each submodule is a full Git repository

## 🆘 Troubleshooting

### Submodules not initialized?
```bash
git submodule init
git submodule update
```

### Submodules out of date?
```bash
git submodule update --remote --merge
git add .
git commit -m "Update submodules to latest"
git push
```

### Want to work on a specific branch in submodule?
```bash
cd ai-security
git checkout your-branch
cd ..
```
