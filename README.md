# Sendly ↔ Hermes Agent (SMS platform adapter)

A drop-in [Hermes Agent](https://github.com/NousResearch/hermes-agent) **platform
adapter** that lets people **text your Hermes agent over SMS** through
[Sendly](https://sendly.live) — and lets the agent **text back**. Sendly handles
carrier verification for you (instant international, toll-free for US/Canada), so
there's no approval gauntlet to send.

It appears as **"SMS (Sendly)"** in `hermes gateway setup`, alongside the
built-in Telegram / Discord / Twilio channels — no core Hermes changes.

## Install

```bash
hermes plugins install SendlyHQ/hermes-sendly --enable
pip install aiohttp
```

`--enable` adds the plugin to `plugins.enabled` (third-party platform adapters
are opt-in). Then configure it (`hermes gateway setup` → **SMS (Sendly)**, or set
the env vars below) and create a Sendly webhook for `message.received` pointed at
this adapter's public URL.

```bash
SENDLY_API_KEY=sk_live_v1_your_key
SENDLY_PHONE_NUMBER=+1...           # a Sendly NUMBER (needed for two-way; see below)
SENDLY_WEBHOOK_SECRET=whsec_...     # signing secret of your Sendly webhook
SENDLY_ALLOWED_USERS=+15551234567   # recommended allowlist (deny-all by default)
```

Start `hermes gateway`, text your number, done. Running locally? Tunnel the
listener so Sendly can reach it: `cloudflared tunnel --url http://localhost:8080`.

## How it works

| Direction | Mechanism |
|---|---|
| Inbound (texter → agent) | Sendly POSTs a signed `message.received` webhook → the adapter verifies `X-Sendly-Signature` (HMAC-SHA256 over `{timestamp}.{body}`, 5-min replay window) → builds a `MessageEvent` → hands it to the agent. |
| Outbound (agent → texter) | `send()` → `POST /api/v1/messages {to, text, from}` with `Authorization: Bearer <key>`. |

## Pick a two-way-capable sender

For people to **text the agent and get replies**, `SENDLY_PHONE_NUMBER` must be a
sender that can **receive**:

- **US / Canada → a toll-free number.** Toll-free is two-way (send + receive); it
  requires carrier verification, which **Sendly handles for you**. This is the
  setup for a US agent.
- **International → a local two-way number** where the country supports it.
- **Alphanumeric sender IDs** (a brand name) are **send-only** — great for
  one-way notifications, but recipients can't reply, so they can't drive a
  two-way agent.

## Configuration

See [`docs/sms-sendly.md`](docs/sms-sendly.md) for the full setup page and the
complete environment-variable reference, and
[`docs/run-hermes-on-fly-with-sendly.md`](docs/run-hermes-on-fly-with-sendly.md)
for an end-to-end deploy on Fly.io.

## Optional: the Sendly SMS skill

A companion Hermes **skill** teaches the agent to use Sendly SMS and OTP well —
when to use OTP vs plain SMS, how Sendly's carrier verification works, and how to
pick a two-way-capable sender:

```bash
hermes skills install SendlyHQ/hermes-sendly/sendly-sms
```

## Notes

- Built against Hermes' documented platform-adapter plugin API
  (`BasePlatformAdapter` + `register()`), modelled on the reference webhook
  adapters (line / wecom / bluebubbles).
- A `sk_test_` key sandboxes everything (no real SMS, no credits) — test first,
  then swap to a `sk_live_` key once verified at
  [sendly.live/verify](https://sendly.live/verify).

MIT — same license as Hermes.
