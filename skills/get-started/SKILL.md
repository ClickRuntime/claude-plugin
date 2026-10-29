---
name: get-started
description: Introduce ClickRuntime and set up a first automation. Use when the person has just connected ClickRuntime, asks what it can do or how it works, or wants help with their first scheduled job, monitor, or webhook handler.
---

Explicit instructions from the person override this skill. If they ask for something different from these steps, follow them, as long as it stays within the safety rules below.

## What ClickRuntime is

ClickRuntime keeps work running after the chat ends. Each automation is a short Python script plus a trigger:

- a schedule (every 5+ minutes, hourly, daily, weekdays, weekly, or monthly at a time and timezone, or a cron expression), or
- a trigger link that another service sends a JSON event to (a form, a payment, a webhook).

The script can call web APIs, read and write Google Sheets, post to Slack, email the owner, and remember values between runs. The assistant builds, deploys and tests the automation itself; the person approves each tool call in this assistant's own permission prompt. Only the person connects accounts (on the provider's consent screen) and adds keys (in the ClickRuntime dashboard).

## ClickRuntime or the assistant's own reminders

Use ClickRuntime only when the work runs code, calls an outside service, delivers outside this chat (email, Slack, a sheet, an API), reacts to an outside event, or has to remember something between runs. Each automation keeps its own state between runs (values and time-bucketed counters), so it can handle only new items, alert only when a value changes, act after a threshold ("after 3 failures in a row"), keep a cooldown, resume from a cursor, or compare this week with last week. The assistant's own scheduled tasks start fresh each time and can't do this reliably. When the person only wants a reminder or a reply in this chat at a time, ClickRuntime is not needed; the assistant's own features handle that if it has them ("remind me at 5 to call mom", "every morning tell me a quote"). If it is unclear, ask one question: "Should this run on its own and send the result somewhere, or just remind you here?"

## First run

1. Call `get_profile` to confirm which account and workspace are connected. The workspace was chosen when the person connected; to use another one they reconnect ClickRuntime. If the result has `attention`, a first automation failed: offer to fix it before anything else, following `write-and-fix-a-script`.
2. If the person already asked for something specific, skip to step 4. Otherwise say in two sentences what ClickRuntime does and suggest one starter from [the starters](references/starters.md), leading with "Email me a summary of this page every morning" unless another starter clearly fits them better. Mention two others in one line. Starters need no connections or keys, so the first automation can be live in a few minutes.
3. For a starter, ask only for what it needs (a URL, a time, a place), then save it with `save_draft`, its `template_id` and `deploy: true`. Once it is live, say what one run does (for example "this sends you one email now") and call `run_automation_now` so they see the first result. Follow [Deploy, test, fix](#deploy-test-fix) until a run succeeds.
4. When they describe their own automation, follow `run-on-a-schedule` for cadence-based work or `run-when-something-happens` for event-based work.
5. If they ask what they already have, call `list_automations`; its result shows the automations list.

## Deploy, test, fix

Turning on, running, testing and applying a change are your own tool calls; the person approves each in this assistant's permission prompt. Deploy, then stop once to report and offer a test. When the person says yes, work through test and fix yourself, without stopping to ask between steps:

1. Deploy. When setup is ready, call `update_draft` with `deploy: true` (`save_draft` with `deploy: true` when nothing needs setting up; for a live or paused automation, `propose_change` with `deploy: true`). `enable_automation` or `apply_change` with the `automation_id` deploys with no other change. The call waits about 45 seconds. If the result says it is still deploying, call `wait_for_deploy` with the `automation_id` until it finishes; installing packages can take a few minutes. If the deploy failed, fix the cause from `deploy.error` and deploy again: a draft that failed to deploy stays a draft and nothing runs; after a failed change, the previous version keeps running.
2. Report and offer a test. After every create or change that deploys, tell the person in a few sentences: the automation is set up (or changed) and live; when it runs next (the next run time for a schedule; for an incoming automation, that it runs when the next event arrives at its trigger address); what one run does outside ClickRuntime (emails sent, Slack posts, rows written, calls to other services). Then offer to run a test now to check it works as expected, and wait for their answer. Do not end with only the next run time. If they decline, stop there.
3. Test, when they say yes. Call `run_automation_now` for a schedule, `send_test_event` with a realistic sample body for an incoming automation, or `replay_event` with an `event_id` from `list_incoming_events` to run a real event again. Each waits for the run and returns its status, error and logs; if it is still running, call `get_run` with `wait: true`.
4. Check and fix. Read the run's status, error and logs. If it failed or did the wrong thing, follow Fixing a failed run in `write-and-fix-a-script`, call `propose_change` with the fix and `deploy: true`, then test again. Repeat without asking between attempts.
5. Report back once a run succeeds and its result matches what the person asked for: what the test run did, any fixes you made along the way, and when it runs next. After about 5 fix attempts that do not work, stop and tell the person what is blocking it and what you tried.

Ask the person only for what you cannot do: connecting an account, adding a key in the ClickRuntime dashboard, or a requirement that is unclear. Say it is live, changed or works only once a tool result shows it.

## Cards and steps

The tools you work with (saving, deploying, running, reading) show no card, so you can call them as often as the work needs. Cards appear only for the delete confirmation and `show_automation`. Setup (connections, apps, keys) has no card: give the links from `get_setup_status`, `begin_connection` and `draft_oauth_provider` in the text, all at once (see The setup loop in `connect-a-service`). Call `show_automation` once a piece of work is done, or when the person needs to look at something (the live automation, a failure you could not fix, the trigger link to paste, the schedule picker); not after every step. Make at most one card-showing call per turn, since only the last card in a turn is shown. Never say a button exists unless a card showing it is on screen.

Deleting is the one action that needs a confirm card: call `prepare_confirmation` with `action: "delete"`. If the person asks to confirm something else on a card first, `prepare_confirmation` also takes `enable`, `resume`, `run_now`, `apply_change` and `send_test_event_live`; after they confirm, call `get_confirmation_status` with the `confirmation_ref`.

## Without cards

Some assistants do not show cards. If a tool result has no card, describe the result in a sentence or two instead. When `prepare_confirmation` returns a `confirm_url`, give the person that link exactly as returned (never write, shorten or change a ClickRuntime link yourself) and say it opens a ClickRuntime page where they confirm or cancel. After they say they have, call `get_confirmation_status` with the `confirmation_ref` and report what it shows. The link works once and expires after 30 minutes; if it has expired, call `prepare_confirmation` again.

## Limits to state when relevant

- Python 3.11 with common packages built in. Other PyPI packages can be declared with a pinned version; they install when the script deploys, which makes that deploy take a minute or two longer. Outbound calls are HTTPS only, and scripts can't start processes.
- Shortest schedule is every 5 minutes.
- Email to yourself and Slack messages (through ClickRuntime's own Slack app) are built in. Reading from Slack or posting as your own Slack bot, Google, and other OAuth services are connected through the person's own app for that service, set up once in the ClickRuntime dashboard. Several accounts of one service (for example work and personal Google) can be connected under names the person picks. Keys for any other service are added in the ClickRuntime dashboard, not in the chat.
- Scripts can use AI (`clickruntime.ai`) with no API key, within the workspace's monthly AI tokens, including on the free plan. Prefer automations that work without AI. When a step needs AI (classifying, summarising, extracting from free text), use ClickRuntime's built-in AI. The ClickRuntime SDK gives scripts an AI model with no API key to set up: `from clickruntime import ai`, then `ai.generate(prompt, system=None, max_output_tokens=512, json_schema=None)`, which returns a result with `.text` (and `.json` when `json_schema` is given). Do not ask which AI provider or model to use and do not ask for an AI key: tell the person the automation uses ClickRuntime's built-in AI. Only when the person asks for a specific provider of their own (for example their OpenAI or Anthropic key), save that key as a secret and call that provider's API instead.

## Refuse or narrow

Decline, with a short reason, when the request is to run malware, stealth monitoring of a person, or credential theft; to bypass another service's access controls or rate limits; to move money, place a trade, or complete a purchase; or to collect health, government, or payment-card data. Never bring up run or AI limits during setup. Only when a tool says a limit was reached, say so as a fact and give the link from that tool's message exactly as returned, with no plan names, prices or advice to change plan. Monthly runs used up: automations can still be turned on or resumed and start when the period resets, but running one now waits until then. AI tokens used up: say so and offer a version without AI.

Never ask the person to paste an API key, token, password, or one-time code. Never put a secret in a tool argument. If a saved key is missing, give them its link to the ClickRuntime dashboard. If they paste a secret anyway, tell them to revoke it and create a new one in the ClickRuntime dashboard, and do not repeat the secret.
