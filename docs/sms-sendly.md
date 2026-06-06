# SMS Setup (Sendly)

Hermes connects to SMS through [Sendly](https://sendly.live). People text your
Sendly number and get AI responses back — the same conversational experience as
Telegram or Discord, but over standard text messages.

The difference from the Twilio gateway: **Sendly handles carrier verification
for you** — international is verified in minutes and US/Canada toll-free is
submitted for you — so there's no approval gauntlet before you can send. You
also pay in simple credits.

This adapter ships as a **drop-in plugin** — no core Hermes changes. Drop the
`sendly/` plugin into `~/.hermes/plugins/` and it appears in `hermes gateway
setup` like any built-in platform.

## Prerequisites

- A [Sendly account](https://sendly.live) and an API key (test key = sandbox,
  live key = real SMS)
- A **two-way-capable sender**: for people to text the agent and get replies you
  need a number that can **receive**. In the **US/Canada** that's a **toll-free
  number** (two-way; Sendly handles the carrier verification for you).
  Alphanumeric sender IDs are **send-only** (one-way notifications, no replies).
- A publicly accessible server — Sendly sends a webhook to your server when an
  SMS arrives
- `aiohttp` — `pip install aiohttp`

## Step 1: Install the plugin

```bash
git clone https://github.com/SendlyHQ/hermes-sendly ~/.hermes/plugins/sendly
# directory layout: ~/.hermes/plugins/sendly/{PLUGIN.yaml,adapter.py}
```

## Step 2: Create a Sendly webhook

In the Sendly dashboard → **Webhooks**, create a webhook:

- **URL**: the public URL of this adapter's listener, path `/webhooks/sendly`
  (e.g. `https://your-server:8080/webhooks/sendly`)
- **Events**: enable `message.received`
- Copy the webhook's **signing secret** — the adapter uses it to verify
  `X-Sendly-Signature`.

## Step 3: Configure Hermes

**Interactive setup (recommended)**

```bash
hermes gateway setup
```

Select **SMS (Sendly)** from the platform list. The wizard prompts for your
credentials.

**Manual setup** — add to `~/.hermes/.env`:

```bash
SENDLY_API_KEY=sk_live_v1_your_key_here
SENDLY_PHONE_NUMBER=+15551234567        # your Sendly number or sender ID
SENDLY_WEBHOOK_SECRET=whsec_...          # the webhook's signing secret

# Security: restrict to specific phone numbers (recommended)
SENDLY_ALLOWED_USERS=+15559876543,+15551112222

# Optional: home channel for cron / proactive delivery
SENDLY_HOME_CHANNEL=+15559876543
```

## Step 4: Expose your webhook

If you're running Hermes locally, tunnel the listener (default port 8080):

```bash
cloudflared tunnel --url http://localhost:8080
# or: ngrok http 8080
```

Set the resulting public URL (with `/webhooks/sendly`) as the Sendly webhook URL
from Step 2.

## Step 5: Start the gateway

```bash
hermes gateway
```

You should see:

```
[sendly] webhook server listening on 0.0.0.0:8080/webhooks/sendly, from: +1555***4567
```

Text your Sendly number — Hermes responds via SMS.

> **Test first.** With a `sk_test_` key, sends are simulated (no real SMS, no
> credits). Swap to a `sk_live_` key once your account is verified at
> [sendly.live/verify](https://sendly.live/verify).

## Environment Variables

| Variable | Required | Description |
|---|---|---|
| `SENDLY_API_KEY` | Yes | Sendly API key (`sk_test_` = sandbox, `sk_live_` = real SMS) |
| `SENDLY_PHONE_NUMBER` | Yes | Your Sendly sending number / sender ID |
| `SENDLY_WEBHOOK_SECRET` | Yes* | Signing secret of the Sendly webhook (verifies `X-Sendly-Signature`) |
| `SENDLY_WEBHOOK_PORT` | No | Listener port (default: 8080) |
| `SENDLY_WEBHOOK_HOST` | No | Bind address (default: 0.0.0.0) |
| `SENDLY_BASE_URL` | No | Override the Sendly API base URL (default: `https://sendly.live`) |
| `SENDLY_INSECURE_NO_SIGNATURE` | No | `true` disables signature checks — **local dev only** |
| `SENDLY_ALLOWED_USERS` | No | Comma-separated E.164 numbers allowed to chat |
| `SENDLY_ALLOW_ALL_USERS` | No | `true` allows anyone (not recommended) |
| `SENDLY_HOME_CHANNEL` | No | Number for cron / proactive delivery |

\* Not required if `SENDLY_INSECURE_NO_SIGNATURE=true` (local dev only).

## SMS-Specific Behavior

- **Plain text only** — markdown is stripped (SMS renders it literally).
- **Long replies are split** — Hermes chunks at natural boundaries; Sendly bills
  per segment.
- **Echo prevention** — messages from your own Sendly number are ignored.

## Security

**Webhook signature validation.** Hermes verifies inbound webhooks with the
`X-Sendly-Signature` header — HMAC-SHA256 over `{timestamp}.{body}` keyed by the
webhook's signing secret, with the timestamp taken from `X-Sendly-Timestamp` and
a 5-minute replay window. Set `SENDLY_WEBHOOK_SECRET` to the webhook's signing
secret. For local dev without a public URL you can set
`SENDLY_INSECURE_NO_SIGNATURE=true` — **never in production.**

**User allowlists.** The gateway denies all users by default. Restrict with
`SENDLY_ALLOWED_USERS`, or `SENDLY_ALLOW_ALL_USERS=true` (not recommended for
bots with terminal access).

> SMS has no built-in encryption. For sensitive use cases, prefer Signal or
> Telegram.

## Troubleshooting

**Messages not arriving**
- Confirm the Sendly webhook URL is correct, public, and has `message.received` enabled.
- Verify `SENDLY_WEBHOOK_SECRET` matches the webhook's signing secret.
- Check the sender is in `SENDLY_ALLOWED_USERS` (or `SENDLY_ALLOW_ALL_USERS=true`).
- Check Sendly dashboard → Webhooks → Deliveries for failures.

**Replies not sending**
- Confirm `SENDLY_PHONE_NUMBER` is set and your account is verified (live key).
- A live key on an unverified account simulates instead of sending — verify at
  [sendly.live/verify](https://sendly.live/verify).
- Check the gateway logs for Sendly API errors (e.g. `insufficient_credits`).

**Webhook port conflicts** — change `SENDLY_WEBHOOK_PORT` and update the webhook URL in Sendly to match.
