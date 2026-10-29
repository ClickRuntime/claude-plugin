# Draft shape

Arguments for `save_draft`, `update_draft`, and `propose_change`. The workspace comes from the connection; there is no workspace argument.

```json
{
  "name": "Status page watch",
  "summary": "Every hour, check the status page and email me if it is not operational.",
  "pattern": "schedule",
  "schedule": {"kind": "hourly", "minute": 0, "timezone": "UTC"},
  "script": "from clickruntime import on_trigger ...",
  "how_it_works": {
    "steps": [
      "Opens the status page.",
      "Reads the overall status it shows.",
      "If it says anything other than operational, emails you the status and the page link."
    ],
    "notes": ["If the page does not load, it emails you that the check failed."]
  },
  "setup_needs": []
}
```

- `pattern`: `schedule`, `incoming`, or `remember_then_schedule`.
- `schedule`: required for `schedule` and `remember_then_schedule`; omit for `incoming`.
- `summary`: one or two plain sentences; it appears on the cards.
- `how_it_works`: required with every `script`. See below.
- `setup_needs`: every connection and secret the script reads. An empty list means nothing to set up.

## How it works

The cards show `how_it_works` to people who cannot read code, next to the services the server detects in the script. It explains how the automation works; the summary already says what it does.

- `steps`: 2 to 8 short plain sentences, in the order the script runs: what it reads, what it decides, and what it changes or sends. Name each service it uses (Stripe, Google Sheets, Slack, email).
- `notes`: up to 4 sentences for branches that matter: nothing to do, a service fails, a condition is not met.
- No code, variable names, field names, or jargon. At most 240 characters per sentence.
- It must match the script exactly. Send a new `how_it_works` with every new `script`; you can also send it alone to fix the wording. If the script contacts a service the steps never mention, the card warns the person.

## Schedule

| `kind` | Fields | Meaning |
|---|---|---|
| `every_minutes` | `every_minutes` (≥ 5 and divides 60: 5, 6, 10, 12, 15, 20, 30), `timezone` | Every N minutes, aligned to the hour |
| `hourly` | `minute` (0–59), `timezone` | Once an hour at that minute |
| `daily` | `time` (`HH:MM`, 24 h), `timezone` | Every day |
| `weekdays` | `time`, `timezone` | Monday to Friday |
| `weekly` | `weekday`, `time`, `timezone` | One day a week |
| `monthly` | `day_of_month` (1–28, or `"last"`), `time`, `timezone` | One day a month |
| `cron` | `expression`, `timezone` | Anything the others cannot say. Use it only when no other kind fits |

`timezone` is an IANA name. Ask the person when a local hour matters and the timezone is unknown. For `every_minutes` and `hourly`, `UTC` is fine.

```json
{"kind": "every_minutes", "every_minutes": 15, "timezone": "UTC"}
{"kind": "hourly", "minute": 30, "timezone": "UTC"}
{"kind": "daily", "time": "07:30", "timezone": "Europe/London"}
{"kind": "weekdays", "time": "09:00", "timezone": "America/New_York"}
{"kind": "weekly", "weekday": "monday", "time": "08:00", "timezone": "Asia/Kolkata"}
{"kind": "monthly", "day_of_month": 1, "time": "09:00", "timezone": "Europe/London"}
{"kind": "monthly", "day_of_month": "last", "time": "18:00", "timezone": "UTC"}
{"kind": "cron", "expression": "0 9 ? * TUE#2 *", "timezone": "Europe/London"}
```

`cron` takes a 5-field cron (`minute hour day-of-month month day-of-week`) or a 6-field AWS cron (adding `year`; Sunday is 1 and one day field must be `?`). It runs in `timezone`. AWS extras: `L` (last day of the month), `15W` (weekday nearest the 15th), `6L` (last Friday), `3#2` (second Tuesday). Runs must be at least 5 minutes apart, and a fixed day 29 to 31 is refused because it skips months: use `L`. The result's `schedule.description` and `next_runs` show how it was read; check them with the person.

Translating words:

| Person says | Schedule |
|---|---|
| every 5 minutes | `{"kind": "every_minutes", "every_minutes": 5}` |
| every minute | not supported; offer every 5 minutes |
| every 7 minutes | not supported (7 does not divide 60); offer 5 or 10 |
| every hour / hourly | `{"kind": "hourly", "minute": 0}` |
| every 2 hours | not a schedule kind; use `hourly` and skip odd hours in the script, or offer hourly |
| every morning | `{"kind": "daily", "time": "08:00"}` after confirming the hour |
| weekday mornings at 9 | `{"kind": "weekdays", "time": "09:00"}` |
| every Friday at 5pm | `{"kind": "weekly", "weekday": "friday", "time": "17:00"}` |
| on the 1st of every month | `{"kind": "monthly", "day_of_month": 1, "time": "09:00"}` after confirming the hour |
| at the end of every month | `{"kind": "monthly", "day_of_month": "last", "time": "18:00"}` |
| on the 30th or 31st | not supported (some months lack them); offer `"last"` |
| second Tuesday of the month | `{"kind": "cron", "expression": "0 9 ? * TUE#2 *"}` |
| last Friday of the month at 5pm | `{"kind": "cron", "expression": "0 17 ? * 6L *"}` (not `FRI#5`: most months have no fifth Friday) |
| every 15 minutes, 9 to 5 on weekdays | `{"kind": "cron", "expression": "0/15 9-16 ? * MON-FRI *"}` |
| twice a day at 9 and 6 | `{"kind": "cron", "expression": "0 9,18 * * ? *"}` |
| cron `*/15 * * * *` | `{"kind": "every_minutes", "every_minutes": 15}` |
| cron `0 9 * * 1-5` | `{"kind": "weekdays", "time": "09:00"}` |
| cron `0 9 1 * *` | `{"kind": "monthly", "day_of_month": 1, "time": "09:00"}` |
| any other cron | `{"kind": "cron", "expression": "<it>"}` |

Add `timezone` to each.

## Setup needs

Each item is `{kind, provider, name, hint}`. The script reads each one with `clickruntime.credentials`.

| `kind` | Required fields | Notes |
|---|---|---|
| `connection` | `provider`: `email`, the id of a ready-made app from `list_app_catalog` (starts with `cr_`), or the slug of a workspace OAuth app such as `slack` or `google` (the workspace's own app, added with `draft_oauth_provider`); `name`: the account (required except for `email`) | A named account, such as `work` or `personal`; read with `credentials.connection(provider, name)`. Reuse a name from `list_connections` or ask the person. Two accounts of one provider are two needs. `email` is always ready. |
| `secret` | `name` (UPPER_SNAKE), `hint` (one sentence: where to create it) | Typed by the person in the ClickRuntime dashboard; read with `credentials.secret(NAME)`. `provider` is optional and descriptive. |
| `notifications` | none | The workspace notifications Slack (ClickRuntime's own app), for scripts that call `notify.slack`. Prefer it when a script only posts messages. |

```json
[
  {"kind": "connection", "provider": "slack", "name": "work"},
  {"kind": "connection", "provider": "google", "name": "work"},
  {"kind": "connection", "provider": "google", "name": "personal"},
  {"kind": "notifications"},
  {"kind": "connection", "provider": "email"},
  {"kind": "secret", "provider": "airtable", "name": "AIRTABLE_TOKEN", "hint": "Create a personal access token at airtable.com/create/tokens with access to the one base."},
  {"kind": "secret", "provider": "stripe", "name": "STRIPE_WEBHOOK_SECRET", "hint": "In Stripe, open Developers > Webhooks, select the endpoint, and reveal the signing secret."}
]
```

Hints are sentences, not URLs alone, and never contain a value. The trigger link of an incoming automation is created with the draft; it is not a setup need you declare.

## Full examples

Incoming:

```json
{
  "name": "Typeform to sheet",
  "summary": "When the signup form is submitted, add a row to the Signups sheet.",
  "pattern": "incoming",
  "script": "...",
  "setup_needs": [
    {"kind": "connection", "provider": "google", "name": "work"},
    {"kind": "secret", "provider": "typeform", "name": "TYPEFORM_WEBHOOK_SECRET", "hint": "In Typeform, open Connect > Webhooks, edit the webhook, and set a secret."}
  ]
}
```

Remember then schedule:

```json
{
  "name": "Daily signup digest",
  "summary": "Collect sign-ups as they arrive and email me a list every evening.",
  "pattern": "remember_then_schedule",
  "schedule": {"kind": "daily", "time": "18:00", "timezone": "America/Chicago"},
  "script": "...",
  "setup_needs": []
}
```
