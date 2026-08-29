# Setup

From zero to a bot replying on your own number.

## 1. Meta side

1. **Business Portfolio** — [business.facebook.com](https://business.facebook.com) →
   Settings → Business info. Fill legal name, **country** and full address before anything
   else; an empty country causes silent delivery failures (see
   [meta-gotchas.md](meta-gotchas.md#130497--business-account-is-restricted-from-messaging-users-in-this-country)).
2. **App** — [developers.facebook.com](https://developers.facebook.com) → Create app →
   use case *"Connect with customers on WhatsApp"* → link it to your portfolio.
3. **Phone number** — Step 2 (Production setup) → *Add phone number*.
   The number must have **no active WhatsApp account**. Pick **voice call** verification if
   the SMS doesn't arrive. Save the 6-digit PIN you set.
4. **Token** — the 24h test token is fine to try. For production create a **System User**
   with the WABA and app assigned, and generate a permanent token.

Note your **Phone Number ID** and **WABA ID** from the API Setup screen.

## 2. n8n side

1. Import `workflow/whatsapp-agent.n8n.json`.
2. Open **Config** and **Config2** (both, they feed different branches) and set:
   - `verifyToken` — any string you invent; it must match what you type in Meta
   - `phoneNumberId` — from API Setup
   - `restaurantNumber` — the number that receives orders, with country code
   - `graphVersion` — `v25.0`
3. Create a **Header Auth** credential:
   - Name: `Authorization`
   - Value: `Bearer YOUR_META_TOKEN` (keep the word `Bearer` and the space)
   - Select it in both HTTP Request nodes.
4. Create an **Anthropic** credential and select it in the *Anthropic Claude* node.
5. Replace the system prompt in the *Agente Robin* node with your own
   (see [`prompts/system-prompt.template.md`](../prompts/system-prompt.template.md)).
6. **Publish** the workflow. The webhook is only live when published.

## 3. Connect the two

n8n has to be reachable from the internet. For a quick test:

```bash
cloudflared tunnel --url http://localhost:5678
```

(The free tunnel URL changes on every restart — fine for testing, not for production. Use a
VPS or n8n Cloud for a real deployment.)

In Meta → WhatsApp → Configuration → Webhook:

- **Callback URL**: `https://your-tunnel/webhook/whatsapp`
- **Verify token**: the same `verifyToken` from Config
- Subscribe the **`messages`** field

Then subscribe the WABA to the app — the step most people miss:

```http
POST https://graph.facebook.com/v25.0/{WABA_ID}/subscribed_apps
Authorization: Bearer {TOKEN}
```

## 4. Test

Add your own number to the allowed list (API Setup screen), send "hi" from your phone, and
watch the execution appear in n8n. If nothing arrives, follow the
[diagnostic order](meta-gotchas.md#quick-diagnostic-order).
