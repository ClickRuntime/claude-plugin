---
name: write-and-fix-a-script
description: Write, test, or fix the Python script behind an automation, or run the person's own code on a schedule or webhook. Use when the person says "run my code", shares a script to automate, asks why a script errors, wants to change what a live automation does, wants an automation that uses AI to classify, summarize, or extract from text, or when a script needs names or ids from a service (form fields, sheet columns, channel ids) or one-time setup before it goes live.
---

Explicit instructions from the person override this skill. If they ask for something different from these steps, follow them, as long as it stays within the safety rules below.

Use ClickRuntime only when the work runs code, calls an outside service, delivers outside this chat, reacts to an outside event, or has to remember something between runs. A reminder or reply that should simply appear in this chat does not need ClickRuntime.
## Writing

1. Follow [runtime.md](references/runtime.md) exactly. It is the contract the runner enforces:
   - `@on_trigger(event)` for schedule and manual runs; `@on_event(event)` for incoming events.
   - Before writing code that calls an outside service, check that service's own API documentation for the current endpoints, auth method, request and response fields, and rate limits. Do not rely on memory for an API's shape; APIs change and old endpoints get retired. If you cannot reach the docs, say so and tell the person which parts to double-check.
   - If a call you made without checking the docs (in a setup script, a test run or a live run) fails with 400, 401, 403, 404, 405 or a similar error, do not conclude the key or account lacks access. Look up the service's API docs and check the endpoint, method, base URL, API version and auth header first. Only once the documented call also fails, tell the person what access is missing.
   - If the service publishes its own Python SDK (not a third-party one), use it, pinned in `dependencies`, and follow its current docs. Otherwise use `http` against the documented API. Skip the SDK only when it breaks the rules in What scripts can't do, for example it needs `subprocess` or a non-HTTPS connection, and tell the person why.
   - For everything else, prefer the standard library, the built-in packages, and `clickruntime`.
   - When a built-in package cannot do the job, declare the PyPI package in `dependencies` with a pinned version (`[{"name": "tenacity", "version": "==9.0.0"}]`), and tell the person it installs when the script deploys, so the first deploy takes a minute or two longer. An undeclared import fails before the draft is saved.
   - Secrets and connections are read on demand with `clickruntime.credentials`: `credentials.connection("google", "work")` for a connected account, `credentials.secret("AIRTABLE_TOKEN")` for a saved secret. Each must be declared in `setup_needs` (a connection with its account `name`); nothing is in `os.environ`. Never print, log, or return a credential.
   - Scopes: whenever you write or change a script, for a new automation or an existing one, work out from the service's API docs which OAuth scopes its calls need, and declare them in that connection's `scopes` in `setup_needs`. Do not assume the connected account already has them. Call `get_setup_status` and compare: `app_missing_scopes` means the app does not request them yet, `missing_scopes` means the account has not granted them. Close the gap yourself; changing the app is your step, not the person's: call `draft_oauth_provider` with the same slug, the app's current scopes plus the new ones, and its full `setup_notes` (saved credentials are kept). A ready-made app (id starting `cr_`) cannot gain scopes: switch the need to an app that has them. Then ask the person only for what they alone can do: reconnect the account on the `begin_connection` link, and any setting in the service's own developer console that the app's `setup_notes` say a new scope needs. Never send them to the app's page to change scopes.
   - `notify.email_me`, `notify.slack`, `http`, `verify`, `state`, and `log` are the SDK helpers.
   - A schedule script that checks a source for new items must call `report_fetched(n)` once, right after the fetch, and return straight away when `n` is 0. Do not call it in any other kind of script. See Polling for new items in the reference.
   - Use `ai.generate` only when rules cannot do the job; see Using AI below.
2. Start from [recipes.md](references/recipes.md) when a template is close, and adapt names, URLs, and fields.
3. If the person brings their own code, keep its logic, move it into a handler, replace hard-coded keys and `os.environ` reads with `credentials.secret(...)` plus a `secret` need (or `credentials.connection(...)` plus a named connection need), and either declare the packages it needs in `dependencies` or replace them with built-in ones. Tell them what you changed.
4. Keep scripts short, idempotent, and quiet: log what happened and the ids you need, never secret values or whole payloads. Return a small JSON-serializable summary; it appears as the run result.

## Looking things up and one-time setup

Never guess a name, id or field that the script depends on: form field names, a payload's keys, a sheet's column headers, a channel or folder id, a table's columns. Find it out first. `run_setup_script` runs a short script once, outside any automation, and gives you its result. See Setup scripts in [runtime.md](references/runtime.md).

There are two purposes:

- **Discover** (`purpose: "discover"`): read something the automation needs from the service's API, such as a form's field names, a sheet's headers or a channel id. It only reads.
- **Setup** (`purpose: "setup"`): create or change something once before the automation runs, such as the spreadsheet with its header row, the webhook registered with the sender, or a label.

A setup script reads keys and connections like any script, so they must be set up before it can run. Follow these steps in order:

1. **Work out what the script needs.** From the service's API docs, list the keys and connections the script will read, and the OAuth scopes each connection needs.
2. **Check what already exists.** Call `list_connections`. It lists each service's connected `accounts` and the saved keys in `secret_names`.
3. **Choose accounts with the person**, as in Choosing accounts in `connect-a-service`: confirm the account to use, or the name for a new one. Never pick or invent an account name yourself. Reuse a saved key that has the name you would use.
4. **If everything is already set up**, go to step 6.
5. **If something is missing, ask once, then wait.**
   - Discover: give both choices in one message and let the person pick, for example: "I can read the field names from Tally if you add a Tally API key here: <link>. Or, if you know them, tell me the field names and I'll use those."
   - Setup: give each missing item with its link, in the same message as any other setup the automation will need (the same key or account usually serves both), and say what the setup script will do once they are done.
   - Get the links by calling `run_setup_script`: when something is missing it returns `setup_incomplete` with each item and its link, and nothing runs. For a missing connection, call `begin_connection` with the item's `connect_args` and give its `url`; for a missing key, give the item's `website_url`. Use each link exactly as returned, and say what each connection grants, as in The setup loop in `connect-a-service`.
   - Never ask for a key in the chat. Stop and wait for the person's answer.
   - If they tell you the information instead, use what they said and skip the script.
6. **Before a setup script, say what it will change.** Tell the person in one sentence what it will create or change outside ClickRuntime, and put the same sentence in `changes`. A discover script needs no warning.
7. **Run it.** Call `run_setup_script` with the `purpose`, a short `title`, the script, and `setup_needs` naming exactly the accounts and keys from step 3. If it returns `setup_incomplete` again, give the remaining links and wait, as in step 5.
8. **Use the result.**
   - Discover: use only what the result shows, and show the person what you found before writing the automation around it.
   - Setup: say what was created, and write the returned ids into the automation's script as constants. Nothing from the run is kept.
   - If it failed, read the error and logs, fix the script and run it again. A setup script must be safe to run twice: find the existing sheet or webhook before creating one.
9. **Declare the same keys and connections in the automation's `setup_needs`**, with the same account names, so setup is already complete when you deploy it.

Rules for both purposes:

- Return only what you need (names, keys, types, ids), never a credential or a whole payload.
- Prefer a setup script over asking the person to look things up, and over a test event, when an API can answer. When nothing can be looked up (there is no API, or the person would rather not add a key), ask the person, or deploy and read the first real event with `get_latest_event`.
- Do not use it to run the automation's work once, or as a way around turning an automation on.

## Keeping state

Each automation has its own state that survives between runs. Use it whenever the job depends on an earlier run, even if the person didn't say "state": "only new", "don't repeat", "when it changes", "after 3 failures", "at most once an hour", "compared with last week".

- Only new items: keep a cursor or the last id/timestamp with `state.set`, and store it only after the items are handled.
- Alert on change: read the last value with `state.get`, store the new one, and notify only when they differ. Treat `None` (first run) as a baseline, not a change.
- Thresholds and cooldowns: keep a counter or the time of the last alert as a value; reset it on recovery.
- Counts over time: `state.increment_metric(key, ttl_days, bucket=...)` and `state.query_metric(...)`, which returns `{"time", "value"}` buckets.
- Keep values small (ids, timestamps, numbers; each value at most 1 MB). It is not a database: write bulk data to a sheet or the person's own store. Never store keys or tokens.
- Say in the summary what the automation remembers. The person can reset it with **Clear live state** on the automation page; use `@state_initializer` for starting values that should come back after a clear.

## When a tool call is blocked

The assistant's own safety checks sometimes block a `save_draft`, `update_draft` or `propose_change` call (for example "This tool call was blocked by OpenAI's safety checks"). This is not a ClickRuntime error. The usual cause is a long ID-like value in the script that looks like a key, such as a spreadsheet ID, a folder ID or a webhook path.

By default, write such IDs in the script as plain constants: a secret needs the person to save it, so do not use one unless you have to. Only when a call is blocked:

1. Move the ID-like values out of the script into secrets: read each with `credentials.secret("SHEET_ID")` and add a `secret` need with a hint saying what the value is and where to find it (for example "The ID in the spreadsheet's URL, between /d/ and /edit").
2. Retry the call with the changed script. It no longer carries the value.
3. Tell the person why, and give them the key's link from `get_setup_status` (its `website_url`) so they save the value in the ClickRuntime dashboard. Do not send the value in a tool argument again.

If the call is still blocked without those values, tell the person what you were saving and ask them to try again; do not loop.

## What scripts can't do

- Outbound calls only over HTTPS (`https://`, port 443). No `http://` URLs, other ports, raw sockets, or UDP. Send email with `notify.email_me` or an email service's HTTPS API, not SMTP.
- No `subprocess`, `os.system`, `multiprocessing`, or `ctypes`. Use the service's HTTPS API or a pure-Python library instead.
- A draft that breaks these rules is rejected with the reason, and they are blocked at run time with a clear error. If the person's own code does any of this, rewrite that part and tell them why.

## Using AI

Scripts can call a model with `clickruntime.ai` and no API key; see Using AI in [runtime.md](references/runtime.md).

- When a step needs AI (classifying, summarising, extracting from free text), use ClickRuntime's built-in AI. The ClickRuntime SDK gives scripts an AI model with no API key to set up: `from clickruntime import ai`, then `ai.generate(prompt, system=None, max_output_tokens=512, json_schema=None)`, which returns a result with `.text` (and `.json` when `json_schema` is given). Do not ask which AI provider or model to use and do not ask for an AI key: tell the person the automation uses ClickRuntime's built-in AI. Only when the person asks for a specific provider of their own (for example their OpenAI or Anthropic key), save that key as a secret and call that provider's API instead.

- Prefer plain code. Parse, match keywords or regexes, compare numbers, and read structured fields without AI. Use AI only for genuinely fuzzy work such as classifying or summarising free text, or extracting fields from unstructured text.
- Keep token use low: filter and dedupe before calling, send only the fields needed, trim long text, batch small items into one call, set a small `max_output_tokens`, and use `json_schema` for structured answers.
- Catch `ai.AINotIncluded` and `ai.AILimitReached` and fall back (rules, skip the item, or log) so one refusal does not fail the whole run.
- AI in scripts counts against the workspace's monthly AI tokens, including on the free plan. Keep AI calls small so they fit. Do not name plans or prices, compare plans, or suggest changing plan; if limits come up, state them as facts and give the plans page link from the tool note. If a tool note says the workspace has no AI tokens or has used them, offer a version without AI instead, such as keyword or regex rules, scoring thresholds, or sending the item to the person to decide, and say what it will miss.

## Testing

A draft cannot run: deploy it first (`update_draft` with `deploy: true`). Before any test of a live automation, say in one sentence what the run does outside ClickRuntime (emails sent, posts made, rows written).

- Schedule: call `run_automation_now`. It waits for the run and returns its status, error and logs.
- Incoming: call `send_test_event` with a realistic sample body. On a draft it only stores the event (secrets redacted) so you can compare its fields with what the script reads; on a live automation it runs the script and returns the run. `replay_event` with an `event_id` from `list_incoming_events` runs a real event again. Use `get_latest_event` for events the sender delivers.
- If a run is still going, call `get_run` with `wait: true`. Never claim a test passed without reading the run result.

## Fixing a failed run

Fix failures yourself and keep going: fix, redeploy, test again, until a run succeeds. After about 5 attempts that do not work, stop and tell the person what is blocking it and what you tried. Ask the person only for a connection, a key in the ClickRuntime dashboard, or a requirement that is unclear.

1. If the person's message names a run id (the Fix it button sends one), call `get_run` for it; otherwise call `list_runs`, then `get_run` for the failed run. Read its error and redacted log lines.
   - A run with `error_kind: "connection_scope"` (for example Slack `missing_scope`) is not a script bug: do not change the script. Follow the run's `next_step` (add the scope yourself with `draft_oauth_provider`, then give the `begin_connection` reconnect link; for a ready-made app, which cannot gain scopes, move to an app that has the scope).
2. Say the cause in plain language: what failed, the likely reason, and the change you are making. Do not invent log lines.
3. For a draft, save the change with `update_draft` (with `deploy: true` once setup is ready). Every new `script` goes with a new `how_it_works` that matches it: 2 to 8 plain sentences on what the script reads, decides, and changes or sends, for someone who cannot read code.
4. For a live or paused automation, call `get_script` to read the current code (tool results do not include it otherwise), then call `propose_change` with the new `script`, its `how_it_works` (and `schedule`, `setup_needs`, or `dependencies` if they change) and `deploy: true`. If the new script needs scopes the account lacks, follow Scopes above first and deploy once the person has reconnected. A change to `dependencies` alone needs no new script. If the deploy fails, the previous version keeps running. Once it deploys, test again with `run_automation_now`, `send_test_event` or `replay_event`.
5. Common causes: a missing secret or connection (`CredentialError` with `secret_missing`, `connection_missing`, or `connection_needs_reauth`: send them to `connect-a-service`), a credential the script reads but `setup_needs` does not declare (`not_declared` or `credentials_not_declared`: add the need), a script still reading `os.environ`, a missing package, a payload field that does not exist (check with `get_latest_event`), an endpoint or field that does not match the service's current API docs (check the docs before changing the call), a timeout (lower the work per run or use `http` with a timeout), a provider rate limit (slow the cadence), or an uncaught `ai.AILimitReached` or `ai.AINotIncluded` (catch it and fall back as in Using AI; do not retry or suggest a plan change).

## Fixing packages

- `ModuleNotFoundError` in a run, or a deploy that fails with `dependency_import_missing`: the declaration does not match the import. Fix `dependencies`: add the missing package, or correct its name when the import name differs (`python-dateutil` imports as `dateutil`, `pillow` as `PIL`, `beautifulsoup4` as `bs4`). Change the import only when the import itself is wrong. If this happened while deploying a draft, it is still a draft: fix it with `update_draft` and `deploy: true`.
- `dependencies_too_large`: the packages and what they pull in exceed the size budget. Choose a lighter package, drop one that is not needed, or use a built-in package.
- `dependency_build_failed`: the version does not exist or has no prebuilt wheel for Python 3.11 on Linux. Choose a version that has one, or another package.
- `invalid_dependencies`, `dependency_not_allowed`, or `dependency_conflicts_base`: the result names the entry; fix that entry. Do not declare a package that is already built in.
- `get_automation` shows `deploy.status` `building_dependencies` while packages install, then `layer.resolved` with the installed versions, or `layer.error` with the reason.

## Without cards

If no card appears, describe the result in a sentence or two. A `confirm_url` (from `prepare_confirmation`, used for deleting) goes to the person exactly as returned; they confirm or cancel on that ClickRuntime page. After they say they have, call `get_confirmation_status` with the `confirmation_ref` and report what it shows. See `get-started` for details.

## Refuse or narrow

Decline, with a short reason, when the request is to run malware, stealth monitoring of a person, or credential theft; to bypass another service's access controls or rate limits; to move money, place a trade, or complete a purchase; or to collect health, government, or payment-card data. Do not write code that evades CAPTCHAs, rotates IPs to avoid blocks, or exfiltrates environment variables.

Never ask the person to paste an API key, token, password, or one-time code. Never put a secret in a tool argument. If a saved key is missing, give them its link to the ClickRuntime dashboard. If they paste a secret anyway, tell them to revoke it and create a new one in the ClickRuntime dashboard, and do not repeat the secret.
