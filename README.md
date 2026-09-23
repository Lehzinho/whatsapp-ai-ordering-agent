# WhatsApp AI Ordering Agent (Claude + n8n + WhatsApp Cloud API)

WhatsApp bot that takes food orders for a burger place. It knows the menu, handles
changes like "no onion", calculates delivery, confirms the order and sends the kitchen a
structured order (JSON) instead of a chat transcript someone has to read and retype.

It runs on the official WhatsApp Cloud API. No unofficial libraries or browser automation,
so the number doesn't get banned.

Video of a real session (uncut): https://youtu.be/wTAgYEjggIE

Left side is the customer's phone, right side is n8n running the flow.

## What it does

- Answers questions about the menu, prices, ingredients, opening hours and delivery area
- Takes the order item by item, with quantities, changes and delivery or pickup
- Calculates subtotal, delivery fee and total
- Confirms address and payment method before closing
- Sends the finished order to the kitchen's WhatsApp, plus the JSON if you want to feed a POS
- Anything it shouldn't decide (complaints, coupons, big group orders) goes to a person

The bot can only sell what is in the system prompt. If something isn't there, it hands off
to a human instead of making up an item or a price.

## How it works

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

The model writes a single message. The part for the customer is plain text, and the part
for the system goes inside markers that get removed before sending:

```
Perfect, your order is confirmed! Estimated delivery: 40-60 min.
[[PEDIDO]]{"itens":[{"nome":"Combo Spider Man","qtd":1,"preco_unit":60.0}],
"subtotal":60.0,"taxa_entrega":8.0,"total":68.0,"modalidade":"entrega",
"endereco":"...","pagamento":"PIX","observacoes":"sem cebola"}[[/PEDIDO]]
```

A Code node cuts that block out, parses the JSON and sends it to the kitchen. `[[HUMANO]]`
does the same for handoff. The LLM only talks; the money and the routing are plain code.

## Why n8n and not a Node service

The restaurant owner needs to see what's going on. In n8n every conversation is an
execution you can open: what came in, what Claude answered, what went to the kitchen and
where it broke. Changing the menu means editing one node, no deploy. The same thing in
Express would be cheaper to host, but the owner couldn't debug it on a Friday night.

The design isn't tied to n8n. The prompt, the control blocks and the number fix below
work the same way in Node/NestJS.

## Things I only found in production

- Brazil's 9th digit: Meta sends the `wa_id` without the mobile 9 (`5564XXXXYYYY`), but
  the send endpoint needs it. If you reply to the raw id you get
  `131030 recipient not in allowed list` from a number that just messaged you.
  `normBR()` in the parse node fixes it.
- Audio, images and stickers also hit the webhook. The bot answers asking for text instead
  of breaking.
- `sent` / `delivered` / `read` / `failed` events come to the same URL. They're filtered
  out, otherwise the bot replies to its own delivery receipts.
- Memory is per customer, keyed by phone number, so the conversation keeps its context
  between messages.

## Repository

```
workflow/whatsapp-agent.n8n.json   n8n workflow to import (16 nodes, placeholders only)
prompts/system-prompt.pt-BR.md     full prompt used in the restaurant
prompts/system-prompt.template.md  empty template for another business
docs/setup.md                      Meta -> n8n -> first message
docs/meta-gotchas.md               errors that block most setups and how to fix them
```

There are no tokens, phone numbers or account ids in the repo.

## Setup

See [docs/setup.md](docs/setup.md). In short: import the workflow, fill the two Config
nodes, create a Header Auth credential with the Meta token and an Anthropic credential,
point the Meta webhook to your n8n URL and subscribe the `messages` field.

## Cost

About US$ 0.002 per conversation with `claude-haiku-4-5` for a menu this size, so around
US$ 2 per thousand conversations. Self-hosted n8n is free and WhatsApp doesn't charge for
replies inside the 24-hour customer service window.

## Case study

Running at Heróis Super Burger, a hero-themed burger place in Caldas Novas, Brazil. The
video is a real session with the live bot.

## License

MIT, see [LICENSE](LICENSE). The menu and brand in the example prompt belong to the
restaurant; replace them with your own.

---

Alexandre Toulios, full-stack developer (React/Next/TypeScript, Node/NestJS, PostgreSQL).
[Upwork](https://www.upwork.com/freelancers/~0104de19b352c1ba9c) ·
[LinkedIn](https://www.linkedin.com/in/alexandre-toulios)
