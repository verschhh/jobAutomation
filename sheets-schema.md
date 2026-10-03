# Google Sheets schema

Create a spreadsheet with these tabs. Freeze row 1 and enable filters.

## Jobs

`job_key`, `source`, `source_message_id`, `received_at`, `title`, `company`, `url`, `location_raw`, `location_bucket`, `work_model`, `salary_min`, `salary_max`, `salary_currency`, `seniority`, `description`, `technical_score`, `customer_facing_score`, `seniority_score`, `location_score`, `remote_hybrid_score`, `salary_score`, `relocation_visa_score`, `company_domain_score`, `total_score`, `recommendation`, `hard_flags`, `status`, `last_seen_at`, `notes`

`job_key` is the stable dedupe key: canonical URL when present, otherwise lowercase company + title + normalized location. `status` is one of `new`, `reviewed`, `approved`, `rejected`, `applied`, `interviewing`, `closed`. The workflow only creates reminders after the user changes status to `approved` or enters an explicit deadline.

## Settings

`key`, `value`, `enabled`, `notes`

Seed with `digest_recipient`, `gmail_label`, `digest_send_hour`, `weekly_review_day`, `default_reminder_days`, and the location/profile values from `scoring.json`. This keeps scoring configurable without editing the workflow.

## Events

`event_id`, `job_key`, `event_type`, `event_date`, `calendar_event_id`, `approved_by_user`, `created_at`, `status`

## DemoData

`scenario`, `from`, `subject`, `body`, `expected_location`, `expected_score_band`. Keep three prepared emails here so the interview demo works without live LinkedIn/Indeed delivery.
