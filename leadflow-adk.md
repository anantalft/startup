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



# LeadFlow ADK - Version 2 Implementation & Deployment Guide

This document contains the step-by-step terminal commands, environment configuration, code updates, Git checkpoint commands, and Cloud Run deployment instructions for Version 2 (Google Sheets Integration + Cloud Run Containerization).

---

## 1. Install Google Sheets Dependencies

Activate your virtual environment and install the required Google Auth and Sheets libraries:

```bash
# Navigate to project workspace
cd ~/startup/leadflow-adk

# Activate existing Python 3.11 virtual environment
source .venv/bin/activate

# Install gspread and Google authentication packages
pip install gspread google-auth

# Update requirements.txt to freeze new dependencies
pip freeze > requirements.txt
```

---

## 2. Google Sheets & Service Account Setup

1. **Create the Google Sheet:**
   - Go to [Google Sheets](https://sheets.new).
   - Spreadsheet Title: `SimpleFlow Leads`
   - Row 1 Headers:
     - `A1`: `Timestamp`
     - `B1`: `Name`
     - `C1`: `Email`
     - `D1`: `Business`
     - `E1`: `Category`
     - `F1`: `Urgency`
     - `G1`: `Scope`
     - `H1`: `Summary`
     - `I1`: `Suggested Reply`
     - `J1`: `Raw Message`

2. **Share Permissions:**
   - Locate the `"client_email"` inside your downloaded service account JSON file.
   - Click **Share** in the sheet and add this email with **Editor** permissions.

3. **Enable GCP APIs:**
   - Enable **Google Drive API** in your Google Cloud Console.
   - Enable **Google Sheets API** in your Google Cloud Console.

---

## 3. Persistent Local Environment Configuration

Append the service account path and sheet title to your local `.env` file:

```bash
cat << 'EOF' >> .env
GOOGLE_APPLICATION_CREDENTIALS="/home/ananta/startup/leadflow-adk/service-account.json"
GOOGLE_SHEET_NAME="SimpleFlow Leads"
EOF
```

---

## 4. Run & Test Google Sheets Sync Locally

Start your FastAPI server with live reloading:

```bash
uvicorn main:app --reload --port 8000
```

1. Open `http://localhost:8000` in your browser.
2. Submit a test inquiry via the form.
3. Verify that the terminal logs confirm the write:
   ```text
   ✓ Successfully appended row to Google Sheets.
   INFO:     127.0.0.1:xxxxx - "POST /api/v1/inquiries HTTP/1.1" 200 OK
   ```
4. Check the Google Sheet to confirm the row has been added.

---

## 5. Security & Git Commit (Version 2 Milestone)

Safeguard credentials and commit the Google Sheets integration:

```bash
# Ensure service-account.json and keys are ignored
cat << 'EOF' >> .gitignore

# Service account keys and local configs
service-account.json
*.json
!data/inquiries.json.example
EOF

# Check git status
git status

# Stage core code changes
git add main.py requirements.txt .gitignore

# Commit Version 2 Google Sheets integration
git commit -m "feat: add Google Sheets integration via Service Account and gspread"

# Tag milestone
git tag -a v2.0.0-alpha -m "Version 2 (pre-deploy): Verified Google Sheets sync and agent integration"

# Verify commit history
git log --oneline -n 3
```

---

## 6. Containerization (Next Step: Docker)

Create the `Dockerfile` for Cloud Run:

```bash
cat << 'EOF' > Dockerfile
FROM python:3.11-slim

ENV PYTHONUNBUFFERED=1 \
    PORT=8080

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

EXPOSE 8080

CMD exec uvicorn main:app --host 0.0.0.0 --port ${PORT}
EOF
```

Create the `.dockerignore` file to exclude unnecessary files from the build:

```bash
cat << 'EOF' > .dockerignore
.venv
venv
__pycache__
*.pyc
*.pyo
.git
.gitignore
*.env
service-account.json
data/inquiries.json
*.log
EOF
```

Optional: Build and run the image locally using Docker:

```bash
docker build -t leadflow-app:v2 .

docker run -p 8080:8080 \
  -e GOOGLE_API_KEY="YOUR_GEMINI_API_KEY" \
  -e GOOGLE_SHEET_NAME="SimpleFlow Leads" \
  -v /home/ananta/startup/leadflow-adk/service-account.json:/app/service-account.json:ro \
  -e GOOGLE_APPLICATION_CREDENTIALS="/app/service-account.json" \
  leadflow-app:v2
```

---

## 7. Cloud Run Deployment

Deploy directly from source using the Google Cloud SDK:

```bash
# 1. Log in and configure active GCP project
gcloud auth login
gcloud config set project YOUR_GCP_PROJECT_ID

# 2. Enable Cloud Run and Cloud Build APIs
gcloud services enable run.googleapis.com cloudbuild.googleapis.com

# 3. Deploy the service to Google Cloud Run
gcloud run deploy leadflow-service \
  --source . \
  --region us-central1 \
  --allow-unauthenticated \
  --service-account="YOUR_SERVICE_ACCOUNT_EMAIL" \
  --set-env-vars GOOGLE_API_KEY="YOUR_GEMINI_API_KEY",GOOGLE_SHEET_NAME="SimpleFlow Leads"
```

Once deployment completes, the terminal will provide the live HTTPS production URL.
