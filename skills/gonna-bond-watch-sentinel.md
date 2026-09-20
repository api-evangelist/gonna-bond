---
name: Watch a merchant with a webhook-armed sentinel
description: Put a 7- or 30-day LEGIT watch on one x402 merchant, receive webhook events on real trust transitions, and replay missed events from the free feed.
api: openapi/gonna-bond-legit-openapi.yml
base_url: https://legit.gonna.bond
operations:
  - create_watch_v1_watch_post
  - watch_status_v1_watches__watch_id__get
  - watch_events_v1_watches__watch_id__events_get
auth: x402 to create; the returned watch id (a bearer capability) to read
cost: one x402 payment per watch (0.004 USDC algorand-mainnet / 0.005 USDC base-mainnet); reads are free
generated: '2026-09-19'
method: generated
---

# Watch a merchant with a webhook-armed sentinel

## Steps

1. **Create the watch.** `POST /v1/watch` (`create_watch_v1_watch_post`) with `{"address": "<merchant>", "webhook_url": "https://<your receiver>", "days": 7 | 30}`. `days` is optional and defaults to 30; the 7-day "diagnostic" tier costs the same. The first response is a 402 x402 challenge — pay and retry (see the x402 skill). The **201** body carries `watch_id`, `address`, `network`, `created_at`, `expires_at`, `ttl_days`, `webhook_configured` and the `initial` `{score, grade}` the sentinel was armed against.
2. **Store the `watch_id` as a secret.** It is a bearer capability: whoever holds it reads status and events for free. Keep it out of logs, screenshots and client-side code.
3. **Receive events.** LEGIT POSTs JSON to `webhook_url` once per real transition, with a 5-second timeout, from a stock `python-httpx` User-Agent, about 2-3 minutes after the event `ts`. Body keys are always `watch_id, address, event, kind, ts, detail`; `event` and `kind` carry the same value. Kinds: `grade_change` (`detail {from, to, score}`), `score_drop` (`{from, to, threshold}` — default threshold 10 points), `outage` (`{attempts, ok}` — 3+ probes, 0 successes), `recovered` (`{attempts, ok}`). `detail.score` is the score at the transition, frozen; call `/v1/check` for the live score.
4. **Treat the webhook as a doorbell, not a queue.** Delivery is best-effort and **never retried**. If your receiver was down, poll `GET /v1/watches/{watch_id}/events` (`watch_events_v1_watches__watch_id__events_get`) — newest first, capped at 200 — and replay what you missed. Check the watch itself with `GET /v1/watches/{watch_id}` (`watch_status_v1_watches__watch_id__get`).
5. **Expect silence.** Events dedup by state, never by time: no transition means no event, and "a quiet feed is a healthy system".

## Rules

- No cancel or delete operation exists; a watch runs to `expires_at`. Events stay readable until garbage collection 7 days (default) past expiry.
- Webhook deliveries are not signed; verify the `watch_id` in the body against the one you hold and treat the feed as the source of truth.
- Creating a watch is not idempotent — a blind retry pays for a second watch.
