---
name: run-on-a-schedule
description: Run work repeatedly on a cadence and deliver the result outside the chat. Use when the person says every hour, every N minutes, every morning, on weekdays, weekly, check daily, keep an eye on, monitor a page or price, poll an API, or set up a cron job, and the work should keep going after this chat.
---

Explicit instructions from the person override this skill. If they ask for something different from these steps, follow them, as long as it stays within the safety rules below.

## Decide first

- Use this skill when the work runs code, calls an outside service, or delivers somewhere other than this chat (email, Slack, a sheet, an API). If the person only wants a reminder or a reply in this chat at a time, ClickRuntime is not needed; do not call ClickRuntime tools for it.
- A one-off question ("what's the weather now") is answered directly, not automated.
- If the work should start because something happened elsewhere (a form, a payment, a webhook), use `run-when-something-happens` instead.

## Steps

1. Restate the outcome in one or two sentences. Ask only for what is missing: what to check or do, how often, and where the result goes. If the hour matters and the timezone is unknown, ask for it. If the script will use outside services (Google, Slack, any connected app or saved key), choose the accounts with the person now, as in Choosing accounts in `connect-a-service`, in the same message as any other question.
2. Build the schedule object from [draft-shape.md](references/draft-shape.md). The shortest cadence is every 5 minutes, and `every_minutes` must divide 60 (5, 6, 10, 12, 15, 20, 30). If the person asks for something faster, offer every 5 minutes.
3. Write the script with `@on_trigger` against [the runtime reference](../write-and-fix-a-script/references/runtime.md). Start from a matching template in [recipes.md](../write-and-fix-a-script/references/recipes.md) when one fits. Use only the listed packages. Prefer `notify.email_me` when the person says "email me" or "tell me"; it needs no setup. If the script checks for new items (rows, emails, orders, issues, feed entries), it is a polling script: it must call `report_fetched(n)` once right after the fetch and return straight away when `n` is 0 (see Polling for new items in the reference). Scripts that do the same work every run do not call it.
4. List every connection and secret the script reads with `clickruntime.credentials` in `setup_needs`. Each connection names the account the person chose in step 1 (`{"kind": "connection", "provider": "google", "name": "work"}`). Add `{"kind": "notifications"}` if the script calls `notify.slack`. For secrets, name them and write the hint as in [providers.md](../connect-a-service/references/providers.md).
5. Call `save_draft` with `pattern: "schedule"`, `name`, `summary`, `schedule`, `script`, and `setup_needs`. Its result lists the next three run times. Do not invent run results. If the person wants to adjust the timing themselves, call `show_automation` with view `schedule`.
6. If the person changes the cadence in the chat, call `update_draft` with the new `schedule` (the schedule picker saves it itself).
7. Call `get_setup_status`. If anything is not ready, follow The setup loop in `connect-a-service`: give the person every remaining step in one reply, each with its link, and wait until they say they have finished.
8. When setup is ready, deploy it: `update_draft` with `deploy: true` (or `save_draft` with `deploy: true` in step 5 when `setup_needs` is empty). If the result says it is still deploying, call `wait_for_deploy`. Say the automation is live only when the result shows the deploy succeeded, and include the next run time, what one run does (for example "this emails you"), and an offer to run a test now. Wait for the answer.
9. If they say yes: call `run_automation_now`, read the run's status, error and logs, and if it fails, fix and redeploy with `propose_change` and `deploy: true`, then run it again, as in Deploy, test, fix in `get-started`, until a run succeeds. Then report what the test did and any fixes, and call `show_automation` so the person sees it.

## Without cards

If no card appears, describe the result in a sentence or two. A `confirm_url` (from `prepare_confirmation`, used for deleting) goes to the person exactly as returned; they confirm or cancel on that ClickRuntime page. After they say they have, call `get_confirmation_status` with the `confirmation_ref` and report what it shows. See `get-started` for details.

## Refuse or narrow

Decline, with a short reason, when the request is to run malware, stealth monitoring of a person, or credential theft; to bypass another service's access controls or rate limits; to move money, place a trade, or complete a purchase; or to collect health, government, or payment-card data. Scraping a public page politely is fine; logging in as someone else or evading blocks is not. Never bring up run or AI limits during setup. Only when a tool says a limit was reached, say so as a fact and give the link from that tool's message exactly as returned, with no plan names, prices or advice to change plan. Monthly runs used up: automations can still be turned on or resumed and start when the period resets, but running one now waits until then. AI tokens used up: say so and offer a version without AI.

Never ask the person to paste an API key, token, password, or one-time code. Never put a secret in a tool argument. If a saved key is missing, give them its link to the ClickRuntime dashboard. If they paste a secret anyway, tell them to revoke it and create a new one in the ClickRuntime dashboard, and do not repeat the secret.
