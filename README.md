# Self-Hosted n8n with Postgres, SearXNG, and Ollama: Complete Setup Guide

This README provides a step-by-step guide to deploying a fully self-hosted automation stack using **n8n** (workflow automation), **Postgres** (database), **SearXNG** (privacy-focused search engine), and **Ollama** (local LLM for AI agents).

---

## Table of Contents
- Features
- Prerequisites
- Project Structure
- Environment Configuration
- Docker Compose Setup
- Starting, Stopping, and Restarting Services
- Post-Setup Configuration
  - Downloading Ollama Model
  - n8n Credentials Setup
    - Postgres (Already Configured)
    - SearXNG
    - Ollama
- Connecting AI Agent in n8n
- Sample Workflow
- Troubleshooting
- Security Notes
- Upgrading and Maintenance

---

## Features

- **n8n**: Visual workflow builder with AI agents supporting tools like search.
- **Postgres**: Persistent database for n8n workflows/executions.
- **SearXNG**: Metasearch engine (aggregates Google, DuckDuckGo, etc.) for privacy-focused web queries.
- **Ollama**: Local LLM (e.g., Llama 3.1) for AI reasoning and tool calling.
- **Dockerized**: One-command deploy; all services share internal communication.
- **Persistent Storage**: Volumes for data, models, and configs to survive restarts.

---

## Prerequisites

**Docker & Docker Compose**: Install Docker Desktop (Windows/Mac) or Docker Engine + Compose (Linux).  
Verify:
```bash
docker --version
docker compose version
```

**Hardware:**
- CPU: 4+ cores
- RAM: 8GB minimum (16GB recommended for Llama 3.1)
- Disk: 20GB+ free (models like Llama 3.1 ~5GB)

**Ports:**  
5433 (Postgres), 5678 (n8n), 8080 (SearXNG), 11434 (Ollama)

**OS:** Linux, macOS, or Windows (with WSL2 recommended)

---

## Project Structure

```plaintext
n8n-stack/
├── docker-compose.yml
├── .env
├── init-data.sh
├── config/
│   ├── settings.yml
├── data/
└── README.md
```

Volumes auto-managed by Docker: `db_storage`, `n8n_storage`, `ollama_storage`.

---

## Environment Configuration

Create a `.env` file with:

```env
POSTGRES_USER=postgres
POSTGRES_PASSWORD=your_secure_password_here
POSTGRES_DB=n8n
POSTGRES_NON_ROOT_USER=n8n_user
POSTGRES_NON_ROOT_PASSWORD=n8n_secure_pass
N8N_ENCRYPTION_KEY=your_n8n_encryption_key_here
```

> **Security Tip:** Never commit `.env` to Git.

---

### Init Script (Optional)
```bash
#!/bin/bash
set -e
if [ -n "$POSTGRES_NON_ROOT_USER" ]; then
  psql -v ON_ERROR_STOP=1 -U "$POSTGRES_USER" <<-EOSQL
    CREATE USER $POSTGRES_NON_ROOT_USER WITH PASSWORD '$POSTGRES_NON_ROOT_PASSWORD';
    GRANT ALL PRIVILEGES ON DATABASE $POSTGRES_DB TO $POSTGRES_NON_ROOT_USER;
EOSQL
else
  echo "No Environment variables given!"
fi
```

---

## Docker Compose Setup

Save as `docker-compose.yml`:

```yaml
version: "3.8"
volumes:
  db_storage:
  n8n_storage:
  ollama_storage:

services:
  postgres:
    image: postgres:16
    restart: always
    environment:
      - POSTGRES_USER=${POSTGRES_USER}
      - POSTGRES_PASSWORD=${POSTGRES_PASSWORD}
      - POSTGRES_DB=${POSTGRES_DB}
    ports:
      - "5433:5432"
    volumes:
      - db_storage:/var/lib/postgresql/data

  n8n:
    image: n8nio/n8n:latest
    restart: always
    environment:
      - DB_TYPE=postgresdb
      - DB_POSTGRESDB_HOST=postgres
      - DB_POSTGRESDB_PORT=5432
      - DB_POSTGRESDB_DATABASE=${POSTGRES_DB}
      - DB_POSTGRESDB_USER=${POSTGRES_NON_ROOT_USER}
      - DB_POSTGRESDB_PASSWORD=${POSTGRES_NON_ROOT_PASSWORD}
      - N8N_ENCRYPTION_KEY=${N8N_ENCRYPTION_KEY}
    ports:
      - "5678:5678"
    depends_on:
      - postgres

  searxng:
    image: searxng/searxng:latest
    restart: unless-stopped
    ports:
      - "8080:8080"

  ollama:
    image: ollama/ollama:latest
    restart: unless-stopped
    ports:
      - "11434:11434"
```

---

## Starting and Stopping

```bash
docker compose up -d
docker compose ps
docker compose logs -f
docker compose down
```

> First run: 5–10 mins (pulling images). Access n8n at [http://localhost:5678](http://localhost:5678)

---

## Post-Setup Configuration

### Downloading Ollama Model

```bash
docker exec -it ollama ollama pull llama3.1
docker exec -it ollama ollama list
docker exec -it ollama ollama run llama3.1 "Hello, world!"
```

---

## n8n Credentials Setup

### Postgres
- Auto-configured by env vars.

### SearXNG
1. Create → “SearXNG API”  
2. URL: `http://searxng:8080`  
3. Test: `/search?q=test&format=json` returns 200 OK.

### Ollama
1. Create → “Ollama API”  
2. URL: `http://ollama:11434`  
3. Model: `llama3.1`

---

## Connecting AI Agent in n8n

1. **Create Workflow** → Add “AI Agent” node  
2. Configure Agent:  
   - Model: “Local Ollama”  
   - Tool: “Local SearXNG”  
   - Memory: Simple memory (keep 5 items)  
3. Trigger: Manual or Chat Trigger  
4. Connect and Test.

---

## Sample Workflow

```json
{
  "name": "AI Search Agent",
  "nodes": [
    { "tool": "searxng-search", "model": "ollama" }
  ]
}
```

---

## Troubleshooting

| Issue | Cause | Fix |
|-------|--------|-----|
| n8n DB Error | Wrong .env | Restart n8n |
| SearXNG 403 | Format/Network | Bind `0.0.0.0`; restart |
| Ollama Connection | Model Missing | Pull model |
| AI Agent “No Tool Support” | Wrong Model | Use Llama3.1 |
| High RAM/CPU | Large Model | Switch to `phi3` |

---

## Security Notes

- Use HTTPS for production.
- Remove unused exposed ports.
- Use Docker secrets for sensitive data.
- Disable debug for SearXNG in prod.

---

## Upgrading and Maintenance

```bash
docker compose pull
docker compose up -d
docker compose down
tar czf backup.tar.gz env config/data/
```
