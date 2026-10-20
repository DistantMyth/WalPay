# Canonical API contracts and event formats

Proposed canonical REST surface, not an existing running service. Base path `/v1`. JSON field names use `snake_case`; database/SDK internals may use different names at typed boundaries. UUIDv7 examples below are illustrative IDs, not real accounts. Provider payload examples belong to 04/08; these are WalPay contracts.

## Transport and common types

HTTPS only outside local development. Browser sessions use opaque Secure HttpOnly SameSite cookies and an explicit CSRF header for mutations. Service clients use scoped short-lived bearer credentials; partner webhook ingestion uses its specified signature protocol. Do not put tokens or personal data in query strings. Standard requests use `application/json`, failures `application/problem+json`, SSE `text/event-stream`.

```json
{
  "currency": "USD",
  "amount_minor": "4800"
}
```

Money: `currency` must be an enabled uppercase 3-letter currency; `amount_minor` matches `^(0|[1-9][0-9]{0,17})$`, with lower endpoint-specific limits. `0` is valid for a fee or allocation, not a payable total. Do not accept decimal currency strings in WalPay commands. Use a versioned currency exponent table at provider boundaries. UUID schema uses `format: uuid` plus the selected generator; do not use example-prefixed IDs in real contracts.

Successful HTTP responses use `{data: <resource_or_collection>, request_id: <uuid>}`. The resource includes `mode` when financial, `version` for mutable resources, and timestamps. Lists use `{data:{items, next_cursor, has_more}, request_id}`. Problem details remain unwrapped. Some planning examples show a resource representation without this envelope for readability; implement the envelope once at the HTTP boundary. Cursor is signed/opaque and binds filters, tenant and stable `(created_at,id)` sorting. Default25/max100; unknown sort fields rejected. Empty lists are 200. GET never causes money movement or sends invitations.

Require `Idempotency-Key` for every externally meaningful mutation: create mission, issue quote, initiate intent, authorize, capture/refund, send request, confirm split, approve job, create payout/invoice/subscription. Scope by authenticated principal, tenant, method and canonical path. Retain a canonical request hash and original result; same key/different body ->409. In-flight ->202 with operation URL. Require `If-Match: "<version>"` for financial draft edits and revision-dependent confirmations; missing ->428, stale ->412.

### Problem details

```json
{
  "type": "https://walpay.example/problems/budget-exceeded",
  "title": "The total exceeds your spending limit",
  "status": 422,
  "code": "BUDGET_EXCEEDED",
  "detail": "The final total is above the approved maximum.",
  "instance": "/v1/payment-intents/0199e000-0000-7000-8000-000000000001",
  "request_id": "0199e000-0000-7000-8000-000000000099",
  "message_key": "payment.budget_exceeded",
  "retryable": false,
  "field_errors": []
}
```

`walpay.example` is a documentation placeholder; replace with the owned API documentation origin. Error code is stable across locale; `detail` is localized from templates and never exposes stack traces/provider secrets. 400 malformed; 401 no session; 403 denied capability/permission with non-enumerating detail; 404 inaccessible/not found; 409 idempotency/state conflict; 412 revision conflict; 422 business invalid; 429 quota; 502 definitive upstream protocol failure; 503 dependency unavailable. A submitted operation with uncertain outcome remains 202/operation resource, never a generic retryable 500 suggesting a fresh payment.

## Public payment state register

| State | Meaning | Allowed next states |
|---|---|---|
| draft | Valid intent exists, no bound authorization | awaiting_user_approval, cancelled, expired |
| awaiting_user_approval | Frozen terms require local verification | awaiting_provider_approval, cancelled, expired |
| awaiting_provider_approval | Local grant valid, PayPal buyer approval outstanding | approved, cancelled, expired, outcome_unknown |
| approved | Provider approval confirmed for exact order | capture_pending, cancelled, expired |
| capture_pending | Durable capture operation submitted/queued | succeeded, failed, outcome_unknown |
| outcome_unknown | External result uncertain | awaiting_provider_approval, approved, capture_pending, succeeded, failed, cancelled only after verified evidence |
| succeeded | Capture confirmed; fulfilment eligible | terminal for original capture; refunds/disputes separate |
| failed | Definitive provider/policy failure, no uncertain capture | terminal; retry requires explicit new intent if valid |
| cancelled | Confirmed local/provider cancellation as applicable | terminal |
| expired | Terms expired with no unresolved external effect | terminal |

Do not declare `expired` solely because a local clock passed while a capture might have occurred. Adapters retain `provider_status`, `last_verified_at` and settlement substatus without forcing all provider vocabulary into one enum.

## Route register

Each implemented operation must add complete OpenAPI request/response/error/security schemas; the rows below specify scope and behavior. Read-only rows inherit pagination and tenant scoping. Unavailable G routes return `CAPABILITY_UNAVAILABLE`, not an accepting fake operation.

| Method and route | Principal / command behavior | Response / main failures |
|---|---|---|
| POST `/auth/passkeys/registration-options` and `/registrations` | Fresh authenticated session; enroll credential after step-up | options /201 credential; challenge/owner errors |
| POST `/auth/passkeys/authentication-options` and `/sessions` | Account-discovery-safe login | options /201 rotated session; uniform failed auth |
| DELETE `/auth/sessions/current` | User revokes current session |204 |
| GET/PATCH `/me` | User; locale/timezone/preferences only | profile; invalid locale412 revision |
| POST `/provider-connections/paypal/login` | User starts state-bound OAuth connection | approval URL and connection attempt |
| GET `/provider-connections/paypal/callback` | OAuth callback; validates state and exchanges code | safe redirect/status; never pays |
| GET `/capabilities` | User plus intended operation/country/currency facts | supported/gated reasons and checked_at |
| POST `/recipients/resolve` | User; exact consented handle/phone lookup | masked confirmed candidate or uniform unavailable |
| POST `/contacts/invitations` | User with consent; throttled |202 invitation operation, expiry |
| GET `/products` and `/products/{id}` | Shopper; approved catalog/search | provenance, price estimate, seller, checkout capability |
| POST `/shopping-missions` | User; immutable initial constraints |202 mission and operation |
| GET/POST `/shopping-missions/{id}` and `/{id}/cancel` | Owner; view or cancel permitted outstanding work | mission;409 busy unknown side effect |
| POST `/shopping-missions/{id}/offers` | Authorized mission worker via scoped internal tool |201 validated offer;422 cap/policy |
| POST `/shopping-missions/{id}/revisions` | Owner; If-Match, revokes superseded proposals |201 mission revision;409 unresolved financial action |
| GET `/shopping-missions/{id}/proposals/{proposalId}` and `/{id}/negotiation-summary` | Owner; exact offer/provenance, redacted seller policy | proposal representation/summary |
| GET `/shopping-missions/{id}/events` | Owner; mission-scoped SSE equivalent to filtered `/events` | reconnectable status events |
| POST `/merchant-negotiations/{sessionId}/messages` | Consenting merchant agent; scoped/signed nonbinding protocol |201 append-only sequence-checked message |
| POST `/quotes` | Approved own-store seller or policy engine |201 immutable quote; stock/price error |
| POST `/payment-intents` | User; quote/purpose, not arbitrary client-priced cart |201 intent; quote/capability errors |
| GET `/payment-intents/{id}` | Owner/authorized merchant limited view | intent with original/provider facts |
| POST `/payment-intents/{id}/authorization-options` | Owner; freeze exact terms, generate challenge |200 challenge linked to action digest |
| POST `/payment-intents/{id}/authorizations` | Owner; verified WebAuthn response |201 one-use grant;401/409 stale/replay |
| POST `/payment-intents/{id}/paypal-order` | Owner; valid authorization and reservation |202 create operation;403/422 gate |
| POST `/payment-intents/{id}/capture` | Owner/service with exact provider-approved order |202 capture operation;409 state |
| POST `/payment-intents/{id}/cancel` | Owner, supported current state |202/200 cancellation;409 outcome_unknown |
| POST `/refunds` | Merchant/refund role; capture, amount, reason |202 refund operation;422 over-refund |
| GET `/operations/{id}` | Scoped owner; no secrets | durable operation and next_poll_after_ms |
| GET `/transactions` and `/{id}` | User/merchant read model | provider-observed status, coverage and fees |
| POST `/receipts/uploads` | Owner; declares MIME/size, pre-sign |201 upload URL/object token |
| POST `/receipts` | Owner; verified uploaded token |202 scan/extraction operation |
| GET `/receipts/{id}` | Owner/permitted participant redacted | draft extraction, confidence and source |
| POST `/receipts/{id}/confirm` | Owner; If-Match, reviewed amount/currency/components |201 immutable receipt revision;422 inconsistent total |
| POST `/splits` | Expense owner; confirmed source expense |201 draft allocations |
| PUT `/splits/{id}/draft` | Owner; If-Match revision |200 corrected draft;422 conservation |
| POST `/splits/{id}/confirm` | Owner; exact revision consent |201 immutable confirmed revision |
| POST `/splits/{id}/requests` | Owner; confirmed allocations + invite consent |202 request batch; no automatic settlement |
| POST `/splits/{id}/accept` and `/{id}/disputes` | Participant scoped to their allocation/revision |201 acknowledgement or dispute; never automatic payment |
| POST `/splits/{id}/revisions` and `/{id}/cancel` | Owner; If-Match, settlements retained |201 corrected revision/200 cancellation of unsent obligations |
| POST/GET `/payment-requests` | User sends or lists obligations |201/202 request; rate/capability errors |
| POST `/payment-requests/{id}/{accept|decline|cancel}` | Bound recipient/sender per action | updated state; accept may offer eligible rail |
| POST `/payment-requests/{id}/send` and `/{id}/remind` | Sender; channel consent, expiry, block and frequency policy |202 notification operation; no unsolicited agent sends |
| POST `/share-intents` | Owner/eligible campaign admin; creates scoped opaque QR/NFC token |201 share token with recipient/purpose binding and expiry |
| POST `/qr-resolutions` | User/public minimal resolver; opaque signed token | canonical immutable recipient/purpose preview |
| POST/GET `/causes` | Approved cause administrator/public limited view | campaign metadata, currency-specific progress |
| GET `/causes/{id}` and `/{id}/events` | Public aggregate-only campaign facts; no donor/payment identities | currency-specific progress and public SSE with abuse limits |
| POST `/causes/{id}/donation-intents` | Donor; beneficiary verified and supported |201 eligible intent; no arbitrary beneficiary |
| GET `/earnings` and `/earnings/export` | Worker/merchant scoped | per-currency earnings and coverage;202 large export |
| POST `/partner-events/{partnerId}` | Authenticated signed partner webhook |202 durable inbox, 409 conflicting replay |
| GET/POST `/work-items` and `/{id}/approvals` | Partner scoped; approved signer and version | records/202 obligation; signed event recommended |
| POST `/payouts` | Approved business funder, not consumer generic P2P |202 payout operation; funding/receiver gate |
| GET `/payouts/{id}` | Funder/recipient limited view | item states, gross/net/currency |
| POST/GET `/invoices` | Connected eligible issuer |202 creation / invoice records |
| POST `/invoices/{id}/{send|cancel|remind}` | Issuer; exact provider entitlement |202 operation; state/consent errors |
| POST `/subscriptions` and `/{id}/cancel` | User/merchant eligibility; provider approval |202 setup/cancel; entitlement from events |
| POST `/fx-quotes` | User; supported pair, purpose and fees |201 exact executable or estimated quote with explicit kind |
| GET `/fx-quotes/{id}` | Owner | immutable quote, expiry and capability |
| POST `/browser-connectors/pairing-sessions` and `/{id}/confirm` | User pairs selected local connector with one-use local proof |201 pairing/connection record; no raw cookie/debug endpoint |
| GET `/browser-connectors` and POST `/browser-connectors/{id}/revoke` | Owner lists/revokes connector and its active grants |200 scoped connectors/revocation state |
| POST `/browser-task-sessions` | Owner; mission, selected tab/site, read/draft/send scope and expiry |201 bounded browser session; site/capability gate |
| GET `/browser-task-sessions/{id}` | Owner; redacted task status | state, allowed scopes, observation freshness |
| GET `/browser-task-sessions/{id}/events` | Owner; scoped browser-session SSE | redacted status/action events, no cookies/raw debug information |
| POST `/browser-task-sessions/{id}/actions` | Scoped local agent/gateway; policy checked tool request |202 read/action/proposal operation; scope/approval error |
| POST `/browser-task-sessions/{id}/{pause|resume|cancel}` | Owner; reconsent on changed tab/site |200/202 state; revokes unsent actions |
| POST `/browser-action-proposals/{id}/authorizations` | Owner; exact seller/thread/action/offer digest and fresh step-up when required |201 one-use action authorization, never PayPal approval |
| POST `/browser-task-sessions/{id}/negotiation-mandates` | Future B7 owner/local-confirmed finite nonbinding mandate, If-Match |201 reviewed standing mandate; no binding checkout authority |
| POST `/browser-negotiation-mandates/{id}/revoke` | Owner; immediate local/server revocation |200 revoked; unsent permits invalidated |
| POST `/webhooks/paypal/{bindingId}` | PayPal signed webhook to configured binding |200/202 durable inbox, invalid signature rejected |
| GET `/events` | Authenticated scoped SSE | event stream; bounded history/poll fallback |
| GET `/health/live` and `/health/ready` | Minimal public/infra health |200/503; detailed dependencies internal only |

All routes are under `/v1`, including `/v1/auth/...`. Slash brace alternatives in this register describe separate concrete endpoints, not literal dynamic action strings accepting arbitrary methods. Internal offer routes are not public agent authority: service scopes, mission binding and policy checks are required.

## Mission example

```http
POST /v1/shopping-missions
Idempotency-Key: 782b6c2e-70a0-4cda-98c7-0ebc81bc7bb4
Content-Type: application/json
X-CSRF-Token: <session-bound-token>
```

```json
{
  "query": "A compact desk accessory",
  "quantity": 1,
  "constraints": {
    "currency": "USD",
    "max_total": {"currency":"USD","amount_minor":"5000"},
    "opening_offer": {"currency":"USD","amount_minor":"3000"},
    "target_total": {"currency":"USD","amount_minor":"3500"},
    "min_offer_to_send": {"currency":"USD","amount_minor":"2500"},
    "condition": ["NEW"],
    "delivery_country": "US",
    "latest_delivery_date": "2026-11-01",
    "platforms": ["WALPAY_STORE", "CHANNEL3_DISCOVERY"],
    "max_negotiation_rounds": 4,
    "currency_policy": "SAME_CURRENCY_ONLY"
  },
  "destination_ref": "0199e000-0000-7000-8000-000000000010",
  "expires_at": "2026-10-17T12:00:00Z",
  "consent": {"allow_negotiation":true,"allow_checkout_preparation":true,"allow_binding_commitment":false}
}
```

```json
{
  "data": {
    "id": "0199e000-0000-7000-8000-000000000011",
    "status": "queued",
    "workflow_state": "DISCOVERY_QUEUED",
    "mode": "sandbox",
    "operation_id": "0199e000-0000-7000-8000-000000000012",
    "version": 1,
    "capability_results": []
  },
  "request_id": "0199e000-0000-7000-8000-000000000099"
}
```

A worker cannot choose `mode` from request body. Mode comes from deployment/credential binding; clients cannot elevate a simulation to sandbox/live.

## Intent, authorization and capture example

Create request: `{"quote_id":"0199e000-0000-7000-8000-000000000020","purpose":"merchant_purchase","mission_id":"0199e000-0000-7000-8000-000000000011"}`. Response201 returns exact payee preview, item/shipping/tax/fee breakdown, Money total, quote expiry, `financial_digest`, version and required approvals. The server resolves price and payee from the quote; it never trusts client `total` or provider order ID.

Authorization options returns `{challenge_id, public_key, action:{intent_id,revision,financial_digest,total,payee_display_name}, expires_at}`. `public_key` follows the selected WebAuthn library's browser JSON format. Complete request contains `challenge_id` and `credential_response`; success returns `authorization_id`, `expires_at`, and the bound digest. 09 defines verification; the response is never an arbitrary `biometric_ok: true` supplied by a client.

`POST /{id}/paypal-order` consumes the local grant for this specific order operation and returns202 `{operation_id,status:"queued"}`. Once created, GET intent includes only provider-returned, validated approval link(s), `awaiting_provider_approval`, and order expiry. PayPal approval return triggers backend GET/verification; it does not capture itself or set success. A fresh capture command uses a new idempotency key for the capture operation while retaining the authorization binding to the same frozen terms.

```json
{
  "id": "0199e000-0000-7000-8000-000000000030",
  "status": "succeeded",
  "total": {"currency":"USD","amount_minor":"4800"},
  "provider": {"name":"paypal","order_id":"EXAMPLE_ORDER","capture_id":"EXAMPLE_CAPTURE","status":"COMPLETED"},
  "settlement": {"state":"provider_confirmed","last_verified_at":"2026-10-16T16:00:00Z"},
  "mode":"sandbox",
  "version":7,
  "request_id":"0199e000-0000-7000-8000-000000000099"
}
```

Refund request: `{capture_id, amount: Money, reason_code}`. It creates a separate operation; refund states `requested, submitted, pending, succeeded, failed, outcome_unknown`. Captured intent remains succeeded and exposes cumulative verified refund amount. Never use a refund success response to imply the customer's bank has completed its own posting.

After any externally submitted financial mutation, transport timeout, malformed/schema-invalid response, unexpected amount/currency/payee or untrusted provider response means `outcome_unknown` plus discrepancy investigation and retained reservation. Only verified evidence proving no financial effect permits definitive `failed`; HTTP502/503 cannot overwrite the accepted operation's uncertainty. Order-create recovery into provider `CREATED` maps to `awaiting_provider_approval` and preserves the same order operation. Before first capture submission, workers recheck the separate capture permit, frozen terms, mandate/consent window, revocation and current capability; recovery reads remain available after expiry.

## Versioned domain events

```json
{
  "event_id":"0199e000-0000-7000-8000-000000000040",
  "event_type":"payment.capture_confirmed.v1",
  "occurred_at":"2026-10-16T16:00:00Z",
  "recorded_at":"2026-10-16T16:00:01Z",
  "aggregate":{"type":"payment_intent","id":"0199e000-0000-7000-8000-000000000030","version":7},
  "tenant_id":"0199e000-0000-7000-8000-000000000050",
  "mode":"sandbox",
  "correlation_id":"0199e000-0000-7000-8000-000000000012",
  "causation_id":"0199e000-0000-7000-8000-000000000060",
  "data":{"capture_id":"0199e000-0000-7000-8000-000000000070","money":{"currency":"USD","amount_minor":"4800"}}
}
```

Consumers deduplicate event IDs, enforce resource ownership and tolerate additive optional fields. Breaking semantic changes require `.v2`; money/recipient fields cannot change meaning under the same event version. Suggested events: `quote.issued`, `authorization.granted`, `payment.capture_confirmed`, `payment.refund_confirmed`, `payment.reconciliation_required`, `split.confirmed`, `payment_request.accepted`, `work.approved`, `payout.item_confirmed`, `invoice.updated`, `subscription.entitlement_changed`, `campaign.progress_changed`, all with version suffixes.

SSE uses `id: <event cursor>`, `event: payment.status_changed`, JSON minimal data and periodic comment heartbeats. Filter authorization is evaluated server-side on authenticated subscriptions and resource delivery; logout/revocation closes the stream. The explicit public `/causes/{id}/events` exception exposes only approved campaign aggregate amounts/counts per currency and no private donor or transaction facts, with separate rate limits. On cursor expiry return a recoverable reset instruction and fetch current state. An SSE event is a notification to refresh, not payment authority.

## Contract implementation checklist

- [ ] Define request, response, errors and authorization for each implemented route.
- [ ] Use response schemas to prevent accidental secret/private-field serialization.
- [ ] Specify nullable versus absent explicitly and reject unknown financial fields.
- [ ] Build idempotency conflict and If-Match tests for every financial command.
- [ ] Generate client/Postman artifacts from OpenAPI; CI rejects unexpected contract drift.
- [ ] Validate examples against schemas; literal `<...>` placeholders appear only in HTTP documentation.
- [ ] Test wrong tenant, wrong user, expired session and capability-disabled cases for every object route.
- [ ] Keep per-route maximum money, list size, upload size and request quotas explicit.
- [ ] Write provider-to-canonical state mapping tables and reject impossible transitions.
- [ ] Document redacted partner credential/signature examples without real secrets.
