# Run Hermes Agent on Fly.io with Sendly SMS

Run a [Hermes Agent](https://github.com/NousResearch/hermes-agent) on a Fly
Machine and give it **two-way SMS through Sendly** — people text a Sendly number,
the agent replies. Sendly handles carrier verification for you, so there's no
Twilio approval gauntlet.

This builds on the base [Run Hermes Agent on Fly.io](https://fly.io/docs/) guide
and adds the Sendly adapter. The **one important difference**: the base Hermes
gateway is outbound-only (no public port), but two-way SMS needs Sendly to reach
the adapter's webhook — so we expose **one** public, signature-verified endpoint.

## Prerequisites

- `flyctl` installed and a Fly.io account
- An LLM API key for Hermes (Anthropic / OpenAI / Gemini / OpenRouter)
- A [Sendly](https://sendly.live) account + API key
- For **two-way** (people text the agent and it replies): a sender that can
  **receive**. In the **US/Canada** that's a **toll-free number** (two-way;
  Sendly handles the carrier verification for you). Alphanumeric sender IDs are
  **send-only** and can't drive a two-way agent.

## 1. Create the app and volume

```bash
fly apps create <your-hermes-app>
fly volumes create data -a <your-hermes-app> --region <region> --size 3
```

## 2. fly.toml — expose the webhook port

Same as the base guide, **plus an `http_service`** so Sendly can deliver inbound
SMS to the adapter (internal port 8080). The dashboard (9119) stays private.

```toml
app = "<your-hermes-app>"
primary_region = "<region>"
machine_config = "machine_config.json"

[build]
  image = "nousresearch/hermes-agent:latest"

[[mounts]]
  source = "data"
  destination = "/opt/data"

# Public, signature-verified endpoint for Sendly's message.received webhook.
[http_service]
  internal_port = 8080
  force_https = true
  auto_stop_machines = false   # keep the gateway warm; it's stateful
  min_machines_running = 1

[[vm]]
  memory = "4gb"
  cpus = 2
```

```json
{
  "containers": [
    { "name": "hermes", "image": "nousresearch/hermes-agent:latest", "cmd": ["gateway", "run"] }
  ]
}
```

> Exposing 8080 is safe: the adapter only serves `POST /webhooks/sendly` and
> rejects anything without a valid `X-Sendly-Signature`. The dashboard (9119),
> which exposes API keys, is **not** published — reach it via `fly proxy`.

## 3. Deploy + install the Sendly plugin

```bash
fly deploy -a <your-hermes-app> --ha=false
```

Hermes stores state on the volume at `/opt/data` (config, `.env`, sessions,
skills, **plugins**). Install the adapter there so it survives deploys, and
symlink the binary onto PATH:

```bash
fly ssh console -a <your-hermes-app> -C \
  "ln -sf /opt/hermes/.venv/bin/hermes /usr/local/bin/hermes"

fly ssh console -a <your-hermes-app>
# inside the machine:
git clone https://github.com/SendlyHQ/hermes-sendly /opt/data/plugins/sendly
/opt/hermes/.venv/bin/pip install aiohttp
```

## 4. Configure Sendly

In the same SSH session (or via `hermes gateway setup` → **SMS (Sendly)**), add
to `/opt/data/.env`:

```bash
SENDLY_API_KEY=sk_live_v1_your_key
SENDLY_PHONE_NUMBER=+1...            # a Sendly NUMBER (required for two-way)
SENDLY_WEBHOOK_SECRET=whsec_...      # signing secret from the webhook you create below
SENDLY_WEBHOOK_HOST=0.0.0.0
SENDLY_WEBHOOK_PORT=8080
SENDLY_ALLOWED_USERS=+447500938194   # recommended: who may chat
```

## 5. Create the Sendly webhook

In the Sendly dashboard → **Webhooks**, create a webhook:

- **URL**: `https://<your-hermes-app>.fly.dev/webhooks/sendly`
- **Event**: `message.received`
- Copy the **signing secret** → that's your `SENDLY_WEBHOOK_SECRET` above.

## 6. Restart + test

```bash
fly machine restart <machine-id> -a <your-hermes-app>
fly logs -a <your-hermes-app>   # look for: [sendly] webhook server listening on 0.0.0.0:8080
```

**Two-way:** text your Sendly number → Hermes replies via SMS.

**Outbound only (smoke test):** confirm sending works without the agent —

```bash
curl -sS -X POST https://sendly.live/api/v1/messages \
  -H "Authorization: Bearer $SENDLY_API_KEY" -H "Content-Type: application/json" \
  -d '{"to":"+447500938194","text":"Hello from Hermes x Sendly","messageType":"transactional"}'
# -> 201 {"status":"queued",...}
```

## Notes

- **Test key first.** A `sk_test_` key sandboxes everything (no real SMS, no
  credits). Swap to `sk_live_` once your account is verified at
  [sendly.live/verify](https://sendly.live/verify) — a live key on an unverified
  account simulates instead of sending.
- **Numbers vs. sender IDs.** Alphanumeric sender IDs are send-only; for the
  agent to *receive* texts, use a real Sendly number.
- **Upgrades** re-pull `nousresearch/hermes-agent:latest`; re-run the symlink
  step. The plugin lives on the volume, so it persists.
