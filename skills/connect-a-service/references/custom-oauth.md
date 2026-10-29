# Connecting any OAuth service

Use this when a service is not built in, uses OAuth 2.0 (authorization code flow), and no ready-made app from `list_app_catalog` covers the automation's scopes and limits. The person registers their own OAuth app with that service; ClickRuntime stores its client ID and secret, runs the consent screen, and refreshes the token before each run.

## Steps

1. Call `list_connections`. If an app for the service is already `ready`, skip to step 6 (and reuse one of its `accounts` if it is the right one). If it is a `draft`, skip to step 4 with its slug.
2. Fetch and read the service's current developer documentation. Use web search or fetch the pages; do not work from memory, because consoles and endpoints change. Read at least the OAuth (authorization code) reference and the page on creating an app. Find:
   - the authorization URL and the token URL (both https),
   - the scopes the script needs, and how the service joins them (space or comma),
   - whether the token request expects the client ID and secret in the form body or as HTTP Basic,
   - whether it supports PKCE, and whether it issues refresh tokens,
   - any extra authorization parameters it requires (for example `access_type=offline` or `owner=user`),
   - where the person creates the app: the console URL, and every button, menu and field on the way, exactly as the docs label them,
   - where the redirect URL goes, where scopes are chosen, and where the client ID and secret are shown.
   If the docs do not say, use the defaults (space, body, no PKCE, refresh on). Do not guess endpoints or console steps: if you cannot find them, say so and ask the person for the docs link.
3. Call `draft_oauth_provider` with a short lowercase `slug` (for example `notion`), the `name`, both URLs, the fewest `scopes`, the options from step 2, the `docs_url`, and `setup_notes`: the setup guide, written as described in [The setup guide](#the-setup-guide). It never takes the client ID or secret. Its result has `website_url`, the guide.
4. Tell the person in a sentence or two what is needed (an app of their own in that service, about five minutes) and give `website_url` as a clickable link, exactly as returned. Do not repeat the steps or the redirect URL in the chat: the guide has them, with copy buttons and the boxes to paste the client ID and secret into. Never ask them to paste the client ID or secret into the chat. If other setup steps are pending, give this link in the same list as theirs (see The setup loop in the skill).
5. Stop and wait until the person tells you they have finished. Then call `get_setup_status` (or `list_connections` when there is no automation yet) and check the app is `ready`; if it is still a draft, say what is missing and give the guide link again.
6. When the app is ready, call `begin_connection` with the slug and the account chosen for the setup need (such as `work`; ask only if none was chosen), and give its `url` as a clickable link. The person approves on the service's consent screen. When they say so, call `get_setup_status` for the next step.

## The setup guide

`setup_notes` is rendered on the setup page as numbered steps. ClickRuntime adds the final step (connecting the account) itself. Write it for someone who has never opened the service's developer console: say what to open, what to click and what to type, in order, using the labels the person will see on screen.

Format:

- `## Step title` starts a step. Use one heading per step, with a short imperative title ("Create a Slack app", "Add the redirect URL"). Aim for three to seven steps.
- Plain sentences under it. A blank line starts a new paragraph.
- `**Create New App**`: a button, menu, tab or field name, exactly as labelled in the service.
- `[api.slack.com/apps](https://api.slack.com/apps)`: a page to open. Link straight to the console page, not a docs overview. https only.
- `` `ClickRuntime automations` ``: a value to type or paste. Shown with a copy icon.
- `{{redirect_url}}`: the ClickRuntime callback URL, with a copy icon. It can sit inside a sentence. Always include it in the step where the service asks for the redirect or callback URL.
- `{{scopes}}`: every scope of the app, each with a copy icon. Put it on its own line in the step where scopes are chosen. Do not type scopes out by hand.
- `{{client_id}}` and `{{client_secret}}`: the boxes where the person pastes the two values, each on its own line. Put them in the step where the service shows the values, right after saying where to find each one. A Save button appears after the last one. If you leave them out, the page adds a separate paste step.
- `## After connecting`: optional last section, shown with the connect step, for anything the person does after approving (for example inviting a bot to channels).

Rules:

- The person reads the guide on the ClickRuntime setup page. Never tell them to open it, and never point `{{redirect_url}}` at anything but the service's redirect or callback URL field. It is not a link to ClickRuntime.
- `{{redirect_url}}`, `{{client_id}}` and `{{client_secret}}` are required; the tool rejects a guide without them. Write titles without numbers (not "Step 3: ..."); the page numbers the steps.
- Every step comes from the docs you read in step 2. If a label might differ (for example a console redesign), describe what to look for instead of inventing a name.
- Say what each step achieves when it is not obvious ("This is where the service sends you back after you approve").
- Mention anything that would otherwise stop the connection: test users, publishing state, admin approval, a plan that lacks API access.
- No keys, tokens or secrets, and no ClickRuntime internals (slugs, `credentials.connection`, setup_needs).
- When you draft the same slug again, send the full guide again; it replaces the old one.

Example (Slack):

```
## Create a Slack app
Go to [api.slack.com/apps](https://api.slack.com/apps) and sign in to your Slack workspace. Click **Create New App**, then **From scratch**. For **App Name** enter `ClickRuntime automations`, pick your workspace and click **Create App**.

## Add the redirect URL
In the left sidebar, open **OAuth & Permissions**. Under **Redirect URLs**, click **Add New Redirect URL**, paste {{redirect_url}}, click **Add**, then **Save URLs**.

## Add the permissions
On the same page, under **Scopes** → **Bot Token Scopes**, click **Add an OAuth Scope** and add each of these:
{{scopes}}
You don't need to click **Install to Workspace**; connecting installs the app for you.

## Copy the Client ID and Client Secret
In the left sidebar, open **Basic Information** and scroll to **App Credentials**. Copy the **Client ID** and paste it here:
{{client_id}}
Click **Show** next to **Client Secret**, copy it and paste it here:
{{client_secret}}

## After connecting
In each channel your automations use, type `/invite @ClickRuntime automations`.
```

## In the script

Declare the account in `setup_needs` as `{"kind": "connection", "provider": "notion", "name": "work"}`. Read its access token with `credentials.connection` inside the function that calls the service, and send it as a Bearer token:

```python
from clickruntime import http, credentials


def call_service(path):
    token = credentials.connection("notion", "work")
    headers = {"Authorization": "Bearer " + token}
    response = http.get("https://api.notion.com/v1" + path, headers=headers, timeout=30)
    response.raise_for_status()
    return response.json()
```

Never log, print, return, or store the token.

## If something goes wrong

- `provider_incomplete`: the client ID or secret is not saved yet. Send the person to the guide (`website_url`).
- The service shows a redirect URI mismatch: the callback URL in the service's app settings must match `callback_url` exactly, including `https` and the path.
- The connection later shows `needs_reauth`: the service revoked the token or does not issue refresh tokens. Call `begin_connection` again with the same slug and account name.
- To change endpoints or scopes, call `draft_oauth_provider` again with the same slug and the full `setup_notes`; the saved credentials are kept. After a scope change each connected account must reconnect to grant the new scope: call `begin_connection` with the same slug and account name and give its link. The app's page (`website_url`) also lists every account with a Reconnect button; give that link if `begin_connection` fails.
- The person says a step doesn't match what they see: read the service's docs again, then draft the same slug with corrected `setup_notes`.
