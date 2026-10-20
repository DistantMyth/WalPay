# PayPal payments, wallet connection, and settlement reliability

Planning baseline: 16 October 2026. This is a proposed backend specification, not implemented code or a claim of enabled PayPal products. Read alongside [the architecture](02-backend-architecture-and-stack.md), [domain and ledger](03-domain-data-and-ledger.md), [API conventions](06-api-contracts-and-events.md), [identity and authorization](09-security-identity-and-authorization.md), and [global currency and FX](15-global-currency-and-fx.md).

## 1. What “connect to PayPal's wallet” means

WalPay uses PayPal as the payment provider. WalPay stores identities, permissions, purchase intentions, transaction references, and accounting evidence; it does not issue a stored balance or hold a user's money. The server cannot assume it may read a consumer's PayPal balance, debit that balance, or spend at arbitrary merchants merely because the consumer linked an identity.

Verified product boundaries as of the planning date:

| Capability | Classification | Backend consequence |
| --- | --- | --- |
| First-party WalPay store checkout using Orders v2 | Public integration; prove with sandbox accounts | Start here; WalPay is the merchant and owns the inventory/order. |
| PayPal buyer approval followed by server capture | Supported checkout pattern | User approval in PayPal is part of the payment protocol; local authentication is additional. |
| Log in with PayPal | Identity integration | Link the consented provider identity; do not interpret identity consent as a financial mandate. |
| Refund an owned/authorized capture | Supported API, subject to merchant permission and provider rules | Refund endpoint is merchant scoped and independently idempotent. |
| Onboard outside merchants and process on their behalf | Platform approval and seller permission required for live use | Use capability gates and onboarding; do not substitute the first-party merchant account. |
| Saved PayPal payment methods / off-session charging | Separate product, eligibility, and explicit customer mandate | Optional later phase; default checkout still requires PayPal approval. |
| Read every consumer's wallet balance and transactions | No general permission established by the selected integration | Do not expose a fake balance or request an unavailable scope. |
| Transfer between arbitrary users as a UPI replacement | Not provided by ordinary merchant Checkout | Requests/splits can create obligations; settlement needs an eligible provider product and corridor. |
| Pay by WalPay PIN or biometric alone | Unsupported as a replacement for required PayPal approval | WebAuthn/PIN can authorize WalPay's action; PayPal still controls its own approval. |
| India-to-India checkout or personal transfers | Unsupported in this PayPal plan | Capability service rejects the corridor before creating a provider order. |

PayPal's Identity documentation describes consent to basic non-financial information. Accordingly, the separation of identity linking and payment authority above is a WalPay design constraint. [Log in with PayPal](https://developer.paypal.com/log-in/how-it-works)

PayPal documents server-side Orders v2 creation and capture after approval. [Orders integration](https://developer.paypal.com/api/rest/integration/orders-api)

India's official help explicitly excludes domestic receipts. Use truthful supported sandbox identities for demonstrations; sandbox success does not establish Indian live eligibility. [India domestic-payment restriction](https://www.paypal.com/in/cshelp/article/can-i-use-paypal-to-receive-payments-from-indian-customers-help1049)

PayPal Complete Payments Platform allows sandbox exploration before live partner approval; live access is approval gated. [Platform onboarding](https://developer.paypal.com/platforms/get-started)

## 2. Modules and runtime boundaries

Implement a modular TypeScript/Fastify backend with these components:

| Component | Responsibilities | Explicitly excluded authority |
| --- | --- | --- |
| `payment-intents` | Canonical amounts, merchant, buyer, quote/policy snapshot, lifetime | Accepting a client-provided final total as authoritative |
| `action-authorization` | Single-use WalPay authorization bound to amount, currency, merchant, quote version and action | Minting PayPal consent |
| `paypal-client` | OAuth client credentials, fixed provider hosts, request IDs, schema validation, timeout classification | Acting on arbitrary client URLs or choosing the merchant from model output |
| `payment-operations` | Durable create/capture/refund operations and recovery | Retrying ambiguous writes under new identifiers |
| `paypal-webhook-inbox` | Authenticity, durable deduplication, dispatch | Trusting browser redirects or webhook event names alone |
| `payment-projector` | Merge authoritative provider evidence into local states | Reversing a completed capture merely because an older pending event arrives |
| `reconciliation` | GET provider resources, identify missing evidence, resolve unknown outcomes | Issuing another charge to repair a missing local record |
| `settlement-ledger` | Immutable journal entries, provider references, reversal entries | Serving a cash-outable WalPay balance |
| `capabilities` | Environment, country, currency, merchant capability and consent checks | Letting an LLM override provider eligibility |

PostgreSQL is authoritative. Redis/BullMQ schedules work and caches short-lived data; loss of Redis must not lose payment instructions. PostgreSQL outbox records recreate jobs. PayPal credentials belong only to the server's secret store. Separate sandbox/live databases, webhook registrations, credentials, queues, and audit streams.

Recommended payment-worker concurrency: configurable, initially two jobs per merchant credential and one active operation per payment. Enforce the latter through a database lease/compare-and-set, not only a queue job ID. Use fixed operation IDs as job IDs; expired leases are recoverable.

## 3. Capability checks and country/currency policy

Store versioned `provider_capability_rules` and merchant capability observations. Each decision records `rule_version`, environment, buyer/seller country evidence, product, currency, source URL, `observed_at`, and result. Country information is from verified provider/merchant onboarding evidence where available; user profile country is a provisional input, not sufficient proof of live eligibility.

```ts
type CapabilityDecision = {
  allowed: boolean;
  mode: "simulation" | "sandbox" | "live";
  reasons: string[];
  ruleVersion: number;
  checkedAt: string;
  expiresAt: string;
};

// Pure deterministic policy. Provider response can still reject an allowed attempt.
function canCreateCheckout(input: CheckoutContext): CapabilityDecision {
  requireKnownEnvironment(input.environment);
  requireCurrencySupportedFor("orders", input.currency);
  requirePositiveMoney(input.total);
  requireMerchantCapability(input.merchant, "PAYPAL_CHECKOUT");
  if (input.buyerCountry === "IN" && input.sellerCountry === "IN") {
    return deny("PAYPAL_DOMESTIC_INDIA_UNSUPPORTED");
  }
  if (input.integration === "third_party") {
    requireActivePlatformAndSellerConsent(input);
  }
  return allowWithEvidence(input);
}
```

Use an explicit product-specific currency registry. A generic ISO-4217 list is not proof of PayPal support; Orders, Payouts, and Subscriptions may differ. USD is the initial real sandbox test currency. Add EUR/GBP/JPY only after adapter and account tests. INR display estimates may be shown as non-payable estimates by downstream clients; do not send INR to an unsupported product. PayPal documents special currency precision and receiving restrictions. [PayPal currency reference](https://developer.paypal.com/api/codes/currency)

Unknown or unavailable product capability returns the canonical `CAPABILITY_UNAVAILABLE` problem with a remediation code; an explicitly unsupported corridor returns `422 UNSUPPORTED_PAYMENT_CORRIDOR`. Provider failures remain possible after a successful local check; never promise acceptance or delivery time from capability metadata.

## 4. Canonical money and calculation rules

All WalPay HTTP money uses `{ "currency": "USD", "amount_minor": "1299" }`. Amount strings match canonical `^(0|[1-9][0-9]{0,17})$`: at most 18 unsigned ASCII digits without exponent or decimal point. Reject negative, leading-plus, whitespace, `NaN`, exponent notation, excessive length, and unknown currencies before BigInt conversion. Enforce positive amounts for charges and refunds; zero is permitted only for non-payment accounting fields. Use `bigint` for calculations and PostgreSQL `numeric(30,0)` for persisted minor units; endpoint/product/account limits are explicitly configured below the 18-digit transport ceiling and provider limits.

```ts
function toPayPalValue(m: Money, exponent: number): string {
  if (!/^(0|[1-9][0-9]{0,17})$/.test(m.amount_minor)) throw invalidMoney();
  const n = BigInt(m.amount_minor);
  if (n <= 0n || n > configuredMaximum(m.currency)) throw invalidMoney();
  if (exponent === 0) return n.toString();
  const divisor = 10n ** BigInt(exponent);
  return `${n / divisor}.${(n % divisor).toString().padStart(exponent, "0")}`;
}
```

Resolve exponent from the supported product/currency registry, never from request JSON. Parse provider decimal strings using a decimal parser with an exact expected exponent; reject unexpected precision rather than rounding silently. Never use JavaScript `Number`, `parseFloat`, or `toFixed` for settlement amounts. USD is the primary product reporting/demo currency; preserve original settlement currencies throughout.

Checkout total equals item totals + tax + shipping + handling + insurance − discounts. Recalculate from a trusted quote snapshot. Discounts cannot make the total negative; every component uses the same settlement currency. Internal FX estimates require a rate ID, source, timestamp, expiration, rounding policy, and explicit “estimate” status. The amount PayPal actually presents to the buyer can include provider-controlled conversion; WalPay cannot guarantee an unquoted funding-currency charge.

An executable conversion quote, where a specifically enabled provider product supports one, must bind source/target amounts and currencies, provider and WalPay fees, net recipient amount, quote authority, expiration and actual settlement evidence. A market-rate estimate is not an executable PayPal quote. WalPay may charge an expressly disclosed business fee only through a supported merchant/platform arrangement; do not invent an arbitrary FX/P2P fee field or treat merchant Checkout as a customer-to-customer currency swap. Peer conversion remains a separate provider/compliance-gated extension in the FX plan.

## 5. Persistence and constraints

The shared schema owns base users, merchants, quotes, journals, and audit events. Add these payment-specific entities:

| Entity | Minimum columns / invariants |
| --- | --- |
| `payment_intents` | UUIDv7 ID; tenant, buyer, merchant; immutable quote ID/version/hash; currency; amount minor; canonical status; mode; expires_at; authorization requirement; optimistic version |
| `provider_operations` | UUIDv7 ID; intent ID; `create_order/capture/refund/get_status`; stable provider request ID; body hash; internal preparation/lease phases mapped to canonical operation states; lease owner/expiry; attempts; next_attempt_at; debug ID |
| `provider_orders` | Provider/environment/merchant account/order ID unique; payment ID unique for the single-active-order MVP; raw provider status; approval URL metadata; updated_at |
| `provider_captures` | Provider/environment/merchant/capture ID unique; order ID; currency/amount; provider status; provider timestamps; final_capture; observed_at |
| `refunds` | UUIDv7 ID; capture ID; requesting actor; reason; currency/amount; canonical `requested/submitted/pending/succeeded/failed/outcome_unknown`; provider refund ID unique when present |
| `provider_observations` | Append-only resource ID, response hash, observation source, sanitized payload reference, provider timestamp and receipt time |
| `webhook_inbox` | Provider, environment, app/webhook identity, event ID unique; raw body encrypted/reference; headers allowlist; verification result; processing state; retry count |
| `idempotency_records` | Tenant, actor, route, caller key unique; canonical request hash; operation ID; HTTP result/status; expiry |
| `authorizations` | User/session, frozen financial digest, order-start expiry, bounded payment-consent expiry, consumed_by_order_operation nullable unique; WebAuthn/PIN assurance metadata |
| `capture_permits` | Internal planned extension to the shared authorization schema: same intent/revision/digest, provider order, originating consumed authorization/order operation, expiry, consumed_by_capture_operation nullable unique |
| `payment_outbox` | UUIDv7 ID; operation/event ID; payload; dispatch state; tenant and ordering key |

Database checks: money currency matches the linked intent/capture; unique provider IDs include environment/account; no successful capture is counted twice; refund reservations plus successful refunds never exceed capture amount; one local action grant binds to one order operation and one capture permit binds to its corresponding capture operation. Permit lineage cannot authorize another intent/order. Journal posting uniqueness uses the canonical financial fact/resource identity and accounting rule, not a delivery observation ID; API and webhook observations of one capture cannot post two economic effects. Cross-row refund limits require row locking/serializable logic, not a SQL `CHECK` that tries to query another row. Application owner checks are reinforced by tenant-aware repository methods and, where adopted, row-level security.

Do not retain full PayPal payer email/address in every observation. Keep a redacted operational view plus encrypted evidence with a defined retention schedule. Logs contain local IDs, provider resource IDs and PayPal `debug_id`, never secrets, authorization headers, PINs, or unfiltered webhook payloads.

## 6. WalPay HTTP contracts

All routes below are proposed WalPay APIs, not PayPal APIs. The route, state, money and error conventions in 06 are authoritative. All mutations require authenticated tenant/actor context and `Idempotency-Key`; revision-dependent financial commands require `If-Match`. The webhook route uses provider authenticity instead of user authentication. Return RFC 9457 problem JSON with uppercase machine codes. UUID examples are illustrative UUIDv7 values.

| Route | Permission / preconditions | Success |
| --- | --- | --- |
| `POST /v1/payment-intents` | Buyer; valid server quote, merchant eligibility | `201` local intent; no provider order yet |
| `GET /v1/payment-intents/{id}` | Buyer or authorized merchant/support scope | `200` sanitized status and evidence freshness |
| `POST /v1/payment-intents/{id}/authorization-options` | Buyer; freeze financial digest and revision | `200` transaction-bound challenge |
| `POST /v1/payment-intents/{id}/authorizations` | Buyer; valid assertion for exact challenge | `201` single-use order-start grant |
| `POST /v1/payment-intents/{id}/paypal-order` | Buyer; valid grant, current reservation and frozen terms | `202` durable order operation; consumes grant for this operation |
| `POST /v1/payment-intents/{id}/capture` | Buyer/authorized service; provider approval and bounded capture permit for same intent/order | `202` stable capture operation |
| `POST /v1/payment-intents/{id}/cancel` | Buyer; uncaptured locally; reconcile ambiguous attempts first | `200/202`; local cancellation does not undo settled money |
| `POST /v1/refunds` | Owning merchant `refund:write`; capture, reason, refundable amount; step-up policy | `202` refund operation |
| `GET /v1/operations/{id}` | Buyer/owning merchant as appropriate | `200` capture/refund/order operation state |
| `POST /v1/webhooks/paypal/{bindingId}` | Binding resolves configured environment/app/webhook; verified PayPal message | Canonical `200/202` only after synchronous verification and durable accepted inbox write |

GET-only manual reconciliation is an internal operations command, not an additional public route here. If a public admin route is later approved, add its full contract to 06 first.

Example create request (amount comes from the quote; client does not set it):

```http
POST /v1/payment-intents
Idempotency-Key: 46d17114-fb9f-4a0c-b5c4-a8aed0e97459
Content-Type: application/json

{
  "quote_id": "0199e6c0-1000-7000-8000-000000000101",
  "quote_version": 3,
  "purpose": "merchant_purchase",
  "delivery_address_id": "0199e6c0-1000-7000-8000-000000000102"
}
```

```json
{
  "id": "0199e6c0-1000-7000-8000-000000000103",
  "status": "awaiting_user_approval",
  "total": { "currency": "USD", "amount_minor": "1299" },
  "merchant": { "id": "0199e6c0-1000-7000-8000-000000000104", "display_name": "WalPay Store" },
  "quote_version": 3,
  "financial_digest": "sha256:canonical-action-digest",
  "expires_at": "2026-10-16T17:10:00Z",
  "provider": { "name": "paypal" },
  "required_actions": ["walpay_step_up", "paypal_buyer_approval"],
  "mode": "sandbox",
  "version": 1,
  "request_id": "0199e6c0-1000-7000-8000-000000000199"
}
```

After local verification, `POST /v1/payment-intents/{id}/paypal-order` takes the issued `authorization_id` and current version. Its transaction consumes that grant for one durable order operation, with retries bound to that same operation. The worker creates the provider order and records a bounded internal capture permit referencing the same frozen intent, originating authorization and returned order ID. Permit creation does not prove provider approval. The API returns an allowlisted provider approval URL only after provider order creation. A browser return URL is a navigation signal; it never marks a payment approved or paid. The server checks the stored PayPal order and its merchant, amount, currency, reference and state.

```http
POST /v1/payment-intents/0199e6c0-1000-7000-8000-000000000103/capture
Idempotency-Key: 50064288-dc61-4089-8de0-8923477764a7
If-Match: "3"

{
  "quote_version": 3
}
```

```json
{
  "payment_id": "0199e6c0-1000-7000-8000-000000000103",
  "operation_id": "0199e6c0-1000-7000-8000-000000000106",
  "status": "capture_pending",
  "status_url": "/v1/payment-intents/0199e6c0-1000-7000-8000-000000000103",
  "mode": "sandbox",
  "version": 4,
  "request_id": "0199e6c0-1000-7000-8000-000000000198"
}
```

If PayPal approval is missing, return `409 PROVIDER_APPROVAL_REQUIRED`; do not manufacture approval or reuse the order-start grant. Capture preparation atomically consumes the separate internal capture permit for one stable capture operation. The short grant expiry governs starting its named order operation; a separately bounded, persisted consent window governs later capture. An expired/revoked capture consent window requires renewed local authorization for the same immutable intent/order before a replacement permit can be issued. It never creates authority for another order or a second payment. A fresh amount, merchant or other material financial term requires a new quote/intent and fresh authorization. An ambiguous submitted capture remains attached to its existing operation regardless of expiry.

Canonical refund request: `{ "capture_id": "0199e6c0-1000-7000-8000-000000000107", "amount": { "currency": "USD", "amount_minor": "500" }, "reason_code": "CUSTOMER_RETURN" }`. The service verifies merchant step-up according to policy and binds its authorization to the refund operation; any additional authorization field must be added to 06/OpenAPI before exposing it. `GET /v1/operations/{id}` and scoped transaction views expose refund state, `refundable_amount`, successful/pending refunds, provider status, and `last_verified_at` without confusing a pending refund with money returned.

## 7. PayPal adapter and wire format

Use server-side OAuth client credentials against fixed hosts:

- Sandbox: `https://api-m.sandbox.paypal.com`.
- Live: `https://api-m.paypal.com`.

Cache an access token by environment and credential identity until shortly before its provider expiry. Use a single-flight refresh lock to avoid a token stampede. In-memory token caching is enough for the first worker deployment; an encrypted shared cache can follow. Never place client secrets in a mobile/web client, AI prompt, fixture, Postman export, or repository.

Map a trusted USD 12.99 store intention to this illustrative provider request. These provider JSON names deliberately differ from WalPay's canonical money format:

```json
{
  "intent": "CAPTURE",
  "purchase_units": [{
    "reference_id": "0199e6c0-1000-7000-8000-000000000103",
    "custom_id": "0199e6c0-1000-7000-8000-000000000103",
    "amount": { "currency_code": "USD", "value": "12.99" }
  }],
  "payment_source": {
    "paypal": {
      "experience_context": {
        "return_url": "https://api.walpay.example/paypal/return",
        "cancel_url": "https://api.walpay.example/paypal/cancel",
        "user_action": "PAY_NOW"
      }
    }
  }
}
```

POST to `/v2/checkout/orders`, persist the returned ID, and read the appropriate approval/payer-action HATEOAS relation returned by the chosen flow. Set `PayPal-Request-Id` to the persisted operation UUID and `Prefer: return=representation`. Do not hardcode a provider checkout URL or reuse one from a fixture. The server return endpoint validates its own stored flow/session correlation; arbitrary `return_url` input is prohibited. [Create order reference](https://developer.paypal.com/api/orders/v2/orders-create)

Capture uses `POST /v2/checkout/orders/{stored_order_id}/capture` with `{}` and its own stable request UUID after the provider approval check. A capture response must be inspected at the capture-resource level; an HTTP success status alone is insufficient. [Capture reference](https://developer.paypal.com/api/orders/v2/orders-capture)

Refund uses `POST /v2/payments/captures/{stored_capture_id}/refund`. Send a validated `amount` for a partial refund and use a distinct persisted refund operation key. Record the refund resource and GET it during reconciliation. [Refund reference](https://developer.paypal.com/api/payments/v2/captures-refund)

For outside merchants only after onboarding, set the payee from the server's verified merchant mapping and use the granted partner context/headers. Never allow callers to inject `PayPal-Auth-Assertion`, partner attribution codes, payment source tokens, or payee identifiers. Any third-party assertion construction must follow the approved integration; it is not an application login JWT and is never accepted as evidence of user authentication.

The MVP uses `CAPTURE`. An `AUTHORIZE` mode is a later inventory-reservation feature requiring authorization IDs, voids, expiration/reauthorization handling, and a separately approved final capture. It must not be marketed as escrow or a guaranteed hold. Do not implement a 29-day shopping-agent budget hold merely because the provider has an authorization lifecycle.

## 8. State machines and invariants

Canonical intention states align with the shared API:

```text
draft -> awaiting_user_approval -> awaiting_provider_approval -> approved
approved -> capture_pending -> succeeded
capture_pending -> failed | outcome_unknown
outcome_unknown -> awaiting_provider_approval | approved | capture_pending | succeeded | failed
uncaptured states -> cancelled | expired                (subject to race checks)
```

The two approval requirements are separately recorded facts; neither is optional. The canonical initial flow obtains local authorization before creating the provider order, consumes its grant at the named order operation, and then awaits PayPal approval. A later local reauthorization may be required if the capture consent window expired even though provider approval exists; its new permit remains bound to the existing intent/order and cannot create a second order. `approved` means the exact provider order is approved and local capture policy remains valid, not paid. A timed-out order creation that GET/same-operation recovery resolves to `CREATED` returns to `awaiting_provider_approval`; it cannot be mapped to approved or paid. Follow the authoritative transition table in 06 for all public states.

Capture resource states and refund states are stored independently from the intention. A payment remains `succeeded` with refund/reversal totals and risk flags after settlement; report effective settlement separately. Avoid destructive transitions that erase original successful evidence.

Rules:

1. Only the payment service may prepare capture; an AI tool may propose an intention, not write `succeeded`.
2. Fulfillment is released only by verified completed capture evidence matching intended merchant/currency/amount.
3. A local cancellation and a provider capture may race. Lock the intention to order local commands; if capture was already sent, reconcile before claiming cancellation. Successful capture requires refund workflow, not a status overwrite.
4. Unknown does not mean failed. Do not recreate a payment while a create/capture result is ambiguous.
5. A transport timeout, worker death, or a truncated response creates `outcome_unknown`; recovery reuses the same operation and request ID.
6. Late pending/approved events cannot downgrade a known successful capture. Refunds/reversals are new facts linked to the original capture.
7. A quote expiry blocks starting a new capture. If an approved capture was already sent before expiry, reconcile it; expiry cannot cancel external money movement retroactively.
8. Merchant/quote identity and complete all-in settlement amount are immutable once the buyer authorizes. A revised deal requires reauthorization.

## 9. Durable capture algorithm

Use a database transaction to prepare work; release database locks before calling PayPal. Holding a SQL transaction open across provider I/O creates long locks and still cannot produce atomicity with PayPal.

```ts
async function prepareCapture(input: CaptureInput, actor: Actor) {
  return db.transaction(async tx => {
    const p = await tx.paymentForUpdate(input.paymentId, actor.tenantId);
    requireOwner(p, actor.userId);
    const cached = await tx.findIdempotency(actor, input.key, input.bodyHash);
    if (cached) return cached; // Different hash under same key raises 409.
    requireApprovedProviderOrder(p);
    requireUnexpiredQuoteAndSameVersion(p, input.quoteVersion);
    requireCurrentMerchantAndCorridorCapability(p);
    const existing = await tx.findNonterminalCapture(p.id);
    if (existing) return tx.bindIdempotencyTo(existing, input);
    requireNoPreviouslySucceededCapture(p);
    const permit = await tx.capturePermitForUpdate(p.id, p.providerOrderId);
    // Original order-start grant is already consumed. Validate its lineage, not reusability.
    requireCapturePermit(permit, actor, p.financialDigest, p.providerOrderId);
    requireOriginatingAuthorizationAndOrderOperation(permit, p);
    requireCurrentConsentWindowAndReservation(permit, p);
    const op = await tx.insertOperation({
      id: uuidv7(), paymentId: p.id, type: "capture", state: "prepared"
    });
    await tx.consumeCapturePermitForOperation(permit.id, op.id);
    await tx.setPaymentState(p.id, "capture_pending");
    await tx.insertOutbox({ type: "capture.requested", operationId: op.id });
    return tx.storeIdempotentAcceptedResult(input, op);
  });
}

async function runCapture(operationId: string) {
  const op = await claimOperationLease(operationId);
  if (!op) return;
  if (op.firstSubmissionAlreadyStarted) {
    // A stale lease/timeout cannot establish that the previous send did not happen.
    return reconcileExistingOperation(op);
  }
  // Under a short transaction, reload frozen terms, permit lineage/expiry, current
  // session/delegation revocation, merchant/corridor capability, quote and reservation.
  // Atomically compare-and-set first submission. No network call while SQL locks are held.
  const firstSendAllowed = await recheckAndRecordFirstSubmission(op);
  if (!firstSendAllowed) return blockBeforeSubmissionAndRequireRemediation(op);
  // body and request ID were frozen when the operation was prepared.
  try {
    const response = await paypal.capture(op.orderId, {
      requestId: op.id, body: {}, timeoutMs: 12000
    });
    validateProviderCapture(response, op.expected);
    await persistObservationAndProjectAtomically(op, response);
  } catch (err) {
    if (isVerifiedDefinitiveNoSideEffectRejection(err)) {
      await recordDefinitiveProviderFailure(op, sanitizedError(err));
    } else {
      // Includes timeout, parse/schema failure and amount/payee mismatch AFTER send.
      // A charge may already exist: retain reservation and quarantine mismatched facts.
      await markUnknownOrDiscrepancyAndScheduleRead(op, sanitizedError(err));
    }
  }
}
```

The worker must recover if it crashes after PayPal succeeds but before `persistObservationAndProjectAtomically`. Reconciliation first reads the known order/capture; if a same-key retry is required by the recovery policy, keep the original request ID and body. Keep local operation records longer than the provider's duplicate-detection window. Do not assume a universal idempotency retention period: PayPal support is API specific. [PayPal idempotency](https://developer.paypal.com/api/rest/reference/idempotency/)

Only a successfully authenticated, parsed provider response whose documented rejection proves that the mutation had no financial effect may mark it definitively failed. A malformed response, unexpected status, mismatched amount/payee, truncated body, protocol exception or local persistence failure after submission creates an unknown/discrepancy case. Keep its reservation and operation lineage until GET reconciliation or reviewed evidence establishes the outcome. A generic HTTP `502` in the surrounding transport must never erase this accepted operation or invite a fresh payment.

First-send freshness has a defined local linearization point: the short transaction records the authorized submission before I/O. Revocation after that point cannot unsend an in-flight call; reconciliation and, when appropriate, a separately authorized refund resolve it. Worker recovery after any possibly sent call performs safe inspection even when consent/quote expired; expiration prohibits new money operations, not evidence recovery.

Retry scheduling proposal: jittered exponential backoff beginning at 2 seconds, capped at 5 minutes for transient reads; respect provider rate limits and `Retry-After` when supplied. Separate retry budget from user-facing timeout. Mark unresolved capture outcomes for operations attention after 15 minutes while continuing safe reads. Stop automated writes if idempotency retention has expired without a conclusive result; escalate reconciliation instead of generating a fresh key.

On an OAuth `401`, refresh once for an unquestionably rejected authenticated request; if there is any possibility the mutation was processed, follow the same ambiguous-outcome recovery path. `403`/permission failures block the merchant capability and create remediation work. `422` business errors are classified by provider issue code; do not blanket-retry declines or insufficient permissions. Preserve provider debug IDs.

## 10. Webhooks: authenticate, record, then project

PayPal requires an HTTPS listener and retries unsuccessful deliveries up to 25 times over three days. Its documented self-verification signs a message containing transmission metadata, webhook ID, and CRC32 of the original bytes; mock simulator events do not support postback verification. These limits make a durable inbox and independent reconciliation necessary. [Webhook integration](https://developer.paypal.com/api/rest/webhooks/rest/)

Initial implementation decision: use PayPal's postback verification for real sandbox/live events; add audited self-verification if required for simulator support or reducing external dependence. The webhook ID is resolved from our registered route/environment mapping, never accepted from an incoming body.

```ts
async function verifyRealPayPalWebhook(rawBody: Buffer, h: Headers, registration: Registration) {
  requireBodySizeBelow(rawBody, 256 * 1024);
  const rawJson = rawBody.toString("utf8");
  requireSingleJsonObject(rawJson);
  const metadata = {
    transmission_id: requireHeader(h, "paypal-transmission-id"),
    transmission_time: requireHeader(h, "paypal-transmission-time"),
    cert_url: requireHeader(h, "paypal-cert-url"),
    auth_algo: requireAllowedAlgorithm(h),
    transmission_sig: requireHeader(h, "paypal-transmission-sig"),
    webhook_id: registration.providerWebhookId
  };
  // Preserve the webhook_event's original JSON bytes in the enclosing document.
  const prefix = JSON.stringify(metadata).slice(0, -1);
  const body = `${prefix},"webhook_event":${rawJson}}`;
  const result = await paypalFor(registration).post(
    "/v1/notifications/verify-webhook-signature", body
  );
  return result.verification_status === "SUCCESS";
}
```

This example intentionally does not reserialize the event object. It is proposed pseudocode; test exact formatting behavior with real app-generated events. The official verification endpoint accepts the transmission fields and configured webhook ID. [Signature verification API](https://developer.paypal.com/api/webhooks/v1/verify-webhook-signature-post)

Listener sequence:

1. Apply raw-body size/content-type limits and route/environment lookup before expensive processing. No user session is needed; there is no public toggle that disables verification.
2. Validate required header shapes/lengths and allowed algorithm. For self-verification, fetch only explicitly approved PayPal certificate hosts/paths over HTTPS, block redirects/private addresses, verify certificate validity, cache safely, and sign the documented exact string.
3. Verify authenticity. Failed verification returns `400` and writes a minimal security audit event. Transient verification API outage returns `503`; do not acknowledge unauthenticated financial evidence.
4. Parse and schema-check authenticated JSON. Resolve its merchant/order to our environment and tenant mapping. An authentic unknown resource is quarantined, not applied to arbitrary local IDs from `custom_id`.
5. Insert inbox row plus outbox record in one SQL transaction. Unique event key includes provider, environment and receiving binding/app registration. If already present, compare payload hash and return canonical `200/202`; mismatch is an alert.
6. Return canonical `200/202` only after successful synchronous postback verification and durable commit. SQL unavailable returns `503`; queue unavailable does not matter because the SQL outbox will retry dispatch. Asynchronous verification is an explicit future alternative requiring a durable untrusted inbox and its own contract, not the initial path.
7. Worker obtains relevant resource state using stored provider IDs when needed, checks payee/currency/amount/reference, writes observation, projects capture/refund status, journal and downstream outbox in one transaction.
8. Unsupported authenticated event types are stored as `ignored` with a reason and acknowledged. Poison events go to a dead-letter table with an operator replay action; do not create an infinite provider retry loop after durable capture.

Do not reject every event merely because its transmission time is older than a five-minute window: legitimate provider retries may arrive much later. Signature authenticity, durable event-ID deduplication, known resource mapping and state guards provide replay protection; timestamps are monitoring evidence. If introducing time limits, define delayed-delivery handling and tested provider compatibility first.

Subscribe to the smallest applicable set: `CHECKOUT.ORDER.APPROVED`, `CHECKOUT.PAYMENT-APPROVAL.REVERSED`, `PAYMENT.CAPTURE.COMPLETED`, `.PENDING`, `.DECLINED`, `.REFUNDED`, `.REVERSED`, `PAYMENT.REFUND.PENDING`, `.FAILED`, and relevant dispute events. Do not depend on `CHECKOUT.ORDER.COMPLETED` for ordinary first-party checkout. Verify exact event names against the account's enabled products before registration. [Webhook event reference](https://developer.paypal.com/api/rest/webhooks/event-names/)

## 11. Reconciliation, journals, and fulfillment

Schedule a reconciliation sweeper independently of webhook delivery. It queries local transitional operations, then GETs stored PayPal order/capture/refund resource paths using the correct environment and merchant context. Never fetch a URL from a receipt, QR payload, webhook link, model response, or caller without trusted endpoint reconstruction/allowlisting.

Suggested schedule: every minute for active unknown operations, every five minutes for recent pending payments, and a daily provider-versus-local audit for the configured reporting window. Rate-limit and batch work per credential. Persist `last_verified_at` and cursor checkpoints. Do not equate a provider reporting feed that lags with proof that a charge did not occur.

Journal examples are accounting metadata describing externally held funds, not spendable accounts:

- Completed capture: balanced gross collection / merchant external-settlement clearing entries, linked to the unique provider capture. Record provider fee/net from actual evidence, separately from estimated fees.
- Pending capture: operational observation only; no settled revenue entry.
- Completed refund: linked opposite-direction journal with unique provider refund ID. Reserve pending refunds separately so concurrent refunds cannot overcommit the same capture.
- Reversal/dispute: adjustment journals and case reference; never edit the original journal amount.

Each journal transaction balances within a currency. Do not sum USD and EUR into an invented balance. Provider net settlement, merchant revenue, buyer payment and bank withdrawal are different facts; the MVP reports only evidence available through its authorized integration. Reconciliation discrepancies create cases, freeze affected automated follow-up actions, and retain the original evidence.

Store fulfillment accepts canonical `payment.capture_confirmed.v1` from the transactional outbox with a unique `(capture_id, fulfillment_action)` constraint. A webhook replay cannot ship twice. Refund or reversal after shipment opens a merchant workflow; it cannot undo physical fulfillment. Stock reservation expiry and capture race require explicit resolution: if captured but stock unavailable, queue a merchant-reviewed refund or back-order decision.

## 12. Security and payment-specific checks

- [ ] Merchant is derived from the owned quote and verified provider mapping.
- [ ] Buyer can read/authorize/capture only their own intent; merchant can refund only their captures.
- [ ] Environment identity is persisted on every provider record and cannot change through request JSON.
- [ ] Total, tax, discounts, shipping, fees and currency match the final authorized quote.
- [ ] AI negotiation caps are checked by deterministic policy before the intention and before capture preparation.
- [ ] A local action grant hashes the exact action type, amount, currency, merchant, quote version, payer, shipping digest where material, and expiry.
- [ ] Grant consumption occurs at one order operation; a separately bounded capture permit references that consumed grant and exact intent/order, with one capture operation only.
- [ ] Worker rechecks first-send freshness, reservations, revocation and capabilities; ambiguous sent operations enter recovery regardless of later expiry.
- [ ] No biometric template is sent to the backend; WebAuthn verifies an assertion and expected user-verification policy.
- [ ] Local PIN fallback is rate-limited, securely hashed and subject to lockout/session-recovery policy; it never becomes a PayPal PIN.
- [ ] No card PAN/CVV or bank-login data passes through WalPay server logs or AI tools.
- [ ] Fixed PayPal hosts, TLS verification, bounded timeouts, response-size caps, and redacted error handling are implemented.
- [ ] Caller idempotency and provider operation idempotency are distinct and both persisted.
- [ ] Every financial mutation has an audit actor, purpose, immutable request digest and operation ID.
- [ ] Refund pending amounts are reserved under a capture lock.
- [ ] Consent revocation and merchant capability suspension stop new operations.
- [ ] A support/admin actor cannot silently bypass buyer approval or mutate journal entries.
- [ ] Production keys cannot load in test runners; sandbox endpoints cannot use production secrets.

## 13. Tests and acceptance evidence

These are tests to implement, not tests executed during this planning task.

| Layer | Required cases | Passing evidence |
| --- | --- | --- |
| Money property tests | Decimal serialization/parsing round trip; zero/negative rejection; JPY; max amount; no floating-point drift | Generated cases and fixed regression fixtures |
| Domain state tests | Grant consumed by order; capture permit lineage; expired consent window; same-order reauthorization; quote changed; cancellation/capture race; create-order unknown recovered to awaiting provider approval | Transition table exhaustively exercised |
| API authorization | Cross-user read/authorize/capture, cross-tenant refund, injected merchant/payee, stale quote | Expected `403/404/409`; no provider mutation |
| Idempotency integration | Same key/body; same key/different body; 20 concurrent captures; consumed grant replay; crash after provider success | Exactly one provider capture and one journal/fulfillment action |
| Post-send validation | Truncated/invalid JSON, unknown provider enum, wrong amount/payee, persistence failure after successful capture | Unknown/discrepancy retains reservation; no second payment; recovery inspects original operation |
| Adapter contract | Provider decimal fields; link relation parsing; missing capture array; unexpected amount/payee | Fail closed, sanitized problem response |
| Webhook authenticity | Valid real sandbox signature; modified byte; wrong webhook ID; missing headers; environment mix-up | Only authentic mapped evidence projects |
| Webhook delivery | Duplicate IDs, reordered events, SQL outage, Redis outage, poison event, late retry | No duplicate journal; durable recovery |
| Ambiguous write recovery | Timeout before response; connection reset after accepted capture; worker process death | Original operation reused; GET reconciliation proves outcome |
| Refund concurrency | Two refunds individually valid but jointly too large; refund pending then failed; partial then full remaining | No over-refund; released reservation on definitive failure |
| Sandbox end-to-end | Create, real buyer approval, capture, real webhook, GET verification, partial refund | Redacted provider IDs/debug IDs and timestamps |
| Reconciliation | Missing webhook; webhook before response; provider amount mismatch; revoked merchant | Correct state or explicit discrepancy case |

Use Vitest, Fastify injection, Testcontainers PostgreSQL/Redis, fast-check for invariants, MSW or a controlled local provider stub for failures, and official PayPal Postman collections for manual contract exploration. A stub cannot prove live capability. Do not replace real sandbox acceptance with the simulator: simulator transport/self-signature tests and real app-generated financial events are separate test lanes.

Initial performance targets (proposed): local accepted-intent/capture API p95 under 300 ms excluding external provider round trips; webhook durable acknowledgement p95 under 2 s in healthy conditions; outbox dispatch under 5 s; zero duplicate financial side effects in concurrency/fault tests. These are engineering targets, not provider guarantees.

## 14. Build phases and exit gates

### P0 — contracts and eligibility

- [ ] Register two sandbox business/personal identities in a supported test corridor.
- [ ] Record available app products/scopes and webhook IDs without committing secrets.
- [ ] Freeze Money, error, intention state and operation schemas.
- [ ] Implement capability rules including domestic India rejection.
- [ ] Prove create-order + buyer approval + server capture manually in sandbox.
- [ ] Record exact provider request/response shapes as redacted fixtures.

Exit: a supported real sandbox capture is observable through GET; unavailable products are labelled correctly.

### P1 — reliable first-party checkout

- [ ] Persist payment operation before external I/O.
- [ ] Add caller/provider idempotency and request digest checks.
- [ ] Integrate one-use order-start grants, separately bounded capture permits and immutable quote checking.
- [ ] Authenticate and durably record real webhooks.
- [ ] Project completed capture into one balanced journal and fulfillment event.
- [ ] Implement unknown-outcome reconciliation and concurrency tests.

Exit: duplicate clicks, process death and webhook replay cause one capture and one fulfillment action.

### P2 — refunds and operational recovery

- [ ] Implement partial/full-remaining refund with reservation locking.
- [ ] Handle pending, failed, reversed and disputed evidence.
- [ ] Add reconciliation discrepancy cases and GET-only operator tools.
- [ ] Test restore/replay using persistent inbox/outbox and stable operation keys.
- [ ] Add dashboards for pending age, unknown outcomes, signature failures and queue lag.

Exit: fault-injected captures and refunds converge correctly with documented manual recovery.

### P3 — approved extensions

- [ ] Verify platform approval and merchant consent before third-party processing.
- [ ] Prove required merchant capabilities in sandbox, then separately in authorized live testing.
- [ ] Add saved-method/off-session flows only with explicit supported mandates and tested revocation.
- [ ] Reassess every country/currency/product combination before enabling it.

Exit: each extension has evidence of its own authority and tests; no general-wallet or domestic-P2P promise is inferred from first-party checkout.
