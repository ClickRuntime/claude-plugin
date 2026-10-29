---
name: look-after-an-automation
description: Check on automations that already exist. Use when the person asks what automations they have, whether one ran, why a run failed, wants one paused, resumed, run once now, renamed, changed, or deleted, or needs help from ClickRuntime support.
---

Explicit instructions from the person override this skill. If they ask for something different from these steps, follow them, as long as it stays within the safety rules below.

## Find it

1. If this chat already has the automation's `automation_id` (from an earlier result), use it. Otherwise call `list_automations` once. Match the person's words to a name or summary. If two match, ask which one. If it is not in the list and the result has `next_cursor`, call it again with that cursor. Never call `list_automations` again just to show something; use the tool for that action.
2. Call `get_automation` to read status, next runs, setup and the last run, and `list_runs` or `get_run` for run details. To show the automation to the person, call `show_automation` (view `overview`, or view `run` with a `run_id` for one run). One card per turn. Do not invent a run result, an incoming payload, or the contents of a secret.
3. A scheduled check that found nothing new, sent nothing and finished within 10 seconds shows in the run history but does not count toward monthly runs. Say this if the person asks why there are so many runs.

## Act on it

- Pause: call `pause_automation`. Say it is paused only after the tool succeeds. Events that arrive while it is paused are stored but never run, even after it resumes.
- Resume: call `resume_automation` with the `automation_id`. Say it is running again only after the tool succeeds.
- Run once now: first say in one sentence what the run does outside ClickRuntime (messages sent, rows written), then call `run_automation_now` with the `automation_id`. It waits for the run and returns its status, error and logs; if it is still running, call `get_run` with `wait: true`. For an incoming automation, `replay_event` with an `event_id` from `list_incoming_events` runs a real event again.
- Rename, or change the schedule, script, setup, or packages of a live or paused automation: call `get_script` first if the script changes, then `propose_change` with only the fields that change and `deploy: true`. The change deploys and the call waits for the result; new packages install at that point, so it takes a minute or two longer. If it is still deploying, call `wait_for_deploy`. If the deploy fails, the previous version keeps running. For a draft, use `update_draft`. Once the change deploys, report it and offer a test, as in Deploy, test, fix in `get-started`.
- A live automation needs a new scope (for example Slack `channels:history`): do the app change yourself, then hand the person only the reconnect. In one turn: call `draft_oauth_provider` with the app's current scopes plus the new one (never ask the person to edit the app's scopes), then `begin_connection` for each account that uses it, and give the person the reconnect links (plus any step in the service's own developer console that the app's `setup_notes` say a new scope needs). When they say it is done, call `propose_change` with the new `setup_needs` (and script) and `deploy: true`. A ready-made app (id starting `cr_`) cannot gain scopes: if `list_app_catalog` shows it lacks the scope, move the need to another ready-made app that has it, or to the workspace's own app, with `propose_change`, then connect that account.
- Reconnecting an account is done by the person, never by you: only they can approve the new scope on the service's consent screen. `begin_connection` returns a reconnect link; give it exactly and ask them to sign in with the same account and approve. Until they do, automations using it keep their current access. If `begin_connection` fails, give the app's page in the ClickRuntime dashboard instead (`website_url` from `draft_oauth_provider`, `/connections/custom/<id>`): it lists each connected account with a Reconnect button. Do not say you will retry the reconnect, and do not tell them to look for a reconnect option.
- Delete: call `prepare_confirmation` with `action: "delete"`. Deleting removes the automation, its runs, and its events, and happens only when the person presses Delete on the card or confirms on the link. Suggest pausing instead if the person may want it back.
- Fix a failure: follow `write-and-fix-a-script`. A deploy that stopped while installing packages shows its reason in `get_automation` (`deploy.error_code`, `layer.error`).
- Missing connection or secret: follow `connect-a-service`.
- Alerts: a failed run always emails the workspace owner. Slack alerts for failed runs are set on the automation's page in the ClickRuntime dashboard, not from the chat; point the person there when they ask.
- A problem with ClickRuntime itself (a platform error, a run that failed outside the script, a connection that will not work, a billing question): ask the person, then call `report_issue` with a clear title, the `category`, what happened and what was expected, and the `automation_id` and `run_id`. Never include keys or tokens. Script bugs are fixed with `write-and-fix-a-script`, not reported.
- Support replies: call `list_issues` or `get_issue`. A message from `operator` is the support team; `waiting_on_user` means they need something. Pass their answer on, and add what they asked for with `add_issue_message`.
- Incoming automation not firing: call `get_latest_event` (or `list_incoming_events` for several) to see whether anything arrived and whether it was rejected for a bad signature, then check the sender setup in [senders.md](../run-when-something-happens/references/senders.md).

## Without cards

If no card appears, describe the result in a sentence or two. A `confirm_url` (from `prepare_confirmation`, used for deleting) goes to the person exactly as returned; they confirm or cancel on that ClickRuntime page. After they say they have, call `get_confirmation_status` with the `confirmation_ref` and report what it shows. See `get-started` for details.

## Refuse or narrow

Decline, with a short reason, when the request is to run malware, stealth monitoring of a person, or credential theft; to bypass another service's access controls or rate limits; to move money, place a trade, or complete a purchase; or to collect health, government, or payment-card data. Never bring up run or AI limits during setup. Only when a tool says a limit was reached, say so as a fact and give the link from that tool's message exactly as returned, with no plan names, prices or advice to change plan. Monthly runs used up: automations can still be turned on or resumed and start when the period resets, but running one now waits until then. AI tokens used up: say so and offer a version without AI.

Never ask the person to paste an API key, token, password, or one-time code. Never put a secret in a tool argument. If a saved key is missing, give them its link to the ClickRuntime dashboard. If they paste a secret anyway, tell them to revoke it and create a new one in the ClickRuntime dashboard, and do not repeat the secret.
