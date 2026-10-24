# PhishSim — Internal Phishing-Simulation Training Tool

A Google Apps Script tool for running authorized internal phishing-awareness
exercises: it sends a simulated phishing email to a sample of employees,
tracks who opened it, clicked the link, or opened the attachment, catches
replies, and emails a weekly results report.

> **Authorization notice.** This tool is for IT Security teams running
> sanctioned awareness exercises with leadership/HR/Legal sign-off, as
> required by your organization's policy. Only ever target addresses you
> are explicitly authorized to test, and use results for coaching, not for
> disciplining individuals.

## How it maps to a typical Python phishing-sim stack

| Python piece | Apps Script equivalent |
|---|---|
| `config.py` | `CONFIG` object + `getConfig()` (Script Properties can override) |
| `db.py` (SQLite) | A Google Sheet ("PhishSim DB") with `Campaigns`, `SentEmails`, `Events` tabs |
| `sender.py` (smtplib) | `GmailApp.sendEmail()` |
| `tracker.py` (Flask) | `doGet(e)` on a deployed Web App, routed by an `?action=` query param |
| `reply_checker.py` (IMAP) | `GmailApp.search()` on plus-addressed replies |
| `weekly_report.py` | `sendWeeklyReport()` |
| `scheduler.py` (cron) | `ScriptApp` time-based triggers via `setupTriggers()` |

## What gets tracked

For each sent email, the tool records, by a per-recipient token:

- **Open** — a 1x1 tracking pixel loads
- **Click** — the "Verify My Account" link is opened
- **Attachment download** — the attachment link is opened
- **Reply** — a reply arrives at the plus-addressed `Reply-To`

Anyone who clicks or opens the attachment lands on an in-the-moment
education page explaining it was a simulation and giving phishing-spotting
tips, rather than a real credential page or payload.

## Setup

1. **Create the project.** New Apps Script project, paste `PhishSim.gs` in.
2. **Fill in `CONFIG`** at the top of the file (or set overrides via
   **File > Project properties > Script properties** — currently `FROM_ADDRESS`
   and `WEB_APP_URL` can be overridden that way):
   - `FROM_ADDRESS` — must be `Name <user@domain>`, and the bare address must
     be a verified "Send mail as" alias on the executing Gmail/Workspace account
   - `TARGET_USERS` — only employees in scope for the exercise, each with a
     unique `id`
   - `USERS_PER_CAMPAIGN` — how many targets get picked per run
   - `REPORT_RECIPIENTS` — who gets the weekly summary
   - `EMAIL_SUBJECT`, `TEMPLATE_NAME` — campaign content
3. **Run `initDb()` once** to create the backing "PhishSim DB" spreadsheet.
   Its ID is stored in Script Properties automatically.
4. **Deploy as a Web App**: **Deploy > New deployment > Web app**, execute
   as yourself, access as **Anyone**. Copy the `/exec` URL into
   `CONFIG.TRACKER_BASE_URL` (or the `WEB_APP_URL` script property).
5. **Run `setupTriggers()` once** to install the scheduled jobs (see below).
6. **Authorize the requested scopes** (Gmail send/search, Sheets, external
   URL fetch for Web App responses) on first run.

### Redeploying after edits

Editing the script does **not** update the live `/exec` URL. After any
change, go to **Deploy > Manage deployments**, edit the existing deployment,
choose **New version**, and deploy. Creating a brand-new deployment instead
changes the URL and breaks links already sent out.

## Scheduled jobs (installed by `setupTriggers()`)

| Function | Schedule | What it does |
|---|---|---|
| `sendCampaign` | Mondays, 9am | Picks `USERS_PER_CAMPAIGN` random targets and sends the simulation email |
| `checkReplies` | Every 15 minutes | Scans unread mail for plus-addressed replies and logs them |
| `sendWeeklyReport` | Fridays, 4pm | Emails the 7-day results table to `REPORT_RECIPIENTS` |

Re-running `setupTriggers()` deletes and recreates all triggers for this
project, so re-run it (not something else) if you change the schedule.

## Files

| File | Purpose |
|---|---|
| `PhishSim.gs` | Everything: config, sheet-backed storage, sending, tracking endpoints, reply checking, weekly report, triggers |

## Data model (PhishSim DB spreadsheet)

| Tab | Columns |
|---|---|
| `Campaigns` | `campaign_id`, `template_name`, `created_at` |
| `SentEmails` | `token`, `campaign_id`, `user_id`, `user_name`, `user_email`, `sent_at` |
| `Events` | `event_id`, `token`, `event_type`, `occurred_at`, `ip_address`, `user_agent` |

`event_type` is one of `open`, `click`, `attachment_download`, `reply`.
`ip_address`/`user_agent` are filled in client-side (see below), since
`doGet(e)` has no access to request headers.

## Notable implementation details

- **Tokens** are 32-character hex UUIDs, validated with a strict regex
  (`_isValidToken`) before any lookup, logging, or sheet write.
- **IP/user-agent capture**: `doGet` can only see the query string, so the
  education page itself calls back into `recordClientMeta()` from the
  visitor's browser, which backfills the `Events` row that `doGet` already
  wrote. The public-IP lookup (via ipify) is best-effort — if it fails or is
  blocked, the user agent is still recorded with an empty IP rather than
  nothing being recorded at all.
- **Reply correlation** uses Reply-To plus-addressing
  (`local+<token>@domain`), so it requires a mail domain that supports
  plus-addressing (Gmail/Workspace does).
- **Unknown or malformed tokens** are rejected early and never reach the
  sheet lookup or logs.
- **Sheet failures during event recording** are caught and logged (look for
  `"recordEvent failed"` / `"recordClientMeta failed"` in **Executions**)
  rather than silently treated as success.

## Troubleshooting

| Problem | Likely cause |
|---|---|
| `FROM_ADDRESS is not in "Name <user@domain>" format` | Fix the `CONFIG.FROM_ADDRESS` string |
| Gmail send fails with an alias error | The bare address in `FROM_ADDRESS` isn't a verified "Send mail as" alias on the executing account |
| Tracking pixel/click page shows "Not found." | Token missing from `SentEmails`, or malformed |
| Events not appearing in the sheet | Check **Executions** for `recordEvent failed` / `recordClientMeta failed` log lines |
| Replies not logged | Confirm the mail domain supports plus-addressing and that `checkReplies` trigger is active |
| Links from an old campaign stop working | The Web App was redeployed as a new deployment (new URL) instead of a new version of the existing one |
| Weekly report doesn't arrive | Check the Friday 4pm trigger exists in **Triggers**, and that `REPORT_RECIPIENTS` is set |

Logs for every function are in the Apps Script **Executions** view.
