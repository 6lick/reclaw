# External Webhooks

This OpenClaw instance supports receiving webhooks from external services.

## Webhook Endpoints

| Endpoint | Use Case |
|----------|----------|
| `/webhooks/graph` | Microsoft Graph (email, calendar, OneDrive) |
| `/webhooks/agentmail` | Agent Mail notifications |
| `/webhooks/{source}` | Any other service - replace `{source}` with a name |

## Your Webhook URL

```
https://reclaw-production.up.railway.app/webhooks/{source}
```

Replace `{source}` with a descriptive name for the service (e.g., `stripe`, `github`, `polymarket`).

## How It Works

1. External service POSTs JSON to `/webhooks/{source}`
2. The wrapper formats the payload and forwards it to the agent via `/hooks/agent`
3. The agent wakes up with a message describing the webhook event
4. You return `202 Accepted` immediately (fire-and-forget)

## Microsoft Graph Special Handling

The `/webhooks/graph` endpoint handles Microsoft's validation handshake automatically:
- When Graph sends `?validationToken=xxx`, it returns the token as plain text
- All other requests are forwarded to the agent

## Registering Webhooks with External Services

### Example: Stripe

```bash
stripe webhooks create \
  --url "https://reclaw-production.up.railway.app/webhooks/stripe" \
  --events payment_intent.succeeded,payment_intent.failed
```

### Example: GitHub

In your repo settings → Webhooks → Add webhook:
- Payload URL: `https://reclaw-production.up.railway.app/webhooks/github`
- Content type: `application/json`
- Select events you want to receive

### Example: Generic Service

Most services just need:
1. Your webhook URL: `https://reclaw-production.up.railway.app/webhooks/{name}`
2. Content-Type: `application/json`
3. HTTP method: POST

## Testing Webhooks

```bash
# Test that the endpoint is reachable
curl -X POST https://reclaw-production.up.railway.app/webhooks/test \
  -H "Content-Type: application/json" \
  -d '{"event": "test", "data": "hello"}'
# Should return: Accepted

# Test Graph validation handshake
curl "https://reclaw-production.up.railway.app/webhooks/graph?validationToken=abc123"
# Should return: abc123
```

## What the Agent Receives

When a webhook arrives, the agent receives a message like:

```
Webhook from stripe: {"event":"payment_intent.succeeded","data":{"amount":1000}}
```

For Microsoft Graph:
```
Microsoft Graph notification: changeType=created resource=users/me/messages/123
```

## Session Keys

Each webhook source gets its own session key:
- `hook:graph:email` - Microsoft Graph
- `hook:agentmail:{timestamp}` - Agent Mail
- `hook:{source}:{timestamp}` - Generic webhooks

This allows you to have separate conversation threads per webhook source.
