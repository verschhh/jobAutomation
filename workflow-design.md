# n8n workflow design

The included `job-opportunity-pipeline.json` is a compact importable demo workflow. For production, split it into three workflows and call them with Execute Workflow:

1. **Ingest + score** (every 5 minutes): Gmail Trigger on the `Job Alerts` label → extract message text → GPT structured output → Code normalizer/scorer → Google Sheets lookup by `job_key` → IF new/changed → append or update `Jobs`.
2. **Digest** (daily): Google Sheets read `status != closed` → Code sort/group by `location_bucket` and score → Gmail send HTML digest. Include score explanation, hard flags, source URL, and an `Approve` link back to a webhook that only changes the row to `approved`.
3. **Calendar + review** (hourly + weekly): read approved jobs/events → Google Calendar create/update with deterministic `event_id` → send a weekly review reminder. Never apply automatically.

## Reliability rules

- Use `source_message_id` and `job_key` for idempotency; a rerun must update, not duplicate.
- Have GPT return JSON only, validate required fields, and route invalid output to an `Exceptions` sheet.
- Keep the original email body and source URL for auditability.
- Set a 30-second timeout and one retry for Gmail/Sheets/Calendar API calls.
- Put an Error Trigger workflow on failures; email the operator with execution URL and node name.
- Use a manual “Demo mode” switch in Settings to read `DemoData` instead of waiting on Gmail.
