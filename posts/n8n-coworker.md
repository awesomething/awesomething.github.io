---
title: How to set up n8n in cowork
date: 2025-09-11
---

Step 1 — Get your n8n API key
Go to workflow.videobook.ai → Settings → API → Create API Key. Copy it immediately (shown once).
Step 2 — Connect n8n in Cowork
Claude app → Settings → Connectors → search "n8n" → Connect → paste the API key.
This gives you 3 tools: search_workflows, get_workflow_details, execute_workflow. That's it — the registry connector can't create workflows.
Step 3 — Store the key for bash
Create a .env file in your workspace folder with just the key:
YOUR_N8N_API_KEY_HERE
Upload it to your connected folder once. Bash reads it with $(cat .env | tr -d '[:space:]') whenever needed.
Step 4 — To CREATE workflows (the workaround)
Since the registry connector has no create_workflow, the method that actually works is:

I navigate Chrome to your n8n instance (already logged in from your browser session)
I use javascript_tool to POST the workflow JSON directly to /api/v1/workflows using the key