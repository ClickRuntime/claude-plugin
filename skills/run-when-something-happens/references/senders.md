# Senders

How to point a service at an automation's trigger link. The link comes from `get_setup_status` (or `save_draft`) for an incoming automation. It accepts `POST` and is easiest to use with a JSON body. It does nothing until the automation is live; events sent before then are stored so the script can be checked with `get_latest_event`.

## Find the details in the sender's docs

Every service sets up webhooks differently, and menus and payloads change. Look up the sender's current webhook documentation (search the web or read its developer docs) for:

- Where webhooks are added: a settings page, a per-form or per-project integration, or only through its API.
- Which events exist, and which ones the job needs. Choose only those.
- The content type it sends, and whether JSON can be chosen.
- Whether and how it signs requests: the header, the algorithm, what is signed, and where the secret is shown.
- Its payload shape, and whether it has a test or redeliver button.

Use only fields the docs or a real delivery show. If the docs are unclear, have the person send one test event and read it with `get_latest_event` before writing the script against it. Never invent field names.

## Registering the webhook through the API

Many services can create webhooks through their API (often a `webhooks` or `webhook_subscriptions` endpoint). When the automation already has a key or connection for the sender, for discovery, other setup, or because the person offered one, register it with `run_setup_script` (`purpose: "setup"`) instead of giving the person steps: point it at the trigger link, choose the events, and set a signing secret if the API allows. Make the script safe to run twice (find an existing webhook for the same URL first) and put the returned webhook id in the automation as a constant, so a `@state_cleaner` can remove it later. Do not ask for a key only to do this. If the API cannot do it or the call fails, give the person the manual steps from the sender's docs.

Some services allow webhooks only through the API. If there is no key and the person does not want to add one, offer polling on a schedule instead.

## Signing

When a sender signs requests, set `signature` on `save_draft` (or `update_draft`, or `propose_change` for a live automation): `provider`, `secret_name`, and `require`. List the same secret as a setup need; the person saves its value in the ClickRuntime dashboard. Then `event.verified` reports the result, and with `require: true`, requests that fail the check never reach the script.

- Built-in providers: `stripe`, `github`, `shopify`, `typeform`, `standard_webhooks` (Svix and other services with `webhook-id`, `webhook-timestamp`, `webhook-signature` headers).
- Any other sender that signs the raw body with HMAC-SHA256: provider `hmac_sha256`, with settings from its docs: `header` (left out, the first of `x-signature`, `x-signature-256`, `x-webhook-signature`, `x-hub-signature-256`, `tally-signature` is read), `encoding` (`hex` or `base64`; left out, either is accepted), and `prefix` (text before the signature such as `sha256=` or `v1=`; left out, an optional `sha256=` is removed).
- A sender that signs something other than the raw body, such as a timestamp plus the body: verify in the script with `verify.generic_hmac_sha256` on the exact message its docs describe, using `event.body` and `event.header(...)`.
- A sender with no signing: if it can add a custom header, add one such as `x-api-key` with a value saved as a secret, and compare it in the script with `hmac.compare_digest`. Otherwise keep the trigger link private.

## Bodies that are not JSON

Some senders post `application/x-www-form-urlencoded` or `multipart/form-data`. Choose JSON if the sender offers it. Otherwise `event.json` may be `None` or an object parsed from the form fields, and some senders put the JSON inside one field. Check one real delivery with `get_latest_event` and read whichever is present.

## Services without webhooks

- Poll the service's API on a schedule (`run-on-a-schedule`) and keep the last seen id or time in state.
- Or, where the service runs user scripts on its own events (for example Google Apps Script on a form or sheet), have that script `POST` JSON to the trigger link.
- Zapier, Make and similar tools can forward events with an HTTP or webhook action: `POST`, the trigger link as URL, a JSON body.

## Testing

Use the sender's own "send test" or redeliver button if it has one, or submit or create one real item. To check the script without the sender, call `send_test_event` with a sample body. For a real delivery, call `get_latest_event`. To try the link by hand:

```bash
curl -X POST "PASTE_TRIGGER_LINK_HERE" \
  -H "Content-Type: application/json" \
  -d '{"type": "test", "message": "hello"}'
```
