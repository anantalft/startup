# LeadFlow ADK - Version 1 Complete Command Reference

This document contains the step-by-step terminal commands used to configure the environment, generate the ADK agent, debug issues, execute the local server, and tag the Version 1 release.

---

## 1. Virtual Environment & Core Dependencies

```bash
# Navigate to the workspace
cd ~/startup/leadflow-adk

# Clean up any existing environment
deactivate 2>/dev/null || true
rm -rf .venv

# Initialize a fresh Python 3.11 virtual environment
python3.11 -m venv .venv
source .venv/bin/activate

# Verify Python version
python --version

# Install primary frameworks
pip install -U google-adk fastapi uvicorn python-dotenv
```

---

## 2. Scaffold the Agent with ADK CLI

```bash
# Verify ADK installation
pip show google-adk
adk --help

# Generate the agent scaffold
adk create leadflow_agent

# Selections made during CLI prompt:
# 1. Enable telemetry? n
# 2. Choose model: gemini-3.5-flash (switched to gemini-2.5-flash in code for stability)
# 3. Choose backend: 1 (Google AI)
# 4. Enter Google API key: [AIzaSy...]

# Check the generated structure
ls -la
```

---

## 3. Directory Layout Setup

```bash
# Create directories for frontend assets and local data store
mkdir -p static data

# Create and verify static/index.html with the landing page code
ls -la static/index.html
```

---

## 4. Agent Testing & Log Debugging

```bash
# Test the agent directly in terminal interactive mode
adk run leadflow_agent

# In a separate terminal, follow ADK execution logs
tail -F /tmp/agents_log/agent.latest.log

# Inspect the most recent error logs (used to identify the 503 error)
tail -n 40 /tmp/agents_log/agent.latest.log
```

---

## 5. Dependency Fixes & Local Server Execution

```bash
# Install dependency required by Pydantic models
pip install email-validator

# Launch FastAPI with auto-reloading enabled
uvicorn main:app --reload --port 8000
```

---

## 6. Verification of Saved Data

```bash
# Check the local JSON output containing lead details & AI analysis
cat ~/startup/leadflow-adk/data/inquiries.json

# Pretty-print JSON format
python3 -m json.tool ~/startup/leadflow-adk/data/inquiries.json
```

---

## 7. Dependency Export & Git Commit (Version 1)

```bash
# Export exact dependencies to requirements.txt
pip freeze > requirements.txt

# Initialize Git repository (if not already initialized)
git init

# Check working tree status
git status

# Stage version 1 project files
git add .gitignore requirements.txt main.py leadflow_agent/ static/

# Verify staged items
git status

# Commit Version 1
git commit -m "feat: complete v1 local pipeline (FastAPI + ADK Agent + JSON storage)"

# Tag release
git tag -a v1.0.0 -m "Version 1: Local landing page to ADK lead qualification and JSON persistence"
```
