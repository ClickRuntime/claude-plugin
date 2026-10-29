---
name: connect-a-service
description: Get an automation's connections and secrets ready. Use when a draft needs Slack, Google, or another service connected (including any service that uses OAuth), needs an API key or signing secret, or when the person says "I've added it", "I connected it", or asks how to connect a service.
---

Explicit instructions from the person override this skill. If they ask for something different from these steps, follow them, as long as it stays within the safety rules below.

Keys never pass through the chat. Connections use the provider's own consent screen; secrets are typed in the ClickRuntime dashboard.

## The setup loop

Setup is a short list of things the person does outside the chat: connect accounts, set up an app of their own, add keys. Give them everything in one reply, with a link for each, then wait.

1. Call `get_setup_status` with the `automation_id`. It returns `steps` (what is left) and a `next_step` that says, for each step, how to get its link.
2. Get every link first. For each missing connection, call `begin_connection` with the item's `connect_args` and use its `url`. For a service whose app does not exist yet, call `draft_oauth_provider` (see [custom-oauth.md](references/custom-oauth.md)) and use its `website_url`, the setup guide; that account is connected later, once the app is ready. For a missing key, use the item's `website_url`, which opens that key in the ClickRuntime dashboard. Calling these tools for several services in one turn is fine.
3. In one reply, give every step as a numbered list with one clickable link each, for example `1. [Connect your Google account (work)](<url>)`. Use each link exactly as the tool returned it; never write, shorten or change a link, and do not add steps that are not listed. Say in a few words what each one is for. For every connection link, first or reconnect, also say in plain words what access the person is granting, from the result's `scopes_plain` (turn scope names into what they allow, such as "read and edit your spreadsheets"), so they know every permission before they open the link. For a key, give the one-sentence hint from providers.md for where they create it. For Slack, say which channels to invite the app to. Do not describe the steps on the pages the links open.
4. Ask the person to tell you when they have finished, and stop. A connection link works once and only for the signed-in person; asking for a new one replaces the old one.
5. When they say they are done, call `get_setup_status` again. If something is still missing, say which and give its link again (call `begin_connection` again for a connection). For an app that has just become ready, call `begin_connection` for its account and give that link.
6. When `ready` is true, setup is done. Deploy and test it yourself, following Deploy, test, fix in `get-started`: `update_draft` with `deploy: true` for a draft (or `enable_automation`), `resume_automation` for a paused automation.

How each kind of need is handled:

- `connection` with provider `email`: nothing to do. Email to yourself is built in.
- `connection` with a ready-made app (`managed: true`, id starting `cr_`): there is no app to set up. Call `begin_connection` with the item's `connect_args` and give its `url`. If the item has `app_missing_scopes` or `available: false`, the app cannot serve this automation: follow its `next_step` and switch the need to another app.
- `connection` with the slug of the workspace's own OAuth app, including `slack` and `google`: if its `setup_status` is `missing`, follow [custom-oauth.md](references/custom-oauth.md) (take the provider's values from its OAuth docs, as in Any other service in [providers.md](references/providers.md)); if it is `draft`, give its `website_url`, the setup guide; when it is `ready`, call `begin_connection` with the item's `connect_args` and give its `url`.
- `notifications`: the workspace notifications Slack (ClickRuntime's own app), used by `notify.slack` and run alerts, separate from Slack accounts a script reads. Call `begin_connection` with `provider: "slack"` and `purpose: "notifications"` (no name) and give its `url`. If the result says it is not available right now, say so and offer `notify.email_me` instead. Ask which channel to post to; the person invites the ClickRuntime app to it.
- `secret`: give the item's `website_url`, which opens the ClickRuntime dashboard on that secret.

## Choosing accounts

Do this before writing the script, because the script names each account in `credentials.connection(provider, name)`.

1. Call `list_connections`. For each service the automation needs, look at that service's `accounts` (each has a `name`, a `status` and usually an `account_label` such as an email address or workspace).
2. Ask the person in one message, covering every service at once:
   - One account connected: "Use your Google account work (me@example.com)?"
   - Several: list them by name and label and ask which one to use.
   - None: say it will be connected during setup and that you'll call it `work` unless they want another name.
3. Use what they choose. Set up a new account only when they ask for one or none of the existing ones fits; then ask for its short lowercase name (letters, digits, underscores), such as `work` or `personal`.
4. Use exactly those names in `setup_needs` and in the script. If the script uses two accounts of one service, such as work and personal Google, declare two needs.
5. Saved keys work the same way: if `list_connections` already has a secret with the name you would use, say you'll reuse it.

If a draft result has `account_check`, a need names an account that does not exist while the service already has accounts. Ask the person which account to use before anything else, and fix the draft with `update_draft`.

## Choosing needs when writing a draft

- Prefer built-ins: `notify.email_me` for "email me", and `notify.slack` with a `notifications` need for "post to Slack". Only when a script must read from Slack or act as the person's own bot does it need a `slack` connection, which is the workspace's own Slack app (slug `slack`), not a built-in. Google is not built in either: unless a ready-made app fits (below), the person adds their own Google app once (slug `google`), then connects as many Google accounts as they like. Built-ins are a convenience, not a limit.
- Every connection need names an account: `{"kind": "connection", "provider": "google", "name": "work"}`, chosen with the person as in [Choosing accounts](#choosing-accounts). Never pick or invent an account name without asking. Only `email` has no name.
- A script that calls `notify.slack` declares `{"kind": "notifications"}`; it posts through the workspace notifications Slack.
- For a service that offers an API key or personal token, declare a `secret` need with an UPPER_SNAKE name and a one-sentence `hint` written as in providers.md. Use the provider's documented name so the person can reuse it across automations.
- For a service that only offers OAuth (or where the person prefers to sign in), call `list_app_catalog` first. It lists ClickRuntime's ready-made apps: apps already registered with the service, so the person only signs in. Compare each app's `scopes` and `limits` with what the script needs (a limit such as "only files the person picks" rules an app out when the script works on a file the person names by ID). When one covers it, use it: declare `{"kind": "connection", "provider": "<app id>", "name": "<account>", "scopes": [...]}` and do not ask the person to choose between it and their own app. Decide per automation; another automation for the same service may need a different app. The list may be empty.
- When no ready-made app fits, add the service as the workspace's own OAuth app: follow [custom-oauth.md](references/custom-oauth.md), then declare `{"kind": "connection", "provider": "<slug>", "name": "<account>"}`. Call `list_connections` first: the app may already exist.
- Check the service's own API docs for its current auth method, key or scope names, and where the person creates them, before declaring the need. Do not rely on memory.
- Ask for the least access: a restricted or read-only key when the provider offers one.
- Only email and Slack notifications are built in. For every other service, follow Any other service in providers.md. Do not claim a built-in connection exists for it.

## Without cards

Setup has no card: the links are always in the text. For other results, if no card appears, describe the result in a sentence or two. A `confirm_url` (from `prepare_confirmation`, used for deleting) goes to the person exactly as returned; they confirm or cancel on that ClickRuntime page. After they say they have, call `get_confirmation_status` with the `confirmation_ref` and report what it shows. See `get-started` for details.

## Refuse or narrow

Decline, with a short reason, when the request is to run malware, stealth monitoring of a person, or credential theft; to bypass another service's access controls or rate limits; to move money, place a trade, or complete a purchase; or to collect health, government, or payment-card data. Do not help obtain someone else's credentials.

Never ask the person to paste an API key, token, password, or one-time code. Never put a secret in a tool argument. If a saved key is missing, give them its link to the ClickRuntime dashboard. If they paste a secret anyway, tell them to revoke it and create a new one in the ClickRuntime dashboard, and do not repeat the secret.
