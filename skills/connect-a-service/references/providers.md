# Providers

How a service gets into a script. Built-ins (email to the owner and Slack for notifications) need at most a button; services with an API key or token are a secret the person adds in the ClickRuntime dashboard; services that use OAuth are a ready-made app from `list_app_catalog` or the workspace's own app ([custom-oauth.md](custom-oauth.md)). Built-ins are a convenience, not a limit: any service with an API can be used.

Scripts read every credential on demand with `from clickruntime import credentials`: `credentials.connection(provider, name)` for a connected account, `credentials.secret(NAME)` for a saved secret. Nothing is in the environment.

Every connection is a named account of a provider, such as `google`/`work` and `google`/`personal`. Declare it as `{"kind": "connection", "provider": "google", "name": "work"}`. Which account to use is the person's choice: follow Choosing accounts in [the connect-a-service skill](../SKILL.md#choosing-accounts) (list them, ask, set up a new one only when they ask or none fits). Two accounts of one provider are two needs.

## Built in

### Email to me (`connection`, provider `email`)

- Nothing to set up. Always ready.
- Script: `notify.email_me(subject, text)`. Sends plain text to the owner's verified address only. 10 per run, 200 per day per workspace.
- Use for "email me", "let me know", "send me".

### Slack for notifications (`notifications`)

- ClickRuntime's own Slack app, used only by `notify.slack` and run alerts. Scripts never receive its token.
- Declare `{"kind": "notifications"}` and connect with `begin_connection(provider="slack", purpose="notifications")` (no name). One per workspace.
- Script: `notify.slack(text, channel="#sales")`. Always pass a channel. The person invites the ClickRuntime app to the channel (`/invite @ClickRuntime`).
- Use this whenever a script only needs to post a message to Slack. It needs no app of the person's own.

## Any other service

Look up the service's current API docs before declaring anything: its auth method, key or scope names, base URL and where the person creates credentials. Do not rely on memory. Menu names change over time; if the person cannot find a page, describe what to look for rather than inventing a URL. Public APIs without keys (RSS feeds, public JSON, status pages) need no setup.

### API key or token (`secret`)

Use when the service offers an API key, personal access token, bot token or secret URL (such as an incoming webhook URL).

- Declare `{"kind": "secret", "provider": "<service>", "name": "<NAME>", "hint": "<hint>"}`. The name is UPPER_SNAKE: the service's documented variable name if it has one, otherwise `<SERVICE>_API_KEY` or `<SERVICE>_TOKEN`. Reusing the same name lets one key serve several automations.
- A service that needs several values (an account ID and a token, a key and a webhook signing secret) gets one secret need per value. Values that are not secret (a from number, a base ID, a chat ID) go in the script.
- Ask for the least access: a restricted, read-only or fine-grained key limited to the resources the script uses, when the service offers one.
- Hint: one sentence on where the person creates it, such as "In <Service>, open the developer or API settings and create a key with read access only." Some keys take a while to activate or must be shared with a resource (a page, a base, a channel) before they work; say so in the hint when the docs say so.
- Script: send the key the way the service's docs say (usually `Authorization: Bearer ...` or an `x-api-key` header), with `http` for timeouts and retries. Use the service's own Python package only when it is simpler and declared in dependencies.

### OAuth account (`connection`)

Use when the service only offers OAuth, or the person prefers to sign in.

- Call `list_app_catalog` first and use a ready-made app when its scopes and limits cover the script.
- Google Sheets: `cr_google_sheets_picked` needs no Google app of the person's own. After connecting, the person picks the spreadsheets it may use; scripts can also create new ones. Use it unless the script must search or list Drive, watch a folder, or use other Google services; then use the workspace's own `google` app. To use a sheet that was not picked, give the person the account's `choose_files_url` from `list_connections` instead of asking for the link.
- Otherwise add the workspace's own app with [custom-oauth.md](custom-oauth.md). Take the authorization URL, token URL, scope separator and any extra auth parameters from the service's OAuth docs. Some services issue a refresh token only with an extra parameter (for example offline access or a consent prompt); check for it, or connections stop working when the first token expires.
- Scopes: only what the script uses, plus any identity scope that lets the account show its email or name. When a later script needs another scope, draft the provider again with it and have the person reconnect.
- When the person's own app is unverified or in testing mode, the consent screen may warn about it. That is expected for their own app; tell them how to continue past it if the docs describe it, and whether testing mode limits how long a connection lasts.
- Script: pass `credentials.connection(provider, name)` as the access token to the service's API or Python client.
- Resources are usually addressed by an ID inside their URL (a sheet, a document, a repository). Ask the person for the link and take the ID from it.

### Safety

Read-only use for payment services. Do not create charges, refunds, transfers or payouts. When an automation sends email to someone other than the owner, use a sending service's key; otherwise use `notify.email_me`.
