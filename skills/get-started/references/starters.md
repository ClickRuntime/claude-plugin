# Starters

One-message first automations. Each one needs no connections and no keys (`setup_needs: []`), emails the owner with `notify.email_me`, and survives a bad page or feed without failing the run. When you save one, pass its id as `template_id` in `save_draft`. Ask only for what the starter needs (usually a URL and a time). Use the person's timezone if you know it, else ask once.

Prefer a starter whose first run will send something, so the person sees it work. Once it is live, say what one run does and run it once with `run_automation_now` so they see the first result. A draft cannot run.

| id | Offer it as | Trigger |
|---|---|---|
| `page-summary-email` | "Email me a summary of this page every morning" | daily |
| `page-changed-email` | "Email me when this page changes" | hourly |
| `feed-digest-email` | "Email me the new posts from this blog or news feed every morning" | daily |
| `hn-top-email` | "Email me the top Hacker News stories every morning" | daily |
| `weather-email` | "Email me tomorrow's weather every evening" | daily |
| `site-up-email` | "Email me if my website goes down" | every 15 minutes |
| `keyword-watch-email` | "Email me when a page mentions a word" | hourly |
| `price-watch-email` | "Email me when a price on a page drops below a number" | hourly |

## page-summary-email

Prompt: "Every morning at 8, email me a summary of https://example.com/news."

- pattern: `schedule`
- schedule: `{"kind": "daily", "time": "08:00", "timezone": "Europe/London"}`
- setup_needs: `[]`

Uses AI when the plan includes it and falls back to the page's first paragraphs otherwise, so it works on every plan.

```python
from bs4 import BeautifulSoup
from clickruntime import on_trigger, notify, http, log, ai

URL = "https://example.com/news"

@on_trigger
def run(event):
    resp = http.get(URL, timeout=30)
    resp.raise_for_status()
    soup = BeautifulSoup(resp.text, "html.parser")
    title = soup.title.get_text(strip=True) if soup.title else URL
    paragraphs = [p.get_text(" ", strip=True) for p in soup.find_all("p")]
    text = "\n".join(p for p in paragraphs if len(p) > 40)[:12000]
    try:
        body = ai.generate(text, system="Summarise this page in five short bullets.", max_output_tokens=400).text
    except ai.AIError as e:
        log.info(f"AI not used: {type(e).__name__}")
        body = "\n\n".join(paragraphs[:5]) or "The page had no readable text today."
    notify.email_me(f"Today: {title}", f"{body}\n\n{URL}")
    return {"chars": len(text)}
```

## page-changed-email

Recipe 1 in [recipes.md](../../write-and-fix-a-script/references/recipes.md) ("Page changed, email me"), unchanged. The first run saves a snapshot and sends nothing, so tell the person the first email comes after the page changes.

## feed-digest-email

Prompt: "Every morning at 7, email me new posts from https://example.com/feed."

- pattern: `schedule`
- schedule: `{"kind": "daily", "time": "07:00", "timezone": "UTC"}`
- setup_needs: `[]`

```python
import feedparser
from clickruntime import on_trigger, notify, state, report_fetched

FEED = "https://example.com/feed"

@on_trigger
def run(event):
    seen = set(state.get("seen") or [])
    entries = feedparser.parse(FEED).entries[:30]
    new = [e for e in entries if e.get("link") and e.link not in seen]
    report_fetched(len(new))
    if not new:
        return {"new": 0}
    if not seen:
        new = new[:5]  # first run: send the latest few so the person sees it work
    lines = [f"- {e.get('title', e.link)}\n  {e.link}" for e in new]
    notify.email_me(f"{len(new)} new posts", "\n".join(lines))
    state.set("seen", [e.link for e in entries if e.get("link")])
    return {"new": len(new)}
```

## hn-top-email

`feed-digest-email` with `FEED = "https://news.ycombinator.com/rss"` and the email subject "Top on Hacker News". Send the top 10 every day instead of only new ones (drop the `seen` state and `report_fetched`).

## weather-email

Prompt: "Every evening at 6, email me tomorrow's weather in Berlin."

- pattern: `schedule`
- schedule: `{"kind": "daily", "time": "18:00", "timezone": "Europe/Berlin"}`
- setup_needs: `[]`

Look up the city's latitude and longitude yourself and put them in the script. Open-Meteo needs no key.

```python
from clickruntime import on_trigger, notify, http

PLACE, LAT, LON, TZ = "Berlin", 52.52, 13.41, "Europe/Berlin"

@on_trigger
def run(event):
    resp = http.get("https://api.open-meteo.com/v1/forecast", params={
        "latitude": LAT, "longitude": LON, "timezone": TZ, "forecast_days": 2,
        "daily": "temperature_2m_max,temperature_2m_min,precipitation_probability_max",
    }, timeout=30)
    resp.raise_for_status()
    d = resp.json()["daily"]
    body = (f"{PLACE} tomorrow: {d['temperature_2m_min'][1]}–{d['temperature_2m_max'][1]} °C, "
            f"{d['precipitation_probability_max'][1]}% chance of rain.")
    notify.email_me(f"Weather tomorrow in {PLACE}", body)
    return {"sent": True}
```

## site-up-email

Recipe 8 in [recipes.md](../../write-and-fix-a-script/references/recipes.md) ("API health check with alert on change") pointed at the person's site, every 15 minutes, with `notify.email_me` instead of `notify.slack` and `setup_needs: []`, emailing on a change between up and down. It only emails on a change, so run it once now with `run_automation_now` to show it working.

## keyword-watch-email

Prompt: "Every hour, email me when https://example.com/jobs mentions 'remote'."

- pattern: `schedule`
- schedule: `{"kind": "hourly", "minute": 0, "timezone": "UTC"}`
- setup_needs: `[]`

```python
from bs4 import BeautifulSoup
from clickruntime import on_trigger, notify, http, state

URL, WORD = "https://example.com/jobs", "remote"

@on_trigger
def run(event):
    resp = http.get(URL, timeout=30)
    resp.raise_for_status()
    found = WORD.lower() in BeautifulSoup(resp.text, "html.parser").get_text(" ").lower()
    if found and not state.get("found"):
        notify.email_me(f"'{WORD}' appeared on the page", URL)
    state.set("found", found)
    return {"found": found}
```

## price-watch-email

Recipe 3 in [recipes.md](../../write-and-fix-a-script/references/recipes.md) ("Price drop watch").
