# Recipes

Starting points for common automations. Adapt names, URLs, fields, schedule and timezone to what the person asked for; do not copy placeholders such as `SHEET_ID` into a draft without replacing them. Each recipe lists the `save_draft` arguments (`pattern`, `schedule`, `setup_needs`) and a complete script. All scripts use only the standard library, the available packages, and `clickruntime` (see [runtime.md](runtime.md)). Secret names and hints follow [providers.md](../../connect-a-service/references/providers.md); sender setup from [senders.md](../../run-when-something-happens/references/senders.md).

Credentials are read inside the handler with `clickruntime.credentials`: `credentials.connection(provider, name)` for a connected account and `credentials.secret(NAME)` for a saved secret, each declared in `setup_needs`. Account names such as `work` are examples: ask the person which account to use and reuse a name `list_connections` already shows. The Slack recipes use `notify.slack(text, channel=...)`, which posts through the workspace's notifications Slack and needs `{"kind": "notifications"}` (never a named account). To post as the workspace's own Slack app instead (added under the slug `slack`, see connect-a-service), declare `{"kind": "connection", "provider": "slack", "name": "..."}` and use `slack_sdk.WebClient(token=credentials.connection("slack", name))`.

## 1. Page changed, email me

Prompt: "Every hour, check https://example.com/careers and email me if it changes."

- pattern: `schedule`
- schedule: `{"kind": "hourly", "minute": 0, "timezone": "UTC"}`
- setup_needs: `[]`

```python
import hashlib
from bs4 import BeautifulSoup
from clickruntime import on_trigger, state, log, notify, http

URL = "https://example.com/careers"

@on_trigger
def run(event):
    resp = http.get(URL)
    resp.raise_for_status()
    text = BeautifulSoup(resp.text, "html.parser").get_text(" ", strip=True)
    digest = hashlib.sha256(text.encode("utf-8")).hexdigest()
    previous = state.get("digest")
    state.set("digest", digest)
    if previous is None:
        log.info("Saved first snapshot")
        return {"changed": False, "first_run": True}
    if digest != previous:
        notify.email_me("Page changed", f"{URL} changed since the last check.")
        return {"changed": True}
    return {"changed": False}
```

## 2. Status page not operational, email me

Prompt: "Every 15 minutes, check our status page and email me if it doesn't say All systems operational."

- pattern: `schedule`
- schedule: `{"kind": "every_minutes", "every_minutes": 15, "timezone": "UTC"}`
- setup_needs: `[]`

```python
from clickruntime import on_trigger, state, notify, http

URL = "https://status.example.com"
PHRASE = "All systems operational"

@on_trigger
def run(event):
    ok = PHRASE.lower() in http.get(URL).text.lower()
    was_ok = state.get("ok")
    state.set("ok", ok)
    if not ok and was_ok is not False:
        notify.email_me("Status page reports a problem", f"{URL} no longer says '{PHRASE}'.")
    if ok and was_ok is False:
        notify.email_me("Status page recovered", f"{URL} says '{PHRASE}' again.")
    return {"operational": ok}
```

## 3. Price drop watch

Prompt: "Check this product page every morning and email me if the price drops below $50."

- pattern: `schedule`
- schedule: `{"kind": "daily", "time": "08:00", "timezone": "America/New_York"}`
- setup_needs: `[]`

```python
import re
from bs4 import BeautifulSoup
from clickruntime import on_trigger, state, log, notify, http

URL = "https://shop.example.com/product/123"
SELECTOR = ".price"
TARGET = 50.0

@on_trigger
def run(event):
    soup = BeautifulSoup(http.get(URL).text, "html.parser")
    node = soup.select_one(SELECTOR)
    if node is None:
        raise ValueError(f"No element matches {SELECTOR}; the page layout may have changed")
    price = float(re.sub(r"[^0-9.]", "", node.get_text()))
    log.info("Price checked", extra={"price": price})
    last_alert = state.get("last_alert_price")
    if price < TARGET and last_alert != price:
        notify.email_me("Price dropped", f"Now ${price:.2f} at {URL}")
        state.set("last_alert_price", price)
    return {"price": price}
```

## 4. New RSS items to Slack

Prompt: "Post new posts from this blog's RSS feed to #news every 30 minutes."

- pattern: `schedule`
- schedule: `{"kind": "every_minutes", "every_minutes": 30, "timezone": "UTC"}`
- setup_needs: `[{"kind": "notifications"}]`

```python
import feedparser
from clickruntime import on_trigger, state, notify, report_fetched

FEED = "https://blog.example.com/feed.xml"
CHANNEL = "#news"

def post(text):
    notify.slack(text, channel=CHANNEL)

@on_trigger
def run(event):
    feed = feedparser.parse(FEED)
    seen = set(state.get("seen") or [])
    new = [e for e in feed.entries if e.get("id", e.get("link")) not in seen]
    report_fetched(len(new))
    if not new:
        return {"new": 0}
    first_run = not seen
    for entry in reversed(new[:10]):
        if not first_run:
            post(f"{entry.get('title', 'New post')}\n{entry.get('link', '')}")
    ids = [e.get("id", e.get("link")) for e in feed.entries][:200]
    state.set("seen", ids)
    return {"new": 0 if first_run else len(new)}
```

## 5. Weekday sheet summary to Slack

Prompt: "On weekdays at 9am, post a summary of open rows in my Leads sheet to #sales."

- pattern: `schedule`
- schedule: `{"kind": "weekdays", "time": "09:00", "timezone": "America/New_York"}`
- setup_needs: `[{"kind": "connection", "provider": "google", "name": "work"}, {"kind": "notifications"}]`

```python
from google.oauth2.credentials import Credentials
from googleapiclient.discovery import build
from clickruntime import on_trigger, notify, credentials

SHEET_ID = "SPREADSHEET_ID_FROM_URL"
RANGE = "Leads!A1:D"
STATUS_COLUMN = "Status"
CHANNEL = "#sales"

def post(text):
    notify.slack(text, channel=CHANNEL)

@on_trigger
def run(event):
    creds = Credentials(credentials.connection("google", "work"))
    sheets = build("sheets", "v4", credentials=creds, cache_discovery=False)
    rows = sheets.spreadsheets().values().get(spreadsheetId=SHEET_ID, range=RANGE).execute().get("values", [])
    if not rows:
        return {"open": 0}
    header, body = rows[0], rows[1:]
    idx = header.index(STATUS_COLUMN)
    open_rows = [r for r in body if len(r) > idx and r[idx].strip().lower() == "open"]
    lines = [f"- {r[0]}" for r in open_rows[:20]]
    text = f"{len(open_rows)} open leads" + ("\n" + "\n".join(lines) if lines else "")
    post(text)
    return {"open": len(open_rows)}
```

## 6. Overdue rows reminder by email

Prompt: "Every weekday at 9, email me any task in my sheet that is past its due date and not done."

- pattern: `schedule`
- schedule: `{"kind": "weekdays", "time": "09:00", "timezone": "Europe/London"}`
- setup_needs: `[{"kind": "connection", "provider": "google", "name": "work"}]`

```python
from datetime import date
from google.oauth2.credentials import Credentials
from googleapiclient.discovery import build
from clickruntime import on_trigger, notify, credentials

SHEET_ID = "SPREADSHEET_ID_FROM_URL"
RANGE = "Tasks!A2:C"  # Task, Due (YYYY-MM-DD), Done (yes/no)

@on_trigger
def run(event):
    creds = Credentials(credentials.connection("google", "work"))
    sheets = build("sheets", "v4", credentials=creds, cache_discovery=False)
    rows = sheets.spreadsheets().values().get(spreadsheetId=SHEET_ID, range=RANGE).execute().get("values", [])
    today = date.today().isoformat()
    overdue = [r[0] for r in rows if len(r) >= 2 and r[1] < today and (len(r) < 3 or r[2].lower() != "yes")]
    if overdue:
        notify.email_me(f"{len(overdue)} overdue tasks", "\n".join(f"- {t}" for t in overdue))
    return {"overdue": len(overdue)}
```

## 7. Daily weather to Slack

Prompt: "Every morning at 7, post the weather for Pune to #general."

- pattern: `schedule`
- schedule: `{"kind": "daily", "time": "07:00", "timezone": "Asia/Kolkata"}`
- setup_needs: `[{"kind": "notifications"}, {"kind": "secret", "provider": "openweather", "name": "OPENWEATHER_API_KEY", "hint": "Sign in at openweathermap.org and copy a key from My API keys."}]`

```python
from clickruntime import on_trigger, notify, http, credentials

CITY = "Pune"
CHANNEL = "#general"

def post(text):
    notify.slack(text, channel=CHANNEL)

@on_trigger
def run(event):
    resp = http.get(
        "https://api.openweathermap.org/data/2.5/weather",
        params={"q": CITY, "units": "metric", "appid": credentials.secret("OPENWEATHER_API_KEY")},
    )
    resp.raise_for_status()
    data = resp.json()
    text = f"{CITY}: {data['weather'][0]['description']}, {data['main']['temp']:.0f}°C"
    post(text)
    return {"text": text}
```

## 8. API health check with alert on change

Prompt: "Every 5 minutes, check https://api.example.com/health and message #ops if it stops returning 200."

- pattern: `schedule`
- schedule: `{"kind": "every_minutes", "every_minutes": 5, "timezone": "UTC"}`
- setup_needs: `[{"kind": "notifications"}]`

```python
from clickruntime import on_trigger, state, notify, http

URL = "https://api.example.com/health"
CHANNEL = "#ops"

def post(text):
    notify.slack(text, channel=CHANNEL)

@on_trigger
def run(event):
    try:
        code = http.get(URL, timeout=10).status_code
    except Exception as exc:  # network error counts as down
        code = 0
    up = code == 200
    was_up = state.get("up")
    state.set("up", up)
    if was_up is not None and up != was_up:
        post(f"{URL} is {'back up' if up else f'down (status {code})'}")
    return {"status": code}
```

## 9. Weekly GitHub issues digest

Prompt: "Every Monday at 8am, email me the open issues labelled bug in my repo."

- pattern: `schedule`
- schedule: `{"kind": "weekly", "weekday": "monday", "time": "08:00", "timezone": "America/Los_Angeles"}`
- setup_needs: `[{"kind": "secret", "provider": "github", "name": "GITHUB_TOKEN", "hint": "Create a fine-grained personal access token in GitHub Developer settings, limited to this repository."}]`

```python
from clickruntime import on_trigger, notify, http, credentials

REPO = "owner/repo"

@on_trigger
def run(event):
    resp = http.get(
        f"https://api.github.com/repos/{REPO}/issues",
        params={"labels": "bug", "state": "open", "per_page": 50},
        headers={"Authorization": f"Bearer {credentials.secret('GITHUB_TOKEN')}", "Accept": "application/vnd.github+json"},
    )
    resp.raise_for_status()
    issues = [i for i in resp.json() if "pull_request" not in i]
    body = "\n".join(f"#{i['number']} {i['title']} {i['html_url']}" for i in issues) or "No open bugs."
    notify.email_me(f"{len(issues)} open bugs in {REPO}", body)
    return {"open_bugs": len(issues)}
```

## 10. Daily AI summary of a feed

Prompt: "Every evening, summarise today's Hacker News front page in five bullets and email it to me."

- pattern: `schedule`
- schedule: `{"kind": "daily", "time": "18:00", "timezone": "Europe/Berlin"}`
- setup_needs: `[]` (built-in AI needs no key)

```python
import feedparser
from clickruntime import on_trigger, notify, ai

FEED = "https://news.ycombinator.com/rss"

@on_trigger
def run(event):
    titles = [e.title for e in feedparser.parse(FEED).entries[:30]]
    try:
        summary = ai.generate(
            "
".join(titles),
            system="Summarise these headlines in five short bullets.",
            max_output_tokens=300,
        ).text
    except (ai.AINotIncluded, ai.AILimitReached):
        summary = "
".join(f"- {t}" for t in titles[:10])
    notify.email_me("Today on Hacker News", summary)
    return {"headlines": len(titles)}
```

## 11. Calendar: tomorrow's agenda by email

Prompt: "Every weekday at 6pm, email me tomorrow's calendar events."

- pattern: `schedule`
- schedule: `{"kind": "weekdays", "time": "18:00", "timezone": "America/Chicago"}`
- setup_needs: `[{"kind": "connection", "provider": "google", "name": "work"}]`

```python
from datetime import datetime, timedelta
from zoneinfo import ZoneInfo
from google.oauth2.credentials import Credentials
from googleapiclient.discovery import build
from clickruntime import on_trigger, notify, credentials

TZ = ZoneInfo("America/Chicago")

@on_trigger
def run(event):
    creds = Credentials(credentials.connection("google", "work"))
    cal = build("calendar", "v3", credentials=creds, cache_discovery=False)
    start = (datetime.now(TZ) + timedelta(days=1)).replace(hour=0, minute=0, second=0, microsecond=0)
    end = start + timedelta(days=1)
    items = cal.events().list(
        calendarId="primary", timeMin=start.isoformat(), timeMax=end.isoformat(),
        singleEvents=True, orderBy="startTime",
    ).execute().get("items", [])
    lines = [f"{e['start'].get('dateTime', e['start'].get('date'))[11:16] or 'all day'} {e.get('summary', '(no title)')}" for e in items]
    notify.email_me(f"Tomorrow: {len(items)} events", "\n".join(lines) or "Nothing scheduled.")
    return {"events": len(items)}
```

## 12. Stripe payment, email me

Prompt: "When a Stripe payment succeeds, email me the amount and the customer's email."

- pattern: `incoming`
- setup_needs: `[{"kind": "secret", "provider": "stripe", "name": "STRIPE_WEBHOOK_SECRET", "hint": "In Stripe, open Developers > Webhooks, select the endpoint, and reveal the signing secret."}]`

```python
from clickruntime import on_event, log, notify, verify, credentials

@on_event
def handle(event):
    if not verify.stripe(event.body, event.header("stripe-signature"), credentials.secret("STRIPE_WEBHOOK_SECRET")):
        raise ValueError("Stripe signature did not verify")
    body = event.json or {}
    if body.get("type") != "checkout.session.completed":
        return {"ignored": body.get("type")}
    session = body["data"]["object"]
    amount = (session.get("amount_total") or 0) / 100
    currency = (session.get("currency") or "").upper()
    email = (session.get("customer_details") or {}).get("email", "unknown")
    notify.email_me("Payment received", f"{amount:.2f} {currency} from {email}")
    log.info("Payment handled", extra={"event_id": event.event_id})
    return {"amount": amount, "currency": currency}
```

## 13. Form submission to a sheet

Prompt: "Whenever my Typeform is submitted, add a row with the name and email to my Signups sheet."

- pattern: `incoming`
- setup_needs: `[{"kind": "connection", "provider": "google", "name": "work"}, {"kind": "secret", "provider": "typeform", "name": "TYPEFORM_WEBHOOK_SECRET", "hint": "In Typeform, open Connect > Webhooks, edit the webhook, and set a secret."}]`

```python
from google.oauth2.credentials import Credentials
from googleapiclient.discovery import build
from clickruntime import on_event, state, verify, credentials

SHEET_ID = "SPREADSHEET_ID_FROM_URL"
RANGE = "Signups!A:C"
NAME_REF = "name"    # field.ref values from the form
EMAIL_REF = "email"

@on_event
def handle(event):
    if not verify.typeform(event.body, event.header("typeform-signature"), credentials.secret("TYPEFORM_WEBHOOK_SECRET")):
        raise ValueError("Typeform signature did not verify")
    if state.get(f"seen:{event.event_id}"):
        return {"duplicate": True}
    answers = {a["field"]["ref"]: a for a in event.json["form_response"]["answers"]}
    name = answers.get(NAME_REF, {}).get("text", "")
    email = answers.get(EMAIL_REF, {}).get("email", "")
    creds = Credentials(credentials.connection("google", "work"))
    sheets = build("sheets", "v4", credentials=creds, cache_discovery=False)
    sheets.spreadsheets().values().append(
        spreadsheetId=SHEET_ID, range=RANGE, valueInputOption="USER_ENTERED",
        body={"values": [[event.received_at, name, email]]},
    ).execute()
    state.set(f"seen:{event.event_id}", True)
    return {"added": email}
```

## 14. Form submission, send a confirmation email

Prompt: "Whenever someone fills my Tally form, send them a thank-you email from hello@mydomain.com."

- pattern: `incoming`
- setup_needs: `[{"kind": "secret", "provider": "resend", "name": "RESEND_API_KEY", "hint": "In Resend, open API Keys and create a key with sending access only."}]`

```python
from clickruntime import on_event, log, http, credentials

FROM = "hello@mydomain.com"

@on_event
def handle(event):
    fields = {f.get("label"): f.get("value") for f in event.json["data"]["fields"]}
    to = fields.get("Email")
    if not to:
        return {"skipped": "no email field"}
    resp = http.post(
        "https://api.resend.com/emails",
        headers={"Authorization": f"Bearer {credentials.secret('RESEND_API_KEY')}", "Idempotency-Key": event.event_id},
        json={"from": FROM, "to": [to], "subject": "Thanks for signing up", "text": "We got your submission and will be in touch."},
    )
    resp.raise_for_status()
    log.info("Confirmation sent", extra={"event_id": event.event_id})
    return {"sent": True}
```

## 15. New GitHub issue to Slack

Prompt: "When a new issue is opened in my repo, post it to #dev."

- pattern: `incoming`
- setup_needs: `[{"kind": "notifications"}, {"kind": "secret", "provider": "github", "name": "GITHUB_WEBHOOK_SECRET", "hint": "In the repository's Settings > Webhooks, edit the webhook and set a Secret."}]`

```python
from clickruntime import on_event, notify, verify, credentials

CHANNEL = "#dev"

def post(text):
    notify.slack(text, channel=CHANNEL)

@on_event
def handle(event):
    if not verify.github(event.body, event.header("x-hub-signature-256"), credentials.secret("GITHUB_WEBHOOK_SECRET")):
        raise ValueError("GitHub signature did not verify")
    if event.header("x-github-event") != "issues" or event.json.get("action") != "opened":
        return {"ignored": True}
    issue = event.json["issue"]
    post(f"New issue: {issue['title']}\n{issue['html_url']}")
    return {"issue": issue["number"]}
```

## 16. Shopify order alert by email

Prompt: "Email me whenever a Shopify order over $200 comes in."

- pattern: `incoming`
- setup_needs: `[{"kind": "secret", "provider": "shopify", "name": "SHOPIFY_WEBHOOK_SECRET", "hint": "In Shopify admin, open Settings > Notifications > Webhooks and copy the signing key shown under the list."}]`

```python
from clickruntime import on_event, notify, verify, credentials

THRESHOLD = 200.0

@on_event
def handle(event):
    if not verify.shopify(event.body, event.header("x-shopify-hmac-sha256"), credentials.secret("SHOPIFY_WEBHOOK_SECRET")):
        raise ValueError("Shopify signature did not verify")
    order = event.json
    total = float(order.get("total_price") or 0)
    if total < THRESHOLD:
        return {"total": total, "alerted": False}
    notify.email_me("Large order", f"Order {order.get('name', order.get('id'))}: {total:.2f} {order.get('currency', '')}")
    return {"total": total, "alerted": True}
```

## 17. Generic webhook to Discord

Prompt: "When my app sends an alert webhook, forward the message to my Discord channel."

- pattern: `incoming`
- setup_needs: `[{"kind": "secret", "provider": "discord", "name": "DISCORD_WEBHOOK_URL", "hint": "In the Discord channel's settings, open Integrations > Webhooks, create a webhook, and copy its URL."}]`

```python
from clickruntime import on_event, http, credentials

@on_event
def handle(event):
    message = (event.json or {}).get("message") or event.body[:1500]
    http.post(credentials.secret("DISCORD_WEBHOOK_URL"), json={"content": message[:1900]}).raise_for_status()
    return {"forwarded": True}
```

## 18. Daily digest of sign-ups (remember then schedule)

Prompt: "Collect sign-ups from my form as they come in and email me the list every evening at 6."

- pattern: `remember_then_schedule`
- schedule: `{"kind": "daily", "time": "18:00", "timezone": "America/Chicago"}`
- setup_needs: `[]`

```python
from clickruntime import on_event, on_trigger, state, notify

@on_event
def remember(event):
    data = event.json or {}
    pending = state.get("pending") or []
    pending.append({"at": event.received_at, "email": data.get("email", ""), "name": data.get("name", "")})
    state.set("pending", pending[-500:])
    return {"stored": len(pending)}

@on_trigger
def digest(event):
    pending = state.get("pending") or []
    if not pending:
        return {"sent": False}
    lines = [f"- {p['name']} <{p['email']}> at {p['at']}" for p in pending]
    notify.email_me(f"{len(pending)} new sign-ups today", "\n".join(lines))
    state.set("pending", [])
    return {"sent": True, "count": len(pending)}
```

## 19. SMS when a keyword appears on a page

Prompt: "Every 10 minutes, check the ticket page and text me if it says 'On sale'."

- pattern: `schedule`
- schedule: `{"kind": "every_minutes", "every_minutes": 10, "timezone": "UTC"}`
- setup_needs: `[{"kind": "secret", "provider": "twilio", "name": "TWILIO_ACCOUNT_SID", "hint": "Copy the Account SID from the Twilio Console home page."}, {"kind": "secret", "provider": "twilio", "name": "TWILIO_AUTH_TOKEN", "hint": "Copy the Auth Token from the Twilio Console home page."}]`

```python
from twilio.rest import Client
from clickruntime import on_trigger, state, http, credentials

URL = "https://tickets.example.com/event/42"
KEYWORD = "on sale"
TO = "+15550000000"
FROM = "+15551111111"

@on_trigger
def run(event):
    found = KEYWORD in http.get(URL).text.lower()
    if found and not state.get("alerted"):
        Client(credentials.secret("TWILIO_ACCOUNT_SID"), credentials.secret("TWILIO_AUTH_TOKEN")).messages.create(
            to=TO, from_=FROM, body=f"'{KEYWORD}' is on {URL}"
        )
        state.set("alerted", True)
    return {"found": found}
```

## 20. Count events per day and report weekly

Prompt: "Count how many webhook pings my service sends each day and email me a weekly total on Fridays."

- pattern: `remember_then_schedule`
- schedule: `{"kind": "weekly", "weekday": "friday", "time": "17:00", "timezone": "UTC"}`
- setup_needs: `[]`

```python
from datetime import datetime, timedelta, timezone
from clickruntime import on_event, on_trigger, state, notify

@on_event
def count(event):
    state.increment_metric("pings", ttl_days=30, bucket="day")
    return {"counted": True}

@on_trigger
def report(event):
    since = datetime.now(timezone.utc) - timedelta(days=7)
    buckets = state.query_metric("pings", bucket="day", from_time=since)
    total = sum(int(b.get("value", 0)) for b in buckets)
    notify.email_me("Weekly ping count", f"{total} pings in the last 7 days.")
    return {"total": total}
```
