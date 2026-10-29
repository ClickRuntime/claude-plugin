# ClickRuntime script reference

Scripts are Python 3.11 and run on AWS Lambda with a time limit per run and only the automation's own declared secrets and connections. Outbound connections are HTTPS only and scripts can't start processes (see What scripts can't do). The SDK is imported as `clickruntime`. A draft never runs; a live automation runs on its schedule, on each incoming event, or once when it is run now (`run_automation_now`) or an event is replayed.

## Handlers

| Decorator | Called for | Argument |
|---|---|---|
| `@on_trigger` | schedule runs and manual "run now" runs | `event` (`clickruntime.Event`), `event.type` is `"schedule"` or `"manual"` |
| `@on_event` | each incoming request to the trigger link | `event` (`clickruntime.Event`), `event.type` is `"incoming"` |
| `@state_initializer` | once, when the automation is first deployed | no argument |
| `@state_cleaner` | when state is cleared, just before the keys are deleted | no argument |

Every script must have the handler its automation needs, applied as a decorator imported from `clickruntime`. Without it every run fails with "No @on_trigger handler defined in script" (or `@on_event`):

- Schedule automation: `@on_trigger` is required.
- Incoming automation: `@on_event` is required. Add `@on_trigger` too if it should also work with "run now", which calls `@on_trigger`.
- `remember_then_schedule` automation: both are required. `@on_event` stores what arrived, and `@on_trigger` acts on it later.
- `@state_initializer` and `@state_cleaner` are optional.

A function only named `on_trigger` or `on_event` is not a handler; it must carry the decorator. Do not write a Lambda-style `handler(event, context)` or a `main()`. The handler's return value must be JSON-serializable; it is stored (redacted) as the run result.

Clearing state always deletes every key and metric, whether or not the script has a `@state_cleaner`, and even if it fails. The cleaner runs first, while the state is still readable, so use it only for side effects: undo things outside ClickRuntime that the state points to (for example, remove a webhook the script registered) or log a final value. It does not need to delete keys. A cleaner run does not count against the run allowance.

```python
from clickruntime import on_event, on_trigger

@on_event
def handle(event):
    event.json                      # parsed JSON body (None if the body is not JSON)
    event.headers                   # dict, lowercase keys
    event.header("Stripe-Signature")
    event.body                      # raw body string, exact bytes as text (alias: event.raw_body)
    event.event_id                  # platform delivery id, stable across retries and replays
    event.received_at               # ISO 8601 UTC
    event.verified                  # {"provider": ..., "ok": bool} or None
    event.is_verified               # True when platform verification passed

@on_trigger
def run(event):
    event.type                      # "schedule" or "manual"
    event.scheduled_time            # UTC time the run was due (schedule only)
```

`Event` is also a dict of the envelope, so `event["event"]` works too.

## Payloads

Incoming:

```json
{
  "type": "incoming",
  "received_at": "2026-01-01T09:00:00Z",
  "event": {"id": "evt_1", "type": "checkout.session.completed"},
  "headers": {"content-type": "application/json"},
  "raw_body": "{\"id\":\"evt_1\",...}",
  "event_id": "ev_01J...",
  "verified": {"provider": "stripe", "ok": true}
}
```

- `event` is the parsed JSON body. Form-encoded bodies may be parsed into an object; `raw_body` is always the original.
- Headers reach the script unredacted so it can verify signatures. They are masked anywhere they are stored or shown.
- `verified` is set when the automation has signature settings (`signature` on `save_draft`, `update_draft`, or `propose_change`, or the Trigger link section in the ClickRuntime dashboard): provider `stripe`, `github`, `shopify`, `typeform`, `standard_webhooks`, or `hmac_sha256` (with an optional header, encoding, and prefix). With `require` on, failed deliveries never reach the script. Otherwise it is `null`.
- Use `event.event_id` for idempotency. The platform also deduplicates on the sender's id header when present.
- Use only fields the person or a stored event (`get_latest_event`) has shown. Do not invent fields.

Schedule: `{"type": "schedule", "scheduled_time": "2026-01-01T09:00:00Z"}`. Manual: `{"type": "manual"}`, optionally with `"requested_by": "assistant"` (plus `"requested_via"`, the display name of the assistant that asked) or `"website"`.

## Secrets and connections

Nothing secret is in the environment. The script asks for each credential when it needs it, with `clickruntime.credentials`:

```python
from google.oauth2.credentials import Credentials
from googleapiclient.discovery import build
from clickruntime import credentials, http

work_token = credentials.connection("google", "work")          # str: access token of the google account named work
personal_token = credentials.connection("google", "personal")  # str: a second account of the same provider
key = credentials.secret("AIRTABLE_TOKEN")                     # str: a saved secret

sheets = build("sheets", "v4", credentials=Credentials(work_token), cache_discovery=False)
http.get("https://api.example.com/me", headers={"Authorization": f"Bearer {work_token}"})
```

Both functions return a plain `str`: the value itself, not an object or dict. There is no `.access_token`, `.token`, or `["access_token"]` on it; use the string directly.

- `credentials.connection(provider, name)` returns the OAuth access token of the connected account `name` of `provider`: the id of a ClickRuntime ready-made app from `list_app_catalog` (it starts with `cr_`), or the slug of an OAuth app the workspace added with its own credentials. Slack is always the workspace's own app, under the slug `slack`. For Google Sheets, use the ready-made `cr_google_sheets_picked` when its limits fit: it opens only the spreadsheets the person picked (their IDs are in the account's `files` in `list_connections`) and the ones the script creates with `spreadsheets.create`; a 404 or 403 on another sheet means it was not picked, so give the person the account's `choose_files_url`. Use the workspace's own Google app, under `google`, for anything else in Google. To only post a message to Slack, use `notify.slack` instead, which needs no app of the workspace's own. Every connection is a named account chosen by the person, such as `work` or `personal`; there is no default account.
- The script never gets the refresh token or the OAuth app's client id and secret; the platform refreshes the access token. Give client libraries only the access token, and do not set them up to refresh by themselves. If a library will not work without a refresh token, call the provider's HTTPS API with `Authorization: Bearer <token>` instead. In a long run, call `credentials.connection(...)` again rather than keeping the first value.
- `credentials.secret(NAME)` returns the value of a saved secret (UPPER_SNAKE name).
- A credential with several parts is one secret per part: a username and password are `credentials.secret("ACME_USERNAME")` and `credentials.secret("ACME_PASSWORD")`, each declared in `setup_needs`. A structured key, such as a Google service-account JSON file, is one secret that the script parses with `json.loads(credentials.secret("GOOGLE_SERVICE_ACCOUNT_JSON"))`.
- Each one must be declared in `setup_needs`: `{"kind": "connection", "provider": "google", "name": "work"}` or `{"kind": "secret", "name": "AIRTABLE_TOKEN", "hint": "..."}`. Two accounts of one provider are two needs. A credential that is not declared is refused. Saving a draft rejects literal calls that are not declared (`credentials_not_declared`, with `missing`); a call with a computed name is checked when the run asks for it.
- Values are available only while the run is running. Call them inside the handler (or a function it calls), not at import time.
- Values are cached for the rest of the run; an access token is fetched again shortly before it expires, and the platform refreshes it when needed.
- Every value is registered for redaction as soon as it is read. Do not `print`, `log`, or `return` a credential; saving a draft rejects that.
- Errors raise `credentials.CredentialError`; its message is safe to show and `.code` says why: `not_declared`, `connection_missing`, `connection_needs_reauth`, `secret_missing`, `run_not_running`, or `no_run` (outside a run). A bad provider, account name, or secret name raises `ValueError`.
- `email` needs no credential: use `notify.email_me`.
- `notify.slack` posts through the workspace notifications Slack, which is separate from the Slack accounts above and is never given to the script; declare `{"kind": "notifications"}` when the script calls it.

## SDK

```python
from clickruntime import on_trigger, on_event, state_initializer, state_cleaner
from clickruntime import state, log, notify, http, verify, redact, credentials, ai, Event, report_fetched
```

- `report_fetched(count)`: record how many new items a polling run found. Call it once per run, with a whole number of 0 or more. See Polling for new items.

- `notify.email_me(subject, text)`: plain-text email to the automation owner's verified address. No key. Limits: 10 per run, 200 per day per workspace, subject up to 200 characters, text up to 100 KB.
- `notify.slack(text, channel=None)`: post through the workspace notifications Slack (connected once per workspace in the ClickRuntime dashboard, separate from Slack accounts the script reads). Always pass `channel` (for example `"#sales"`); a default channel exists only if one was captured when it was connected, and the call fails with `channel_required` otherwise. Needs `{"kind": "notifications"}` in `setup_needs`. To post as a connected Slack account instead, use `slack_sdk.WebClient(token=credentials.connection("slack", "<name>"))`.
- Both raise `notify.NotifyError` on failure (not connected, limit reached, outside a run).
- `ai.generate(prompt, system=None, max_output_tokens=512, json_schema=None)`: platform AI, no key needed. `prompt` is a str or a list of str / `ai.Text(...)` parts (text only). Returns `AIResult` with `.text`, `.json` (parsed when `json_schema` is given), `.usage` and `.remaining` (`input_tokens`, `output_tokens`). Calls count against the workspace's monthly AI tokens, including on the free plan. The prompt's input tokens are counted before the call and it is refused when they would pass the month's input limit. Limits per call: about 32,000 input tokens, `max_output_tokens` up to 2,048; 50 calls per run. Tokens from runs that later fail still count. Raises `ai.AINotIncluded` (workspace has no AI tokens), `ai.AILimitReached` (period's tokens or the run's calls used up), `ai.AIRateLimited`, `ai.AIInputTooLarge`, `ai.AIUnavailable`; all subclass `ai.AIError`. See Using AI.
- `http.get/post/put/patch/delete/head/request(...)`: `requests` with a 20 s default timeout. Retries connection errors and 429/502/503/504 for GET, HEAD, OPTIONS, PUT, and DELETE, honoring `Retry-After`. POST and PATCH are not retried. `http.session(timeout=20, retries=3, backoff=0.5)` gives a reusable session. Responses are `requests.Response`.
- `verify.stripe(raw_body, header, secret, tolerance=300)`, `verify.github(raw_body, header, secret)`, `verify.shopify(raw_body, header, secret)`, `verify.typeform(raw_body, header, secret)`, `verify.standard_webhooks(raw_body, headers, secret, tolerance=300)`, `verify.generic_hmac_sha256(raw_body, signature, secret, encoding="hex", prefix="")` (`encoding` is `"hex"` or `"base64"`). All return `bool` and compare in constant time. Pass `event.body`, never re-serialized JSON.
- `state.get(key)` (None if missing), `state.set(key, value)` (JSON-serializable), `state.delete(key)`, `state.clear_all()`.
- `state.increment_metric(key, ttl_days, bucket="day", amount=1, initial_value=0)`, `state.decrement_metric(...)` (same arguments), `state.query_metric(key, bucket="day", from_time=None, to_time=None)`, `state.clear_metric(key)`. Buckets: `minute`, `hour`, `day`, `week`, `month`.
- `log.debug/info/warning(message, extra=None)`, `log.error(message, extra=None, exc_info=False)`. `extra` is a dict. Logs are shown to the person.
- `credentials.connection(provider, name)`, `credentials.secret(NAME)`: see Secrets and connections.
- `redact.redact(value)`: mask secret values in a str, dict, or list.

Redaction: the runner masks every credential the script has read (and its base64, URL-safe base64, and URL-encoded forms) and well-known token shapes in logs, results, errors, and stdout. Still, never log, return, or store in state a token or key. Log that a call succeeded and the ids you need.

## Using AI

Use `ai.generate` only for work rules cannot do: classifying or summarising free text, or pulling fields out of unstructured text. Use plain code for anything with a fixed shape (JSON, CSV, dates, prices, keyword or regex matches, thresholds).

Spend few tokens:
- Filter first: send only new or changed items, drop items rules can already decide, dedupe.
- Trim each item: strip HTML, signatures and quoted replies; keep only the fields needed; cut long text (for example `body[:2000]`).
- Batch small items into one call with numbered lines, and ask for one label per line or a JSON list.
- Set `max_output_tokens` to what the answer needs (a label needs about 20) and use `json_schema` for structured answers.
- Never send whole pages, attachments, logs or secrets.

Handle refusals so the run still does its other work:

```python
from clickruntime import ai, log

try:
    label = ai.generate(prompt, max_output_tokens=20).text.strip()
except (ai.AINotIncluded, ai.AILimitReached) as e:
    log.warning("AI unavailable, using keyword rules", {"code": e.code})
    label = keyword_label(text)
```

## Polling for new items

A schedule script that checks a source for new items (new rows, emails, orders, issues, feed entries) is a polling script. It must call `report_fetched(n)` exactly once, right after the fetch, with the number of new items. When `n` is 0 it must return straight after the call, with no other work: no notification, no other API calls, no state writes.

```python
from clickruntime import on_trigger, report_fetched, state, http

@on_trigger
def run(event):
    since = state.get("cursor")
    items = http.get("https://api.example.com/items", params={"since": since}).json()["items"]
    report_fetched(len(items))
    if not items:
        return {"summary": "Nothing new"}
    for item in items:
        ...  # act on each new item
    state.set("cursor", items[-1]["updated_at"])
    return {"summary": f"{len(items)} new"}
```

- Count only items that are new to this run, after removing ones already handled.
- Do not call it in `@on_event` handlers, or in schedule scripts that do the same work every time (a daily report, a status check, a reminder).
- A second call in the same run raises an error and fails the run.
- A scheduled check that reports 0, sends no notification, and finishes within 10 seconds is not counted toward the workspace's monthly runs. It still shows in the run history.

## Schedules

The shortest cadence is every 5 minutes. Timezones are IANA names such as `America/New_York`. `scheduled_time` is always UTC; convert with `zoneinfo` when the script needs local time.

## Available packages

Standard library, plus: requests, httpx, pytz, python-dateutil (`dateutil`), beautifulsoup4 (`bs4`), lxml, feedparser, openpyxl, PyYAML (`yaml`), Jinja2 (`jinja2`), google-api-python-client (`googleapiclient`), google-auth (`google.auth`, `google.oauth2`), slack_sdk, stripe, twilio, Markdown (`markdown`), and `clickruntime`. Prefer these. A draft that imports anything else without declaring it is rejected with a message naming the import.

## Extra packages

When the standard library and the packages above cannot do the job, declare the PyPI package in `dependencies` on `save_draft`, `update_draft`, or `propose_change`:

```json
"dependencies": [{"name": "tenacity", "version": "==9.0.0"}, {"name": "humanize", "version": "==4.11.0"}]
```

- Pin an exact version (`==`). Leave `dependencies` out to keep the current list; `[]` removes them all.
- At most 15 packages. Plain PyPI names and version specifiers only: no URLs, paths, `git+`, extras, environment markers, or pip options.
- Packages install as prebuilt wheels for Python 3.11 on Linux x86_64 (manylinux). A package with no such wheel fails with `dependency_build_failed`; pick another version or a lighter package.
- The installed packages, with what they pull in, must fit in 95 MB unzipped, or the deploy fails with `dependencies_too_large` and the measured size.
- Do not declare a package that is already available above. Declaring it with no version (or its installed version) is ignored with a note; another version gives `dependency_conflicts_base`. `clickruntime`, `boto3`, `botocore`, `awslambdaric`, `pip`, `setuptools`, and `wheel` give `dependency_not_allowed`.
- The import name can differ from the package name: `python-dateutil` imports as `dateutil`, `beautifulsoup4` as `bs4`, `pyyaml` as `yaml`, `pillow` as `PIL`, `scikit-learn` as `sklearn`, `python-slugify` as `slugify`, `attrs` as `attr`, `protobuf` as `google.protobuf`. The deploy checks every import against what the packages actually installed and fails with `dependency_import_missing` naming the import when none provides it.
- Packages install when the script deploys (turning on, or applying a change), so the first deploy of a new package set takes a minute or two longer. `get_automation` shows `deploy.status` `building_dependencies` meanwhile, then `layer.resolved` with the installed versions. An identical package set is reused without reinstalling.
- Packages known to be malicious (an OSV `MAL-` advisory, or our own deny list, which includes typosquats such as `colourama`) are refused with `blocked_package`; check the spelling of the name.
- A run that fails with `ModuleNotFoundError` means a package is missing or misnamed: fix `dependencies`, not the import, unless the import is wrong.

## What scripts can't do

- Outbound calls only over HTTPS (`https://`, port 443) to public addresses. No `http://` URLs, other ports, raw sockets, UDP, or listening sockets. Email goes through `notify.email_me` or an email service's HTTPS API, not SMTP.
- No `subprocess`, `os.system`, `os.popen`, `os.exec*`, `os.spawn*`, `os.fork`, `multiprocessing`, or `ctypes`. Use the service's HTTPS API or a pure-Python library instead.
- `smtplib`, `ftplib`, `imaplib`, `poplib`, `telnetlib`, and `socketserver` can't be imported.

A draft that breaks these rules is rejected with the reason, for example "Starting processes is not allowed." Anything the check can't see is caught at run time with an error such as "Outbound connections are allowed only over HTTPS (port 443). Port 25 was blocked." Every outbound connection is recorded. Repeated blocked attempts can pause the workspace.

## Setup scripts

`run_setup_script` runs a script once, outside any automation, and returns its result to you. It is not saved as an automation, has no schedule or trigger link, and nothing is left running afterwards. Use it to:

- Look something up that the automation needs and an API can tell you: a form's field names, a spreadsheet's column headers, a Slack channel id, an Airtable base's tables. Use `purpose: "discover"`. A discovery script only reads.
- Do one-time setup before an automation goes live: create the spreadsheet with its header row, register a webhook with the sender, create a label or folder. Use `purpose: "setup"` and say in `changes` what it creates or changes outside ClickRuntime.

The script is an ordinary script with an `@on_trigger` handler (`event.type` is `"manual"`). It follows every rule on this page: declare its credentials in `setup_needs` and its packages in `dependencies`, HTTPS only. It has no `state`. Return what you need as a small JSON-serializable value: names, ids, types. Never return or log whole payloads or any credential; the result is redacted like every run.

```python
from clickruntime import on_trigger, credentials, http

@on_trigger
def run(event):
    key = credentials.secret("ACME_API_KEY")
    form = http.get("https://api.acme.example/v1/forms/FORM_ID",
                    headers={"Authorization": f"Bearer {key}"}).json()
    return [{"key": f["id"], "label": f["label"], "type": f["type"]} for f in form["fields"]]
```

- Its keys and connections must be set up before it runs. First check `list_connections` and choose the accounts with the person. If something is missing, the call returns `setup_incomplete` with each missing item and its link, and nothing runs. Give the person those links and wait until they say they are done (for discover, also offer that they can tell you the information instead), then call again.
- Declare the same keys and connections, with the same account names, in the automation that uses the result.
- Setup scripts must be safe to run twice: check whether the sheet, webhook or label already exists before creating it, and return its id either way.
- Registering a webhook: when the automation already uses an API key or connection for the sending service (for discovery, other setup, or because the person offered one), and the sender's API can create webhooks, register it with a setup script pointed at the trigger link instead of asking the person to do it in that service. Do not add a key only for this. If it fails, first check the call against the service's API docs (see the next point); give the person the manual steps only if the documented call also fails.
- An outside API call that fails with 400, 401, 403, 404, 405 or a similar error is often the wrong endpoint, method, base URL, API version or auth header, not missing access. If you wrote the call without checking the service's API docs, look them up now and compare before concluding the key, plan or account lacks access, and before asking the person to do anything. Say what you found.
- Put any id the automation needs later (a new spreadsheet id, a webhook id) into the automation's script as a constant. Nothing from a setup run is kept for later runs.
- Each setup run counts as one run and shows on the Setup runs page in the ClickRuntime dashboard with its code and result.

## Run outcomes

- `success`: the handler returned; the value is the run result.
- `failed`: the script raised. The run shows the error, traceback, and logs, redacted.
- `error`: a platform problem, not the script's fault.

A deploy that fails because of the packages (`dependency_build_failed`, `dependencies_too_large`, `dependency_import_missing`) means the new version does not run. After a change, runs keep using the last version that deployed. When turning on a draft fails this way, nothing has ever deployed, so the automation goes back to a draft with its schedule and trigger link stopped. `get_automation` shows the code and message in `deploy` and `layer.error`; fix `dependencies` (or the import), then deploy again (`propose_change` or, for a draft, `update_draft`, with `deploy: true`).

Logs are capped at 4 MB per run, oldest dropped first.
