---
name: Pay for a LEGIT verdict over x402
description: Call one of LEGIT's five paid verdict operations by answering the HTTP 402 x402 challenge with a USDC payment on Algorand or Base.
api: openapi/gonna-bond-legit-openapi.yml
base_url: https://legit.gonna.bond
operations:
  - compare_v1_compare_get
  - arena_v1_arena_get
  - history_v1_history_get
  - deep_check_v1_deep_check_post
  - create_watch_v1_watch_post
auth: x402
cost: 0.004 USDC per call on algorand-mainnet, 0.005 USDC per call on base-mainnet
generated: '2026-09-19'
method: generated
---

# Pay for a LEGIT verdict over x402

Five operations are paid per call: `GET /v1/compare` (`compare_v1_compare_get`, 2-5 addresses), `GET /v1/arena` (`arena_v1_arena_get`), `GET /v1/history?address=` (`history_v1_history_get`), `POST /v1/deep-check` (`deep_check_v1_deep_check_post`, up to 10 addresses) and `POST /v1/watch` (`create_watch_v1_watch_post`). There are no accounts and no API keys; payment is the credential.

## Steps

1. **Make the call with no payment.** The response is **HTTP 402**. The body is an x402 v2 `PaymentRequired` document — `{x402Version: 2, error, resource, accepts[], extensions}` — and the same document is base64-encoded in the `PAYMENT-REQUIRED` response header.
2. **Pick an offer** from `accepts[]`. On 2026-09-19 there were two: `scheme: exact` on `algorand:wGHE2Pwdvd7S12BL5FaOP20EGYesN73ktiC1qzkkit8=` for `amount: "4000"` of USDC ASA `31566704` (= $0.004, gasless via facilitator.goplausible.xyz), and on `eip155:8453` (Base) for `amount: "5000"` of USDC `0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913` (= $0.005, via facilitator.payai.network). Each offer carries `payTo` and `maxTimeoutSeconds: 300`.
3. **Shape the call from the challenge if you have not read the OpenAPI**: `extensions.bazaar.info.input` and `inputSchema` describe the request, `output.example` and `output.schema` the response.
4. **Pay on the chosen rail and retry the identical request** with the `PAYMENT-SIGNATURE` request header carrying the payment payload. A settled call returns the verdict body (200, or 201 for a watch); `PAYMENT-RESPONSE` is exposed as a response header.
5. **Read the verdict.** `/v1/compare` returns `entries[]`, `winner` (null when no tracked entry has a score) and a one-sentence `rationale`; untracked addresses have `score: null` and cannot win.

## Rules

- **Nothing is reversible.** There is no refund, void or dispute path for a paid call, and no cancel for a watch; confirm the address list before paying. Free operations (`/v1/check`, `/v1/report`, `/v1/batch-check`, `/v1/arena/preview`) answer most questions without spending.
- **Nothing is idempotent.** A retried `POST /v1/watch` whose first response was lost creates and charges a second sentinel. Keep the first response.
- `/v1/history` returns 404 for an address LEGIT does not track — check with `/v1/check` first so you do not pay for a 404.
- The 402 is the normal first response, not an error; the `error` field reads "PAYMENT-SIGNATURE header is required".
- Prices are fixed per rail (`x-payment-info` in the spec: `mode: fixed`) and Algorand is deliberately the cheapest rail.
