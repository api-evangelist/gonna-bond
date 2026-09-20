---
name: Pre-payment trust check
description: Check an x402 merchant address with LEGIT before paying it for the first time, using only free operations.
api: openapi/gonna-bond-legit-openapi.yml
base_url: https://legit.gonna.bond
operations:
  - check_v1_check__address__get
  - report_v1_report__address__get
  - batch_check_v1_batch_check_post
  - route_v1_route_get
  - networks_v1_networks_get
auth: none
cost: free
generated: '2026-09-19'
method: generated
---

# Pre-payment trust check

Use this before sending an x402 payment to an address you have not paid before. Every step is free and needs no credential.

## Steps

1. **Check the address.** `GET /v1/check/{address}` (`check_v1_check__address__get`). Read `score` (0-100), `grade` (A+ to F, or `UNRATED`), `provisional` and `breakdown.explanation`. `grade: UNRATED` with `breakdown.sample_size: 0` means LEGIT has no measurements yet — treat that as "unknown", not "bad".
2. **Need the full picture?** `GET /v1/report/{address}` (`report_v1_report__address__get`) adds `rank`, `percentile`, `networks_served` and a 30-row `history[]` of `{ts, score}`. Leave `demo` unset: `demo=true` returns fabricated illustration data (always flagged `demo: true`) for five hardcoded demo addresses.
3. **Checking several counterparties?** `POST /v1/batch-check` (`batch_check_v1_batch_check_post`) with `{"addresses": [...]}` — 1 to 20 addresses, one call.
4. **Choosing a merchant for a task?** `GET /v1/route?need=<query>` (`route_v1_route_get`) returns `chosen`, `alternatives[]` and a `payment_hint`. `chosen` can be null.
5. **Confirm the rail is healthy** before you pay: `GET /v1/networks` (`networks_v1_networks_get`) reports per-chain facilitator health.

## Rules

- No authentication exists; do not send API keys or bearer tokens.
- Untracked addresses return `score: null` / `UNRATED`; `GET /v1/history` and `GET /v1/receipts` return **404** for them ("an honest 404, never an invented series"). Do not retry a 404 as if it were transient.
- A 422 means a malformed address or a batch outside 1-20 entries; fix the field named in `detail[].loc`.
- The leaderboard body is cached for 30 seconds; a report is generated per call (`generated_at`).
- No rate limit is published and no `RateLimit-*` or `Retry-After` header is sent.
