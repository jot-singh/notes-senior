# Webhooks & Event Callbacks

## What You'll Learn
How to implement server-to-client push notifications via webhooks. You'll understand webhook design patterns, security considerations, retry strategies, and how to build reliable event delivery systems.

## Why This Matters
Polling is inefficient: clients waste resources checking for updates that may not exist. Webhooks invert the model—servers push events when they happen, enabling real-time integrations. Payment processors (Stripe), version control (GitHub), and CI/CD systems (Jenkins) all use webhooks. Understanding webhook design is essential for event-driven architectures and integrations. Poor webhook implementation causes missed events, security vulnerabilities, and integration failures.

## Overview

Webhooks enable servers to push events to clients via HTTP POST to client-provided URLs. Instead of:

```
Client polls every 5 seconds:
"Any new orders?"
"Any new orders?"
"Any new orders?"
"Any new orders?"  → "Yes! Order #123"

96% of requests wasted
```

Webhooks push when events occur:
```
Event happens → Server POSTs to client URL immediately
Zero wasted requests
```

## Webhook Registration

### CRUD Endpoints

```
POST   /webhooks          Create webhook subscription
GET    /webhooks          List webhook subscriptions
GET    /webhooks/{id}     Get webhook details
PATCH  /webhooks/{id}     Update webhook
DELETE /webhooks/{id}     Delete webhook

GET    /webhooks/{id}/deliveries  List delivery attempts
GET    /webhooks/{id}/deliveries/{deliveryId}  Get delivery details
POST   /webhooks/{id}/test  Send test event
```

### Registration Example

**Request**:
```json
POST /webhooks
Authorization: Bearer token

{
  "url": "https://myapp.example.com/webhooks/orders",
  "events": ["order.created", "order.completed", "order.cancelled"],
  "secret": "whsec_abc123...",  // For signature verification
  "active": true,
  "metadata": {
    "environment": "production",
    "owner": "team-billing"
  }
}
```

**Response**:
```json
201 Created
{
  "id": "wh_xyz789",
  "url": "https://myapp.example.com/webhooks/orders",
  "events": ["order.created", "order.completed", "order.cancelled"],
  "active": true,
  "created_at": "2026-01-11T10:30:00Z",
  "updated_at": "2026-01-11T10:30:00Z"
}
```

### URL Validation

**On Registration**: Verify client controls the URL

```http
POST /webhooks
{
  "url": "https://myapp.example.com/webhooks/orders",
  "events": ["order.created"]
}

→ Server sends verification challenge:
POST https://myapp.example.com/webhooks/orders
{
  "type": "url_verification",
  "challenge": "3eZbrw1aBm2rZgRNFdxV2595E9CY3gmdALWMmHkvFXO7tYXAYM8P"
}

← Client must respond with challenge:
200 OK
{
  "challenge": "3eZbrw1aBm2rZgRNFdxV2595E9CY3gmdALWMmHkvFXO7tYXAYM8P"
}

✓ Webhook registered
```

## Event Payload Design

### Minimal Payload (Recommended)

**Advantages**: Small, fast, doesn't leak data if intercepted

```json
POST https://myapp.example.com/webhooks/orders
{
  "id": "evt_abc123",
  "type": "order.completed",
  "timestamp": "2026-01-11T10:30:00Z",
  "data": {
    "orderId": "ord-789"
  }
}
```

Client fetches full details:
```
GET /orders/ord-789 → full order data
```

### Full Payload (Alternative)

**When**: Real-time requirements, avoid round-trip

```json
{
  "id": "evt_abc123",
  "type": "order.completed",
  "timestamp": "2026-01-11T10:30:00Z",
  "data": {
    "order": {
      "id": "ord-789",
      "customerId": "cust-456",
      "status": "completed",
      "total": 99.99,
      "items": [ /* full item details */ ]
    }
  }
}
```

**Trade-off**: Larger payloads, potential data staleness

## Security

### HMAC Signature Verification

**Server Side** (generating signature):
```java
public String generateSignature(String payload, String secret) {
    try {
        Mac hmac = Mac.getInstance("HmacSHA256");
        SecretKeySpec keySpec = new SecretKeySpec(secret.getBytes(), "HmacSHA256");
        hmac.init(keySpec);
        byte[] hash = hmac.doFinal(payload.getBytes());
        return "sha256=" + Hex.encodeHexString(hash);
    } catch (Exception e) {
        throw new RuntimeException("Failed to generate signature", e);
    }
}

// Send webhook
String payload = objectMapper.writeValueAsString(event);
String signature = generateSignature(payload, webhookSecret);

HttpHeaders headers = new HttpHeaders();
headers.set("X-Webhook-Signature", signature);
headers.set("X-Webhook-ID", event.getId());
headers.set("X-Webhook-Timestamp", String.valueOf(event.getTimestamp()));

restTemplate.postForEntity(webhookUrl, payload, String.class);
```

**Client Side** (verifying signature):
```javascript
const crypto = require('crypto');

app.post('/webhooks/orders', (req, res) => {
  const receivedSignature = req.headers['x-webhook-signature'];
  const payload = JSON.stringify(req.body);
  const secret = process.env.WEBHOOK_SECRET;
  
  // Calculate expected signature
  const expectedSignature = 'sha256=' + crypto
    .createHmac('sha256', secret)
    .update(payload)
    .digest('hex');
  
  // Constant-time comparison
  if (!crypto.timingSafeEqual(
    Buffer.from(receivedSignature),
    Buffer.from(expectedSignature)
  )) {
    console.error('Invalid signature');
    return res.status(401).send('Invalid signature');
  }
  
  // Signature valid; process event
  const event = req.body;
  processEvent(event);
  
  res.status(200).send('OK');
});
```

### Replay Attack Prevention

**Include Timestamp**: Reject old events

```javascript
const timestamp = req.headers['x-webhook-timestamp'];
const now = Date.now();
const eventTime = new Date(timestamp).getTime();

// Reject events older than 5 minutes
if (Math.abs(now - eventTime) > 5 * 60 * 1000) {
  return res.status(400).send('Event too old');
}
```

### IP Whitelisting

**Server Side**: Document sender IPs
```
Webhooks originate from:
- 54.12.34.56
- 54.12.34.57
- 54.12.34.58
```

**Client Side**: Firewall rules
```bash
# Only allow webhooks from known IPs
iptables -A INPUT -p tcp --dport 443 -s 54.12.34.56 -j ACCEPT
iptables -A INPUT -p tcp --dport 443 -s 54.12.34.57 -j ACCEPT
iptables -A INPUT -p tcp --dport 443 -j DROP
```

## Reliability & Retry Logic

### At-Least-Once Delivery

Guarantee: Event delivered at least once (possibly duplicates)

**Server retry logic**:
```java
public void deliverWebhook(Webhook webhook, Event event) {
    int maxRetries = 5;
    int attempt = 0;
    
    while (attempt < maxRetries) {
        try {
            HttpResponse response = sendWebhook(webhook.getUrl(), event);
            
            if (response.getStatusCode() >= 200 && response.getStatusCode() < 300) {
                // Success
                logDelivery(webhook, event, "success", attempt);
                return;
            }
            
            // 4xx errors: don't retry (client error)
            if (response.getStatusCode() >= 400 && response.getStatusCode() < 500) {
                logDelivery(webhook, event, "failed", attempt);
                return;
            }
            
            // 5xx errors: retry
            attempt++;
            Thread.sleep(calculateBackoff(attempt));
            
        } catch (Exception e) {
            attempt++;
            if (attempt >= maxRetries) {
                logDelivery(webhook, event, "failed_after_retries", attempt);
                alertOps(webhook, event);
            } else {
                Thread.sleep(calculateBackoff(attempt));
            }
        }
    }
}

private long calculateBackoff(int attempt) {
    // Exponential backoff: 1s, 2s, 4s, 8s, 16s
    return (long) Math.pow(2, attempt) * 1000;
}
```

### Client Idempotency

**Problem**: Retries cause duplicate processing

**Solution**: Track event IDs

```javascript
const processedEvents = new Set();  // Or Redis set

app.post('/webhooks/orders', async (req, res) => {
  const eventId = req.headers['x-webhook-id'];
  
  // Check if already processed
  if (processedEvents.has(eventId)) {
    console.log('Duplicate event, ignoring');
    return res.status(200).send('Already processed');
  }
  
  // Process event
  await processEvent(req.body);
  
  // Mark as processed (TTL: 7 days)
  processedEvents.add(eventId);
  redis.setex(`webhook:${eventId}`, 7 * 24 * 60 * 60, '1');
  
  res.status(200).send('OK');
});
```

### Delivery Status Tracking

```json
GET /webhooks/wh_xyz789/deliveries

[
  {
    "id": "del_123",
    "event_id": "evt_abc",
    "event_type": "order.created",
    "attempt": 1,
    "status": "success",
    "status_code": 200,
    "response_time_ms": 145,
    "delivered_at": "2026-01-11T10:30:05Z"
  },
  {
    "id": "del_124",
    "event_id": "evt_def",
    "event_type": "order.completed",
    "attempt": 3,
    "status": "failed",
    "status_code": 500,
    "error": "Connection timeout",
    "last_attempt_at": "2026-01-11T10:35:15Z",
    "next_retry_at": "2026-01-11T10:40:15Z"
  }
]
```

## Testing & Debugging

### Test Event Endpoint

```
POST /webhooks/wh_xyz789/test
{
  "event_type": "order.created"
}

→ Sends test event to webhook URL
```

### Webhook Replay

```
POST /webhooks/wh_xyz789/deliveries/del_124/retry

→ Retries failed delivery
```

### Local Development (ngrok)

```bash
# Expose local server to internet
ngrok http 3000

→ Forwarding https://abc123.ngrok.io → localhost:3000

# Register webhook
POST /webhooks
{
  "url": "https://abc123.ngrok.io/webhooks/orders"
}
```

## Monitoring & Alerting

### Metrics

- Delivery success rate (by webhook)
- Retry rate
- Average response time
- Queue depth (pending deliveries)

### Alerts

```yaml
alerts:
  - name: WebhookFailureRate
    condition: failure_rate > 10%
    duration: 15 minutes
    action: notify team + disable webhook
  
  - name: WebhookResponseSlow
    condition: p95_response_time > 5s
    duration: 10 minutes
    action: notify webhook owner
  
  - name: WebhookQueueBacklog
    condition: queue_depth > 10000
    action: scale workers
```

### Dashboard for Clients

```
Webhook: wh_xyz789
URL: https://myapp.example.com/webhooks/orders
Status: ✓ Active

Last 24 hours:
- Deliveries: 1,245
- Success: 1,230 (98.8%)
- Failed: 15 (1.2%)
- Avg response time: 125ms

Recent Deliveries:
Event               Status  Attempts  Response Time
order.created       ✓       1         95ms
order.completed     ✓       1         145ms
order.cancelled     ❌       3         timeout
```

## Best Practices

1. **Idempotency**: Track event IDs; handle duplicates
2. **Quick Response**: Return 200 immediately; process async
3. **Signature Verification**: Always validate HMAC
4. **Minimal Payloads**: Include ID, let client fetch details
5. **Retry Strategy**: Exponential backoff with max attempts
6. **Timeout Handling**: Fail fast; don't let webhooks hang
7. **Monitoring**: Alert on high failure rates
8. **Documentation**: Clear examples and signature algorithm
9. **Testing**: Provide test mode and replay endpoints
10. **Graceful Failure**: Disable webhook after persistent failures

## Common Pitfalls

❌ **No retry limit**: Endless retries waste resources
❌ **Synchronous processing**: Webhook times out; server retries; duplicate work
❌ **No signature**: Anyone can send fake events
❌ **Large payloads**: Slow delivery; higher failure rates
❌ **No idempotency**: Retries cause duplicate orders/charges
❌ **Poor error handling**: 200 OK even on processing failure
❌ **No monitoring**: Silent failures; integrations break
