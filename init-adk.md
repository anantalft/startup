
cd ananta@ananta-ThinkPad-L14-Gen-3 ~/startup/leadflow-adk 
deactivate
rm -rf .venv
python3.11 -m venv .venv
source .venv/bin/activate

python --version

pip install -U google-adk fastapi uvicorn python-dotenv
pip show google-adk
adk --help


Step 2: create ur first agent
**(.venv) ananta@ananta-ThinkPad-L14-Gen-3 ~/startup/leadflow-adk $** adk create leadflow_agent
Help improve the ADK (CLI and Web UI) by allowing Google to collect pseudonymized usage data?

What is collected: Names of subcommands and flags (no user-provided values or arguments), execution metrics (duration, exit state), environment specs (OS, Python version), and aggregated Web UI feature interactions. No personally identifiable information (PII) is collected.

This is OFF by default. You can opt out at any time using the 'adk telemetry disable' command or Web UI user settings.

Enable telemetry? [Y/n]: n
Choose a model for the root agent:
1. gemini-3.5-flash
2. gemini-3.8-flash
3. Other models (fill later)
Choose model (1, 2, 3): 1
1. Google AI
2. Vertex AI
3. Login with Google
Choose a backend (1, 2, 3): 1

Don't have API Key? Create one in AI Studio: https://aistudio.google.com/apikey

Enter Google API key: AIzaSxxxxxxxxxxx - https://aistudio.google.com/api-keys?utm_source=chatgpt.com

Agent created in /home/ananta/startup/leadflow-adk/leadflow_agent:
- .env
- .gitignore
- __init__.py
- agent.py

⚠️  WARNING: Secrets (like GOOGLE_API_KEY) are stored in .env.

Next steps:
- Chat with your agent in the terminal: adk run leadflow_agent
- Or in the browser: adk web .
- Have your coding agent evaluate and deploy it: uvx google-agents-cli setup
  (other install options: https://adk.dev/get-started/agents-cli/)

**(.venv) ananta@ananta-ThinkPad-L14-Gen-3 ~/startup/leadflow-adk $** ls


