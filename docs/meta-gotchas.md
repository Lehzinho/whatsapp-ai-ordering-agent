# WhatsApp Cloud API: errors that block a setup

I ran into all of these while putting a real number in production. If your integration is
stuck, look for the symptom you're seeing.

---

## "My webhook is verified, but I never receive a message"

This is the most common one and it's easy to miss in the docs.

Creating the app and passing webhook verification is not enough. The WhatsApp Business
Account has to be subscribed to your app:

```http
POST https://graph.facebook.com/v25.0/{WABA_ID}/subscribed_apps
Authorization: Bearer {TOKEN}
```

Expected: `{"success": true}`. Until you run it, the verification handshake (`GET` with
`hub.challenge`) succeeds and no `POST` ever arrives. Nothing in the UI tells you.

Check what is subscribed with `GET /{WABA_ID}/subscribed_apps`.

---

## `130497: Business account is restricted from messaging users in this country`

The API accepts your `POST /messages` with a `200` and a message id, then a status
webhook arrives seconds later with `status: failed` and this error. You won't see it if you
only log the send response.

Cause: Meta sees a mismatch between where the business is and where the recipient is.
Typically an unverified portfolio with no country/address using a US test number to message
another country.

What fixed it, cheapest first:

1. Fill Business settings → Business info: legal name, country, full address.
2. Register a real phone number from the target country instead of the US test number.
3. Complete business verification (2–10 business days).

The test number is only for development. Don't count on it to deliver to another country.

---

## `131030: Recipient phone number not in allowed list`

Two different causes.

Development mode: The test number only messages numbers you explicitly add to the
allowed list. Add the recipient in the API Setup screen.

Brazil's 9th digit: Meta sends the inbound `wa_id` as `5564XXXXYYYY` (12 digits, no
mobile 9), but the send endpoint expects the number as dialed, with the 9. Replying to the
raw `wa_id` produces `131030` for a number that literally just messaged you.

```js
function normBR(n) {
  n = String(n || '')
  if (n.startsWith('55') && n.length === 12) return n.slice(0, 4) + '9' + n.slice(4)
  return n
}
```

Keep the raw value too, and pick one form for memory keys and dedupe.

---

## The SMS verification code never arrives

Happens a lot in Brazil. Requesting it again usually doesn't help.

Choose "Voice call" instead of SMS. A robot calls and reads the 6 digits. It works on
numbers where SMS silently fails, and it's available on the same screen.

While registering you also set a 6-digit 2FA PIN. Write it down: you need it to
re-register the number later and there's no way to recover it yourself.

---

## `This phone number is already registered on a WhatsApp account`

The Cloud API only accepts numbers with no active WhatsApp account. Just not using
WhatsApp on that number anymore isn't enough, the account still exists on Meta's side.

Free it: install WhatsApp on that number, register (voice call works here too), then
Settings → Account → Delete my account. Wait ~3 minutes for propagation and register
again in Cloud API.

---

## `401 / code 190: invalid OAuth access token`

Three different causes:

- The test token expires in 24 hours. Fine for a first test, not for production.
  Create a System User in Business Settings, assign the WABA and the app, and generate a
  token with no expiration.
- The `Bearer ` prefix. The header value must be `Bearer EAAxxxx…`, not the bare token.
- Workspace-scoped API keys. Some Anthropic keys are scoped to a workspace and require an
  `anthropic-workspace-id` header. If your HTTP node returns "Bad Request" mentioning the
  workspace, generate a non-workspace key instead.

---

## Status events answering themselves

`sent`, `delivered`, `read` and `failed` arrive on the same webhook URL as messages.
If your parser doesn't distinguish them, delivery receipts get treated as inbound text and
the bot answers itself in a loop.

Read `entry[].changes[].value`: real messages are under `messages[]`, receipts under
`statuses[]`. Ignore anything without `messages[]`.

---

## Quick diagnostic order

When something doesn't work I check in this order, cheapest first:

1. Is the WABA subscribed to the app? (`GET /{WABA}/subscribed_apps`)
2. Is the webhook receiving `POST`s at all? (n8n executions / your logs)
3. Does the send call return `200` with a message id?
4. Does a status webhook arrive right after, and what does it say? (most answers are here)
5. Is the token expired, or missing the `Bearer` prefix?
6. Does the business have country and address filled in?
