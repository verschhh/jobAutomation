# Job Opportunity Pipeline — complete setup checklist

This document is the handoff between the files already prepared in this repository and the few actions that require your accounts, your VPS, or your decisions.

## What is already prepared

- Docker Compose for PostgreSQL, n8n, and Caddy.
- HTTPS reverse proxy configuration.
- Environment-variable template.
- Configurable scoring model and location list.
- Google Sheets schema.
- Importable offline demo workflow with prepared sample jobs.
- Production workflow design, reliability rules, and interview script.
- `.gitignore` protection for `.env` and local secrets.

## 1. Decide the values you will use

Write these down privately; do not commit them to Git:

| Value | Example | Your value |
|---|---|---|
| VPS IPv4 | `203.0.113.10` | |
| n8n subdomain | `n8n.example.com` | |
| Google account | `you@example.com` | |
| Gmail label | `Job Alerts` | |
| Digest recipient | your email | |
| Time zone | `Europe/Paris` | |

Keep the scope deliberately small for the interview: Gmail, Sheets, Calendar, and one GPT credential.

## 2. Create the Google Sheet

1. Create a new Google Spreadsheet named `Job Opportunity Pipeline`.
2. Add these tabs:
   - `Jobs`
   - `Settings`
   - `Events`
   - `DemoData`
   - `Exceptions`
3. Copy the header rows from [sheets-schema.md](./sheets-schema.md).
4. Freeze row 1 and enable filters on `Jobs`.
5. In `Settings`, add at least:

   | key | value | enabled |
   |---|---|---|
   | `gmail_label` | `Job Alerts` | `TRUE` |
   | `digest_recipient` | your email | `TRUE` |
   | `demo_mode` | `TRUE` initially | `TRUE` |
   | `weekly_review_day` | `Friday` | `TRUE` |
   | `default_reminder_days` | `3` | `TRUE` |

6. Copy the contents of `scoring.json` into your notes or settings rows so you can explain the configuration during the interview.

## 3. Prepare the OVH VPS

Do this only if you want the public HTTPS demo. The offline workflow can be rehearsed locally first.

1. Create a small Ubuntu LTS VPS at OVH.
2. Add an SSH key during provisioning.
3. In OVH DNS, create an `A` record:

   ```text
   n8n.yourdomain.com  ->  VPS_PUBLIC_IP
   ```

4. Wait for DNS propagation and verify it:

   ```bash
   dig +short n8n.yourdomain.com
   ```

5. Allow inbound ports `22`, `80`, and `443`. Do not expose port `5678` publicly.
6. Connect to the server and install Docker Engine plus the Docker Compose plugin using Docker's official Ubuntu instructions.
7. Copy this repository to the VPS, for example with Git or `scp`.

## 4. Configure secrets and start the stack

On the VPS, inside the project directory:

```bash
cp .env.example .env
openssl rand -base64 36
nano .env
```

Set:

```dotenv
N8N_HOST=n8n.yourdomain.com
POSTGRES_USER=n8n
POSTGRES_PASSWORD=<unique-long-password>
POSTGRES_DB=n8n
N8N_ENCRYPTION_KEY=<random-value-at-least-32-characters>
GENERIC_TIMEZONE=Europe/Paris
```

Then validate and start:

```bash
chmod 600 .env
docker compose config
docker compose pull
docker compose up -d
docker compose ps
docker compose logs --tail=100 caddy n8n
```

Open `https://n8n.yourdomain.com`. Caddy should obtain the certificate automatically. If it does not, check DNS, ports 80/443, and `docker compose logs caddy`.

Create the n8n owner account. Store its password in your password manager.

## 5. Create Google OAuth credentials

In Google Cloud Console:

1. Create a project named `Job Opportunity Pipeline`.
2. Enable:
   - Gmail API
   - Google Sheets API
   - Google Calendar API
   - Google Drive API
3. Configure the OAuth consent screen.
4. Add yourself as a test user if the app remains in testing mode.
5. Create an OAuth client for a web application.
6. In n8n, create the Google OAuth2 credential first and copy the exact redirect URL n8n displays.
7. Add that redirect URL to the Google OAuth client.
8. Authenticate the credential with the Google account that owns the Sheet and receives job emails.

Use the narrowest scopes offered by the n8n credential. Gmail send access is only needed for the digest; Calendar write access is only needed for approved reminders.

## 6. Import and rehearse the offline workflow

1. In n8n, choose **Import from File**.
2. Select `job-opportunity-pipeline.json`.
3. Open the workflow and run it manually.
4. Confirm the output from `Normalize and score` contains:
   - `jobKey`
   - `locationBucket`
   - individual score components
   - `totalScore`
   - `recommendation`
   - `hardFlags`
5. Confirm `Build grouped digest` produces HTML grouped by location.

This imported workflow is intentionally self-contained for a reliable demo. It does not yet call Gmail, GPT, Sheets, or Calendar.

## 7. Build the production ingestion workflow in n8n

Create a second workflow named `Jobs — ingest and score`.

Recommended node order:

1. **Schedule Trigger** — every 5 minutes.
2. **Gmail: Get Many Messages** — filter by the `Job Alerts` label and a recent time window.
3. **Code: Expand messages** — output one item per message and preserve Gmail `messageId`.
4. **Gmail: Get Message** — retrieve the full body.
5. **Information Extractor / OpenAI** — return strict JSON with:

   ```json
   {
     "title": "",
     "company": "",
     "url": "",
     "location_raw": "",
     "work_model": "remote|hybrid|onsite|unknown",
     "salary_min": null,
     "salary_max": null,
     "salary_currency": "",
     "seniority": "junior|mid|senior|lead|unknown",
     "description": "",
     "deadline": null,
     "source": ""
   }
   ```

   Tell the model to use `null` for missing values and never invent a URL, salary, deadline, or visa requirement.

6. **Code: Normalize and score** — adapt the scoring code from the imported demo workflow to read the values in `scoring.json` and your `Settings` tab.
7. **Google Sheets: Lookup Rows** — look up `jobKey` in `Jobs`.
8. **IF: Existing job?**
   - Existing: update the row and `last_seen_at`.
   - New: append a row with `status = new`.
9. **Google Sheets: Append to Exceptions** for invalid GPT output or missing required fields.

Important: use `source_message_id` and `job_key` as idempotency keys. Re-running the workflow must update the existing row rather than append duplicates.

## 8. Build the digest workflow

Create `Jobs — daily digest`:

1. Schedule Trigger — once daily, for example 08:00 Europe/Paris.
2. Google Sheets — read `Jobs` where `status` is not `closed`.
3. Code — sort descending by `total_score`, then group by `location_bucket`.
4. Code — build HTML containing title, company, location, work model, score explanation, hard flags, and source URL.
5. Gmail — send to `digest_recipient`.

Include a clear disclaimer in the email: “This is a recommendation. No application has been submitted.”

## 9. Add the approval and Calendar workflow

Create `Jobs — approval and reminders`:

1. Add a webhook endpoint for approval links in the digest.
2. Validate the job key and use a signed or random approval token; do not trust arbitrary row numbers from the URL.
3. Update the matching `Jobs` row to `approved`.
4. Create a Google Calendar event only for an approved job or an explicit deadline.
5. Write the Calendar event ID to `Events`.
6. Use deterministic event IDs or an Events lookup so retries do not create duplicates.

Suggested event types:

- `application_deadline`
- `follow_up`
- `interview`
- `weekly_review`

Never add an “apply” action to the workflow. The user must submit applications manually.

## 10. Add failure handling

Create an n8n Error Trigger workflow that:

1. Receives failed executions.
2. Sends an email containing workflow name, node name, execution ID, and error message.
3. Writes a row to `Exceptions`.

For Gmail, Sheets, Calendar, and GPT nodes, configure a short timeout and one retry. Keep the original email body or a safe excerpt for auditability, but avoid storing unnecessary personal data.

## 11. Security and backup checklist

- Keep `.env` out of Git; `.gitignore` is already configured.
- Use a long unique PostgreSQL password and encryption key.
- Keep port 5678 private.
- Use a strong n8n owner password and MFA on Google and OVH accounts.
- Pin `N8N_VERSION` after rehearsal instead of using `latest` during the interview.
- Back up PostgreSQL and n8n data volumes.
- Test restoring a backup before relying on the system.
- Remove unused Google OAuth credentials and n8n users.
- Do not place API keys or OAuth tokens in Code nodes.

## 12. Final acceptance test

Run this test before the interview:

1. Send or load three sample job emails from different sources.
2. Confirm all three appear in Gmail under `Job Alerts`.
3. Run ingestion twice; confirm no duplicate rows.
4. Confirm one job has a hard flag for missing salary or location mismatch.
5. Confirm the digest is grouped by location and sorted by score.
6. Approve one job.
7. Confirm exactly one Calendar reminder is created.
8. Run the approval workflow again; confirm no duplicate Calendar event.
9. Temporarily break a test credential and confirm the Error Trigger notifies you.
10. Set `demo_mode = TRUE` and verify the offline fixtures still work.

## 13. Interview demo sequence

Use the script in [README.md](./README.md):

1. Architecture and human approval boundary.
2. Scoring configuration.
3. Offline fixture run.
4. Ranked and grouped output.
5. Sheets audit trail and deduplication.
6. Approval to Calendar reminder.
7. Error handling and Demo mode fallback.

If live Gmail or OAuth fails, switch to the imported fixture workflow immediately and explain that the same normalization/scoring stage is being demonstrated with prepared inputs.

## Definition of done

The project is ready when the final acceptance test passes, the VPS is reachable over HTTPS, OAuth credentials work, the Sheet is populated, approval creates one Calendar event, backups exist, and the complete demo runs in under 12 minutes.
