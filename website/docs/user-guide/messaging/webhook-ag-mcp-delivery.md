---
sidebar_position: 14
title: "Webhook delivery via AG MCP"
description: "Route webhook responses to WhatsApp Access Gateway contacts through MCP"
---

# Webhook delivery via AG MCP

A webhook route can deliver the final agent response through the WhatsApp Access Gateway (AG) MCP server by setting:

```yaml
deliver: ag
```

This is different from a normal gateway platform such as `telegram` or `discord`: Hermes calls the AG MCP tools directly.

## Requirements

- The AG MCP server must be configured and reachable by Hermes.
- The MCP server must expose:
  - `list_proactive_contacts`
  - `request_proactive_message`
- The incoming webhook must identify the sender with a contact ID, phone number, or email address.
- The sender must exist in AG's eligible proactive-contact list.

Hermes resolves the sender on every delivery. It does not use a fixed contact ID unless one is explicitly configured in `deliver_extra.contact_id`.

## Dashboard

On `/webhooks`, click **New subscription** and select **AG (MCP)** in **Deliver to**. The option is available for both agent responses and direct delivery. For the dynamic mode, leave **Deliver only** unchecked so Hermes can process the incoming message and send the final answer through AG.

Use a prompt containing the incoming fields, for example:

```text
Origin: {source}
Sender: {sender}
Message: {message}

Answer the sender safely and helpfully.
```

After creating or changing a subscription, restart the gateway if the Dashboard indicates that a restart is required.

## Route configuration

A dynamic subscription can use this shape:

```json
{
  "ag": {
    "description": "Access Gateway",
    "events": ["message"],
    "secret": "replace-with-a-secret",
    "prompt": "You received an external message.\n\nOrigin: {source}\nSender: {sender}\nMessage: {message}\n\nProcess it safely and answer helpfully.",
    "skills": [],
    "deliver": "ag"
  }
}
```

For a static route in `config.yaml`, use the same `deliver` and `prompt` fields under `platforms.webhook.extra.routes`.

## Sender matching

Hermes checks the following webhook fields:

```text
contact.id
contact_id
sender_id
sender
from
phone
phone_number
email
author
```

When `sender` is an object, its `id`, `contact_id`, `phone`, `phone_number`, `email`, and `sender_id` fields are also checked.

Phone numbers are normalized before comparison. For example, these values match:

```text
+5511999999999
+55 11 99999-9999
```

If no eligible AG contact matches, Hermes does not send a message and records the reason in the gateway log.

## Message flow

1. The webhook signature is validated.
2. The event is checked against the route's `events` list.
3. The prompt is rendered from the payload.
4. Hermes runs the agent.
5. Hermes calls `list_proactive_contacts` through the AG MCP server.
6. The sender is matched to an eligible contact.
7. Hermes calls `request_proactive_message` with the final response.
8. The delivery ID becomes the MCP idempotency key, preventing duplicate sends on webhook retries.

## Example request

The route accepts the generic `message` event:

```json
{
  "event_type": "message",
  "source": "test",
  "sender": {
    "phone_number": "+5511999999999"
  },
  "message": "Hello from the external channel"
}
```

For the generic HMAC V2 format, sign the exact string `<timestamp>.<body>` with the route secret:

```bash
SECRET='replace-with-route-secret'
TIMESTAMP=$(date +%s)
BODY='{"event_type":"message","source":"test","sender":{"phone_number":"+5511999999999"},"message":"Hello from the external channel"}'
SIGNATURE=$(printf '%s.%s' "$TIMESTAMP" "$BODY" | openssl dgst -sha256 -hmac "$SECRET" -hex | sed 's/^.* //')

curl -i -X POST 'http://localhost:8644/webhooks/ag' \
  -H 'Content-Type: application/json' \
  -H "X-Webhook-Timestamp: $TIMESTAMP" \
  -H "X-Webhook-Signature-V2: $SIGNATURE" \
  --data "$BODY"
```

The endpoint normally returns `202 Accepted` because agent processing and delivery run asynchronously.

## Explicit contact override

A route may bypass dynamic matching with:

```json
"deliver": "ag",
"deliver_extra": {
  "contact_id": "contact-uuid"
}
```

Use this only when every message on the route must go to the same AG contact. Dynamic matching is safer for multi-channel ingress.

## Troubleshooting

Check the gateway log:

```bash
grep -iE 'AG delivery|webhook.*route=ag|MCP tool' ~/.hermes/logs/gateway.log ~/.hermes/logs/agent.log | tail -50
```

Common errors:

- `could not find sender/contact fields`: add `sender`, `phone_number`, `email`, or `contact_id` to the payload.
- `could not map webhook sender`: the sender is not in AG's eligible contact list.
- `MCP tool ... is not registered`: reconnect AG MCP or restart Hermes.
- `request_proactive_message` returns an approval result: the AG governance policy requires approval before the contact can be messaged.
