# WhatsApp AI Ordering Agent (Claude + n8n + WhatsApp Cloud API)

A production-shaped WhatsApp agent that takes real orders: it knows the menu, handles
customizations, calculates delivery, confirms the order, and hands the kitchen a
**structured JSON order** — not a transcript someone has to read and retype.

Built end to end on the **official WhatsApp Cloud API** (no unofficial libraries, no
browser automation, nothing that gets a number banned).

**▶ Watch it run — real, uncut session:** https://youtu.be/wTAgYEjggIE

The left side is a customer's phone. The right side is n8n executing the agent live.

---

## What it actually does

| | |
|---|---|
| **Answers** | menu, prices, ingredients, opening hours, delivery area |
| **Takes orders** | item by item, with customizations ("no onion"), quantities, delivery vs pickup |
| **Calculates** | subtotal, delivery fee, total |
| **Confirms** | address and payment method before closing |
| **Delivers** | a structured order to the kitchen's WhatsApp, plus JSON your POS can consume |
| **Escalates** | complaints, coupons, group bookings → tagged for a human, no guessing |

The agent never invents an item or a price: everything it can sell lives in the system
prompt, and anything outside it is a handoff.

## Architecture

```
WhatsApp  ──▶  Cloud API webhook  ──▶  n8n
                                        │
                     ┌──────────────────┼──────────────────┐
                     ▼                  ▼                  ▼
             GET  verify token    parse message    per-customer memory
                                        │
                                        ▼
                          Claude (claude-haiku-4-5) + system prompt
                                        │
                        ┌───────────────┴───────────────┐
                        ▼                               ▼
              reply to the customer            [[PEDIDO]] JSON  ──▶ kitchen
                                                [[HUMANO]]      ──▶ human handoff
```

**The control-block trick.** The model writes one message. Everything the customer should
see is plain text; everything the *system* needs is wrapped in sentinels the customer
never sees:

```
Perfect, your order is confirmed! Estimated delivery: 40–60 min.
[[PEDIDO]]{"itens":[{"nome":"Combo Spider Man","qtd":1,"preco_unit":60.0}],
"subtotal":60.0,"taxa_entrega":8.0,"total":68.0,"modalidade":"entrega",
"endereco":"...","pagamento":"PIX","observacoes":"sem cebola"}[[/PEDIDO]]
```

A Code node strips the block, parses the JSON, and routes it. `[[HUMANO]]` works the same
way for escalation. This keeps the LLM doing one job (talking) while the workflow stays
deterministic about money and routing.

## Why n8n and not a plain Node service

Because the person who owns the restaurant has to be able to *see* it. Every conversation
is a visible execution: what came in, what Claude answered, what went to the kitchen, and
where it failed. Changing the menu is editing one node, not a deploy. The same agent in
300 lines of Express is cheaper to run and impossible for the owner to debug at 9pm on a
Friday.

If a client prefers plain Node/NestJS, the same design ports directly — the interesting
parts (prompt, control blocks, normalization) are framework-agnostic.

## Details that only show up in production

- **Brazilian 9th digit.** Meta delivers `wa_id` without the mobile 9 (`5564XXXXYYYY`), but
  the send endpoint needs it. Reply to the raw id and you get `131030 — recipient not in
  allowed list`, from a number that just messaged you. `normBR()` in the parse node fixes it.
- **Non-text messages.** Audio, images and stickers reach the webhook too. They're detected
  and answered politely instead of crashing the flow.
- **Status webhooks.** `sent`/`delivered`/`read`/`failed` events arrive on the same URL and
  are filtered out, so a delivery receipt never triggers a reply.
- **Per-customer memory** keyed by phone number, so the conversation survives across messages.

## Repository

```
workflow/whatsapp-agent.n8n.json   importable n8n workflow (16 nodes, placeholders only)
prompts/system-prompt.pt-BR.md     the full agent prompt from the live case
prompts/system-prompt.template.md  blank template to adapt to any business
docs/setup.md                      install: Meta → n8n → first message
docs/meta-gotchas.md               the errors that stop most setups, and the fix for each
```

No credentials, tokens, phone numbers or account ids are in this repository, by design.

## Setup

See [`docs/setup.md`](docs/setup.md). Short version: import the workflow, fill the two
Config nodes, add a Header Auth credential with your Meta token and an Anthropic
credential, point the Meta webhook at your n8n URL, subscribe the `messages` field.

## Cost

Roughly **US$ 0.002 per conversation** with `claude-haiku-4-5` on a menu this size —
about US$ 2 per thousand conversations. n8n self-hosted is free; WhatsApp charges nothing
for replies inside the 24-hour customer service window.

## Case study

Built for **Heróis Super Burger**, a hero-themed burger place in Caldas Novas, Brazil.
The video above is a real session on the live agent, not a mockup.

## License

MIT — see [LICENSE](LICENSE). The menu and brand in the case-study prompt belong to their
owner and are there as an example; replace them with your own.

---

Built by **Alexandre Toulios** — full-stack developer (React/Next/TypeScript, Node/NestJS,
PostgreSQL) working on AI agents and automation.
[Upwork](https://www.upwork.com/freelancers/~0104de19b352c1ba9c) ·
[LinkedIn](https://www.linkedin.com/in/alexandre-toulios)
