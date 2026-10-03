# AI Job Opportunity Pipeline

Starter kit for a Salesforce Solution Engineer interview: Docker deployment files, an offline fixture workflow, scoring configuration, and a production implementation checklist.

The included workflow scores two prepared jobs and generates digest HTML. Gmail ingestion, GPT extraction, persistent deduplication, Sheets writes, email delivery, approvals, and Calendar integration are not implemented yet. The scoring demo embeds simplified rules and does not load `scoring.json`. JSON and Compose validation do not verify an n8n import or live integrations.

For the complete, ordered handoff checklist, start with [SETUP-CHECKLIST.md](./SETUP-CHECKLIST.md).

## Deploy on an OVH Linux VPS

1. Point `n8n.yourdomain.com` (or another subdomain) to the VPS IPv4 address with an OVH A record. Allow inbound TCP 80/443 and SSH; do not expose 5678.
2. Install Docker Engine and Compose plugin. Copy this directory to the VPS.
3. Run `cp .env.example .env`; set a unique hostname, long random database password, and a 32+ character encryption key. Keep `.env` out of Git and backups that are not encrypted.
4. Run `docker compose config` to validate, then `docker compose up -d`.
5. Open `https://n8n.yourdomain.com`, create the owner account, and configure credentials.

Caddy obtains and renews the certificate automatically once DNS and ports are correct. Back up the `postgres_data` and `n8n_data` volumes, test restore before the interview, and pin `N8N_VERSION` after a successful rehearsal instead of using `latest` during the actual demo.

## Google OAuth

In Google Cloud Console create a project, enable Gmail API, Sheets API, Calendar API, and Google Drive API. Configure the OAuth consent screen with the minimum scopes and add the interview account as a test user. In n8n, create Google OAuth2 credentials and copy n8n’s exact redirect URL into an authorized redirect URI. Grant Gmail read/send, Sheets read/write, Calendar read/write, and Drive file access only as needed. Re-authenticate after changing scopes.

Import `job-opportunity-pipeline.json` for an offline rehearsal. Replace the fixture Code node with Gmail Trigger + Gmail Get Many for production, add GPT structured output, Sheets lookup/upsert, Gmail Send, approval webhook, and Calendar nodes as described in `workflow-design.md`.

## 10–12 minute demo script

Steps involving Sheets, approvals, Calendar, or failure handling require completing the production workflow checklist first.

1. **0:00–1:00** — Show the architecture: Gmail → AI normalization → scoring → Sheets → human approval → digest/Calendar.
2. **1:00–3:00** — Open `scoring.json` and explain weighted, configurable criteria and hard flags.
3. **3:00–6:00** — Run the imported workflow with the prepared fixtures; show two roles grouped by location and ranked score.
4. **6:00–8:00** — Open the Jobs sheet and point out stable `job_key`, raw source traceability, score breakdown, and status.
5. **8:00–10:00** — Change one role to `approved`, demonstrate the approval gate, and create a follow-up Calendar reminder.
6. **10:00–12:00** — Show failure handling/idempotency, then explain that Demo mode is the fallback if Gmail or OAuth is unavailable.

Never claim that the demo automatically applies. The safe boundary is: recommend, ask for approval, create a reminder, and leave the application action to the user.
