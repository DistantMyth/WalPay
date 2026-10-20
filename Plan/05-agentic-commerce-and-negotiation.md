# Agentic commerce, negotiation, and the WalPay store — backend plan

Planning date: 2026-10-16. No integration, purchase, message, deployment, or money movement has been executed by this plan. All routes below are proposed **WalPay** `/v1` routes, not claims that PayPal exposes routes with these names. Use the shared TypeScript/Fastify, PostgreSQL, Redis/BullMQ stack. All domain IDs are UUIDv7. Money is `{ "currency": "USD", "amount_minor": "12999" }`; minor-unit values are base-10 integer strings, never floating-point numbers. See the common API, payment, security, and testing plans for shared controls. `06-api-contracts-and-events.md` owns canonical DTOs/envelopes; detailed domain additions below must enter that schema register. Success responses use `{data:<resource>,request_id}`; shortened examples may omit `request_id`. Revisioned mutations require matching `If-Match`; mission API `version` is the revision used by internal domain checks.

## 1. Product promise and honest capability boundary

The MVP can demonstrate a buyer agent finding a WalPay-store product through the user's connected browser or approved API, negotiating with a separate seller agent within seller policy, producing an exact all-in quote, obtaining WalPay step-up authorization and **PayPal buyer approval**, and capturing a sandbox payment. [16-user-browser-agent-and-marketplace-automation.md](16-user-browser-agent-and-marketplace-automation.md) specifies the existing Playwright extension/local companion, browser action contracts, user/site consent and real browser test workflow. A user can compare external discoveries, but a result is purchasable only if its adapter has a proven, enabled checkout capability.

Capability labels used throughout this document:

| Capability | MVP classification | Evidence needed to enable it |
| --- | --- | --- |
| WalPay own-store search, inventory, quote, deterministic seller negotiation | Supported local/sandbox | Seeded catalog, real backend state, documented seller policy |
| User-selected existing browser discovery/negotiation on WalPay or consenting test store | Supported browser/sandbox after connector spike | Scoped local Playwright extension companion, per-site/tab grant, reviewed action adapter and exact approval for seller sends |
| PayPal Checkout order creation, buyer approval, capture, refund | Supported sandbox; production approval/account gates | Sandbox business/buyer accounts, tested events and reconciliation |
| PayPal Agent Toolkit/MCP order and invoice assistance | Supported sandbox after tool discovery | Current supported tool schema and least-privilege credentials |
| PayPal remote shopping tools | Approval/feature gated | Runtime tool inventory and successful sandbox capability probe |
| eBay Browse discovery | Sandbox integration; production gated | Marketplace, credentials, permitted production access |
| eBay Order checkout or auction bids | Approval gated, disabled by default | Written/provider entitlement, exact API support and explicit user authorization |
| Arbitrary buyer-to-seller eBay Best Offer negotiation | Unsupported in MVP | A documented buyer negotiation API and applicable access; do not substitute unrelated APIs |
| Facebook Marketplace discovery, messaging, negotiation, checkout | API unavailable in verified plan; browser mode policy gated; simulated fixtures/manual handoff available | Documented allowed automation path plus scoped user consent; browser access does not remove site permission requirements |
| Channel3 search/lookup | Sponsor discovery integration, credential dependent | Key, current schema, terms and quota verified |
| Channel3 checkout | Early-access/approval gated | Access actually granted; provider/merchant PayPal compatibility independently proven |
| PayPal wallet-wide unattended consumer spending | Unsupported assumption | A specific provider authorization product and permitted use case, not merchant OAuth alone |

PayPal documents merchant-oriented local/remote MCP connections and separate sandbox endpoints. Official repository guidance additionally identifies remote commerce tools behind a feature header. Tool presence and header acceptance are not contractual access or end-user purchase consent. [PayPal MCP guide](https://developer.paypal.com/ai-tools/mcp-server), [official toolkit tool inventory](https://github.com/paypal/AI-Toolkit/blob/main/skills/paypal-best-practices/references/mcp-tools.md), checked 2026-10-16.

eBay's Buy Offer API places proxy auction bids and is limited release. Its Sell Negotiation flow sends seller discounts to interested buyers. Neither is evidence of a general buyer bargaining API. Production Buy APIs have eligibility, approval, and contractual requirements. [Offer overview](https://www.developer.ebay.com/api-docs/buy/offer/static/overview.html), [seller negotiation guide](https://developer.ebay.com/api-docs/sell/static/marketing/offers-to-buyers.html), [Buying Integration Guide](https://developer.ebay.com/api-docs/buy/static/buying-ig-landing.html), checked 2026-10-16.

No documented general Facebook Marketplace buyer API was established during research. The API adapter remains unavailable; the separate user-browser adapter has an explicit site-policy gate and can be enabled only for documented allowed workflows plus user consent. Meta describes unauthorized automation as contrary to its terms; a browser session, scraper service or Messenger assumption is not permission. [Meta scraping explanation](https://about.fb.com/news/2021/04/how-we-combat-scraping/), checked 2026-10-16. This is a product access decision, not a claim that no private Meta partnership exists. See plan 16 for real browser automation on owned/consenting stores and the gated external-marketplace path.

Channel3 exposes discovery/lookup documentation while its developer page marks programmatic checkout as coming soon/early access. Use it to broaden comparison without promising settlement or negotiation. [Channel3 developer capabilities](https://trychannel3.com/developers), [Channel3 documentation](https://docs.trychannel3.com/), checked 2026-10-16.

## 2. Service boundaries and package layout

Start as one modular API deployment plus a worker deployment; do not build a distributed service for every agent.

| Module/package | Backend responsibility | Data and dependencies |
| --- | --- | --- |
| `commerce-intents` | User objective, consent, constraints, state machine | PostgreSQL; shared auth/policy |
| `catalog` | Own-store products, variants, search and stock | PostgreSQL text search; optional Elastic sponsor index |
| `marketplace-adapters` | Normalize approved partner discoveries/quotes | Typed HTTP clients, Redis cache, quota and circuit breakers |
| `browser-task-orchestrator` | User-selected browser sessions, typed scoped actions and observation provenance | Local paired companion/Playwright extension; plan 16; no direct model CDP/eval/cookie access |
| `negotiation` | Protocol sessions, deterministic constraints, seller proposals | PostgreSQL append-only offer history |
| `agent-runtime` | Bounded reasoning, tool selection, proposal generation | Model gateway, schema validator, policy gateway |
| `budget-policy` | Spend limits and concurrent reservations | PostgreSQL transactions; Redis is only a hint/cache |
| `checkout-orchestrator` | Immutable proposal -> PayPal order -> approved capture | Shared PayPal provider adapter and event outbox |
| `commerce-workers` | Discovery, quote refresh, negotiation, recovery | BullMQ with stable job IDs and retry policy |
| `evidence` | Reproducible traces with redacted prompts/tool decisions | PostgreSQL metadata, encrypted object store where necessary |

Proposed source structure: `src/modules/{commerce,catalog,negotiation,budgets}`, `src/integrations/{paypal,ebay,channel3}`, `src/agents/{buyer,seller,tools}`, `src/workers/commerce`, `tests/{unit,property,contract,integration,e2e,redteam}`. Provider APIs never enter an LLM's unrestricted network context.

## 3. Intent, lower/upper caps, and immutable constraints

Ask for currency, quantity, maximum **all-in** spend, intended negotiation opening offer, desired target price, acceptable condition/variants, delivery region and deadline, allowed platforms, and whether seller contact may be attempted on each allowed integration. The address is stored securely and sent to only the selected merchant when needed for a reliable quote. A postal region can be used for early estimates.

Do not interpret a lower cap as permission to waste money. Expose these distinct fields:

- `max_total`: hard maximum payable for this intent, including item price, tax, shipping, disclosed WalPay platform charges, required provider fees and executable conversion cost.
- `opening_offer`: user's allowed opening bargaining amount; it is a negotiation strategy input, not a payment minimum.
- `target_total`: soft savings goal. Missing it does not grant an agent permission to exceed `max_total`.
- `min_offer_to_send`: optional user-imposed floor on outbound bargaining messages; accepting a genuine cheaper offer remains allowed after a quality/fraud check.
- `min_acceptable_quality`: condition, authenticity and functional constraints. A lower price is never a reason to silently relax these.
- `currency_policy`: `SAME_CURRENCY_ONLY` for the first USD sandbox payment; capability-gated exact-FX paths follow the global currency plan. Global discovery and primary USD reporting retain every original-currency amount. An FX estimate is informational; another-currency purchase requires an unexpired executable provider quote, bounded final payable currency amount, disclosed platform/provider charges, fresh approval and actual conversion-product access. A USD report conversion is never payment authority. See `15-global-currency-and-fx.md`.

Validate `0 <= min_offer_to_send <= opening_offer <= max_total` and `target_total <= max_total` when present. If the requested fields conflict, reject with named fields before starting tools. No model can amend these constraints. A user amendment creates `version + 1`, cancels unused proposals, and invalidates approval challenges.

```http
POST /v1/shopping-missions
Idempotency-Key: <client-generated-key>
{
  "query": "Refurbished noise-cancelling headphones with warranty",
  "quantity": 1,
  "constraints": {
    "currency": "USD",
    "max_total": {"currency":"USD","amount_minor":"15000"},
    "opening_offer": {"currency":"USD","amount_minor":"10500"},
    "target_total": {"currency":"USD","amount_minor":"13000"},
    "min_offer_to_send": {"currency":"USD","amount_minor":"9000"},
    "condition": ["NEW","CERTIFIED_REFURBISHED"],
    "warranty_months_min": 6,
    "delivery_country": "US",
    "latest_delivery_date": "2026-11-01",
    "platforms": ["WALPAY_STORE","CHANNEL3_DISCOVERY"],
    "max_negotiation_rounds": 4,
    "currency_policy": "SAME_CURRENCY_ONLY"
  },
  "consent": {"allow_negotiation":true,"allow_checkout_preparation":true}
}
```

Return `202` under `{data,request_id}` with `data.id`, `status: "queued"`, `workflow_state: "DISCOVERY_QUEUED"`, `version: 1`, `capability_results`, `operation_id` and `mode: "sandbox"`; provide an authenticated operation/events reference. Consent for discovery/negotiation is separate from approving a final transaction. `consent.allow_checkout_preparation` authorizes preparation, not capture. Workflow states below belong to `workflow_state`; they do not replace generic operation `status` or payment state.

Common errors: `422 CAP_CONFLICT`, `422 UNSUPPORTED_CURRENCY`, `409 CAPABILITY_UNAVAILABLE`, `403 CONSENT_REQUIRED`, `429 AGENT_QUOTA_EXCEEDED`. Return a stable code, human-readable localized message, field pointers, request ID and retriable flag using the common error envelope.

## 4. Data model and constraints

| Table | Important fields / constraints |
| --- | --- |
| `commerce_intents` | `id, tenant_id, user_id, revision, state, constraints_json, expires_at, cancelled_at`; owner-scoped reads/writes |
| `catalog_products` | `id, merchant_id, title, category, active, prohibited_goods_status`; merchant owns catalog |
| `catalog_variants` | `id, product_id, sku, currency, listed_minor, stock_on_hand, stock_reserved, version`; stock counts nonnegative |
| `seller_pricing_policies` | `variant_id, version, floor_minor, max_discount_bps, valid_until, max_rounds`; buyer agent cannot read secret floor |
| `product_candidates` | `intent_id, source, source_item_id, variant_fingerprint, raw_snapshot_hash, observed_at, listing_url, capability_snapshot`; uniqueness per intent/source/item/variant |
| `commerce_quotes` | Immutable exact components, currency, merchant/payee ID, inventory hold, policy version, validity, canonical hash, `binding=false` by default |
| `negotiation_sessions` | `intent_id, seller_id, state, round, policy_revision, deadline, max_rounds` |
| `negotiation_messages` | `session_id, seq, type, offer_minor, currency, quote_id, message_hash, prior_hash`; unique `(session_id, seq)` |
| `checkout_proposals` | `intent_id, revision, quote_id, digest, state, expires_at, payment_attempt_id`; one immutable price/merchant/currency tuple |
| `budget_windows` | `user_id, currency, window_start, window_end, cap_minor, spent_minor, reserved_minor, version`; unique policy/window/currency |
| `budget_reservations` | `proposal_id, window_id, amount_minor, state, expires_at, provider_operation_state`; unique active proposal reservation |
| `agent_runs` | Purpose, model/version, tool schema version, token/time cost, decision summary, trace ID; redact PII |
| `agent_tool_calls` | Action, trusted actor, resource scope, input hash, policy decision, output hash, status; no credential material |

Use `numeric(30,0)` or validated bounded `bigint` for minor-unit storage, one shared upper bound; map to `BigInt` in TypeScript. Database checks require currency-consistent components and `sum(components) == total`. A signed or hashed quote does not prove a third-party price is correct; fetch authoritative data before payment.

## 5. Deterministic negotiation workflow

1. Persist the immutable intent revision and outbox event. Worker claims the job with a lease.
2. Run own-store and allowed discovery adapters within a total timeout. Normalize variants, availability, product condition and data freshness. Do not merge similarly named products into one SKU.
3. Filter hard constraints before ranking. Rank by all-in total, quality evidence, warranty and delivery fit. Affiliate commissions cannot influence ranking without disclosed user choice.
4. Build candidate summaries without buyer address, private caps, seller secret floor, or tokens in the prompt. Private user caps remain in the policy process; model sees only inputs necessary to recommend a candidate.
5. Obtain a negotiation session only from an adapter advertising `NEGOTIATION_NONBINDING`. Apply explicit seller policy for the WalPay store. Browser mode may negotiate through a permitted selected seller-chat surface under plan 16; each external send has an exact one-use permit from direct local user approval or the optional finite standing mandate in section 7.1 of that plan. Use mocked marketplace fixtures for unsupported integrations, visibly tagged `SIMULATED` in API evidence. A missing API never triggers an automatic browser bypass.
6. Buyer agent proposes a bounded next offer. Deterministic validation checks currency, permitted item, number of rounds, monotonic concession policy, per-message rate limit, outgoing offer floor and upper total estimate.
7. Seller agent may explain and propose discounts. An authoritative seller-policy engine validates discount/floor/stock and issues quotes. The LLM cannot set arbitrary discounts or expose another seller's pricing.
8. Stop on acceptable quote, deadline, cap exhaustion, no improvement for two rounds, seller rejection, revoked consent, unavailable adapter, prompt attack, or max rounds. A stop is an ordinary state with a clear reason, not unlimited retry.
9. Refresh stock, merchant identity, tax, shipping, fees and quote expiry. If any mandatory component is unknown, state is `NEEDS_EXACT_QUOTE`; an estimate cannot become a payable proposal.
10. Reserve budget and stock atomically where owned locally, issue immutable final proposal, await step-up plus PayPal approval. User can reject without payment.

`ACCEPT_QUOTE` means the buyer is willing to review that quote; it is not an external binding purchase. APIs that make binding offers or auction bids require a distinct explicit authorization path and are disabled for MVP. Do not rename a binding bid as a harmless negotiation tool.

Proposed state machine:

```text
DISCOVERY_QUEUED -> DISCOVERING -> CANDIDATES_READY -> NEGOTIATING
 -> NEEDS_EXACT_QUOTE -> PROPOSAL_READY -> USER_APPROVAL_REQUIRED
 -> PROVIDER_APPROVAL_REQUIRED -> CAPTURE_QUEUED -> CAPTURE_IN_FLIGHT
 -> PAYMENT_CONFIRMED -> FULFILLMENT_PENDING -> COMPLETED

Nonterminal waits: USER_INPUT_REQUIRED, REQUOTE_REQUIRED, PROVIDER_STATE_UNKNOWN
Terminal alternatives: NO_MATCH, DECLINED, CANCELLED, EXPIRED, FAILED
```

Cancellation racing with an in-flight provider request enters `CANCEL_REQUESTED`, not instant `CANCELLED`. Resolve the provider outcome; if capture succeeded, show paid and start a permitted refund workflow. Never erase the payment.

## 6. Agent-to-agent protocol for owned/consenting merchants

This is a WalPay proposal protocol, not an invented PayPal/eBay/Meta standard. Future ACP/UCP/A2A interoperability is a separate adapter requiring its own conformance and provider eligibility. Do not implement a new payment rail.

```http
POST /v1/merchant-negotiations/{session_id}/messages
{
  "protocol":"walpay.negotiation.v1",
  "message_id":"<uuidv7>",
  "sequence":3,
  "in_reply_to":"<uuidv7>",
  "type":"COUNTER_OFFER",
  "item":{"merchant_id":"<uuidv7>","variant_id":"<uuidv7>","quantity":1},
  "offer":{"currency":"USD","amount_minor":"11500"},
  "scope":"ITEM_PRICE_ONLY",
  "quote_requested":true,
  "valid_until":"2026-10-16T16:00:00Z",
  "constraints_digest":"sha256:<digest>",
  "prior_message_hash":"sha256:<digest>"
}
```

Allowed types: `OFFER`, `COUNTER_OFFER`, `QUOTE`, `DECLINE`, `END`. `QUOTE` includes an immutable quote reference and exact component breakdown. Free text is optional explanatory data with a bounded length; it cannot add terms, authorize payment or alter amounts. Offers explicitly identify whether price is item-only or all-in. Require merchant-scoped OAuth credentials/mTLS for partner agents, session allowlists, signed request timestamp/body hash where deployed, bounded clock skew, nonce replay protection and idempotency. Do not accept remote identity from a user-controlled `merchant_id` alone.

Lock session row while appending a message; require expected sequence/revision and unique message ID; replay identical input returns original result; conflicting replay is `409`. Reject after expiry, wrong SKU, cross-currency response, signature failure, invalid sequence, cap breach or revoked seller capability. User's upper cap stays private; seller gets an offer, not the buyer's full budget.

## 7. Budget reservation and checkout correctness

Two caps apply independently: intent maximum and user-configured aggregate policy for the period. A PostgreSQL reservation represents **permission budget**, not money held in a WalPay wallet. There is no custody, escrow, balance debit, or spendable WalPay credit.

For each currency/window: `spent_minor + reserved_minor <= cap_minor`. A payment must also satisfy `proposal.total <= intent.max_total`. Reserve the exact all-in total; uncertainty blocks payment rather than relying on an arbitrary contingency. Use one currency for MVP. If user has no aggregate policy, still enforce the intent and per-intent active proposal uniqueness.

```ts
// Domain sketch: actual implementation uses a transaction helper and bounded BigInt parser.
async function reserveForProposal(tx: DbTx, p: Proposal) {
  const intent = await tx.lockIntent(p.intentId); // SELECT ... FOR UPDATE
  assertOwnerRevisionAndLiveConsent(intent, p);
  assertExactLiveQuote(p.quote, p.digest);
  const total = parseMinor(p.quote.total.amount_minor);
  if (total > parseMinor(intent.maxTotal.amount_minor)) throw capExceeded();
  // Lock all applicable aggregate windows in stable ID order to avoid deadlocks.
  const windows = await tx.lockBudgetWindowsInOrder(intent.userId, p.quote.currency);
  const existing = await tx.findReservationByProposal(p.id);
  if (existing) return existing; // Unique key protects retry races.
  for (const w of windows) {
    if (w.spent + w.reserved + total > w.cap) throw budgetUnavailable(w.id);
  }
  await tx.reserveInventory(p.quote.inventoryHold); // Owned-store only.
  const r = await tx.insertReservation(p.id, total, "RESERVED");
  for (const w of windows) await tx.incrementReserved(w.id, total);
  await tx.insertOutbox("commerce.proposal.reserved", {proposal_id: p.id});
  return r;
}
```

At capture dispatch, lock proposal/reservation, recheck consent, quote/order/merchant/currency/total digest and deadlines, persist a stable provider request ID and `IN_FLIGHT`, commit, then call provider outside the database transaction. A timeout is `UNKNOWN`, and the reservation remains consumed until reconciliation establishes the outcome. Do not release because a BullMQ lease, quote, UI, or ordinary reservation TTL expired after dispatch.

Before dispatch, expiry releases only a reservation known never to have caused a provider side effect; cancel an associated unused provider order where supported and reconcile races. After a confirmed capture, atomically move the amount from reserved to spent once and emit fulfillment. Refunds affect reporting but do not automatically replenish a spending policy unless an explicit versioned policy says so. Rollovers retain in-flight reservations in the window in which they were authorized. An old intent cannot escape a new cap by crossing midnight.

Capture is not replayed with new provider idempotency IDs after an ambiguous response. Read provider status using the stored order/attempt ID and reconcile. Webhooks and synchronous responses converge on a unique provider capture key and transition guard. Database outbox/inbox provides retryable delivery, not a claim of distributed exactly-once execution.

## 8. Final proposal and approval contracts

```http
GET /v1/shopping-missions/{id}/proposals/{proposal_id}
{
  "data": {
    "id":"<uuidv7>",
    "mission_version":1,
    "state":"USER_APPROVAL_REQUIRED",
    "merchant":{"id":"<uuidv7>","name":"WalPay Demo Store","verified":true},
    "items":[{"variant_id":"<uuidv7>","quantity":1,"condition":"CERTIFIED_REFURBISHED"}],
    "breakdown": {
      "items":{"currency":"USD","amount_minor":"11500"},
      "shipping":{"currency":"USD","amount_minor":"500"},
      "tax":{"currency":"USD","amount_minor":"960"},
      "fees":{"currency":"USD","amount_minor":"0"},
      "discount":{"currency":"USD","amount_minor":"0"},
      "total":{"currency":"USD","amount_minor":"12960"}
    },
    "expires_at":"2026-10-16T16:05:00Z",
    "digest":"sha256:<canonical-terms-digest>",
    "required_actions":["WALPAY_STEP_UP","PAYPAL_BUYER_APPROVAL"],
    "mode":"sandbox"
  }
}
```

Create a shared payment intent referencing the immutable proposal using `POST /v1/payment-intents`. `POST /v1/payment-intents/{id}/authorization-options` returns a short-lived WebAuthn challenge bound to user, origin, proposal digest, amount, payee, revision and nonce. Platform biometrics may unlock a passkey on the device; the backend receives a signed assertion, never a fingerprint or face image. A fallback PIN verifies only WalPay step-up, with server-side rate limiting and recovery; it cannot replace PayPal approval.

`POST /v1/payment-intents/{id}/authorizations` accepts `{challenge_id, credential_response}` and records the step-up authorization after a transactional guard. The server loads the bound digest from its challenge/proposal records; a caller-supplied digest is never authorization evidence. `POST /v1/payment-intents/{id}/paypal-order` prepares its provider order and returns the required PayPal action. No client-supplied amount, recipient or order ID can override the stored proposal. Provider order creation and its credentials belong to the payment service, not the LLM.

`POST /v1/payment-intents/{id}/capture` is a guarded command after PayPal approval; it returns `202`, shared payment state `capture_pending`, and an attempt ID. Server fetches and validates provider state; a browser callback or return URL alone is not proof. Any price/address/shipping/payee change makes the old approval unusable and requires re-quote/re-approval. Do not use approval of proposal A to pay proposal B. Shopping workflow states above are separate from shared payment states (`draft`, `awaiting_user_approval`, `awaiting_provider_approval`, `approved`, `capture_pending`, `succeeded`, `failed`, `cancelled`, `expired`, `outcome_unknown`). Refunds have their own records/state.

`POST /v1/shopping-missions/{id}/cancel`, `POST /v1/shopping-missions/{id}/revisions`, `GET /v1/shopping-missions/{id}/events` and `GET /v1/shopping-missions/{id}/negotiation-summary` are owner scoped. Event streams use cursor IDs and reconnect; hide private seller policy and other buyers' messages.

PayPal's standard integration separates order preparation, buyer interaction and server capture; Orders/Payments references define provider operations. The extra WalPay step-up is an application control. [Standard Checkout integration](https://developer.paypal.com/platforms/checkout/standard/integrate), [Orders v2](https://developer.paypal.com/api/orders/v2), [Payments v2](https://developer.paypal.com/api/payments/v2), checked 2026-10-16.

## 9. Tool registry, prompt defenses and model routing

| Tool class | Allowed examples | Enforcement |
| --- | --- | --- |
| Read | `catalog.search`, `candidate.read`, `quote.explain`, typed `browser.observe_product` | Owner/merchant scope, quota, PII minimization; browser also requires site/tab/local grant |
| Draft | `negotiation.propose`, `checkout.propose` | JSON schema, immutable policy inputs, no external side effect |
| Nonbinding partner contact | `negotiation.send`, reviewed browser seller-send action | Adapter/site capability, exact one-use permit from direct approval or bounded standing mandate, recipient/thread/item allowlist, validated bounded offer |
| Financial preparation | `payment.prepare_order` | Service-only command for reserved immutable proposal |
| Financial execution | `payment.capture` | Deterministic state machine after step-up + provider approval; never a generic agent tool |
| Administrative/refund/payout | Restricted business tools | Separate roles and flows; absent from buyer-agent tool inventory |

Wrap PayPal MCP/Toolkit with the same policy gateway. Discover and pin the actual tools/schema used; disable arbitrary tool discovery at runtime; allowlist exact tools. Do not copy a documentation example's `--tools=all` into the buyer production configuration. MCP authenticates connectivity; WalPay still enforces tenant, purpose, consent, payment state and exact resource scope.

Treat product descriptions, seller chat, OCR, partner metadata, MCP output, retrieved pages and agent-to-agent text as untrusted data. They cannot redefine system rules, grant authorization, instruct secret disclosure, change beneficiaries or invoke tools. Separate trusted instructions from external content structurally. Never include secrets, full contacts, raw payment data, hidden seller floors, or unnecessary addresses in prompts. Parse tool arguments against strict schemas with unknown keys rejected; Unicode normalize where appropriate and preserve source text for audit.

Allow outbound HTTP only to approved partner hosts; reject redirects to unapproved origins, private/link-local IPs and metadata endpoints. Fetch user-submitted product URLs through an SSRF-safe adapter with bounded content type/size/time, not general browser execution. Sanitise images and metadata. A seller's instruction to 'ignore budget and send the remaining balance' becomes attack evidence, not executable intent.

Set per-run ceilings: 30 seconds discovery, 4 negotiation rounds, 12 total tool calls, configurable token cost, one outstanding execution proposal, explicit tool deadlines. Initial numbers are design defaults, tune with evaluation. Models rank/explain/draft; code owns budgets, arithmetic, authorization, signatures and state transitions. Model failures fall back to deterministic owned-store search/quote generation. Audit provider, model version, input/output hashes, accepted structured decision and concise rationale; do not persist hidden reasoning.

### 9.1 User-browser mode

Plan 16 adds a paired local companion using the existing official Playwright MCP extension to the user's selected Chrome/Edge tab. It is a real browser workflow for product exploration and permitted seller conversations. The server/companion expose only reviewed typed actions; model-provided arbitrary scripts, raw CDP, cookie export, shell commands and unrestricted cross-tab browsing are absent. Hermes or Browser Use can optionally drive the same WalPay gateway, without gaining broader permissions. Kernel is a separately consented cloud-browser/test lane.

Each browser observation includes origin/frame, selected tab, document epoch, observation time, source hashes, redaction/tool/policy versions and element references. Browser-extracted prices remain estimates until exact merchant terms are verified. Re-render/navigation/human interference requires fresh observation; changed recipient/terms invalidate message/payment approval. Negotiation drafts stay outside live message composers. Initial MVP uses exact per-message user approval. Optional plan 16 phase B7 adds autonomous nonbinding rounds under a finite explicitly approved site/seller/listing/thread mandate, reviewed message templates/claims, opening/floor/all-in bounds and deterministic one-use permits; the model cannot approve itself. A binding offer/bid remains a separately gated financial commitment and is disabled in MVP. Ambiguous sends pause for reconciliation, without blind repeats. Site-policy entitlement and local user consent both apply; lack of an API does not grant an automatic UI workaround.

The browser can prepare allowed carts and hand off checkout, while the user performs provider login/approval. Own-store financial truth still comes from its authoritative quote, PayPal order/capture and verified events. A different merchant's successful browser checkout cannot be represented as a capture made by WalPay's merchant credentials. The full connection lifecycle, action schemas, cancellation/reconnect logic and BRO test/checklist suite live in [plan 16](16-user-browser-agent-and-marketplace-automation.md).

## 10. Own-store responsibilities

- Merchant catalog APIs: `POST /v1/merchants/{id}/products`, `POST /.../variants`, `PATCH /.../pricing-policies` with business roles, expected version, audit, prohibited-item checks and idempotency. A merchant cannot edit another merchant's catalog.
- Catalog search cannot advertise out-of-stock variants as guaranteed. Search freshness is informational; payment requires an inventory hold with expiry and quantity checks.
- Stock hold must cover payment handoff duration or force refresh before capture. An in-flight capture preserves stock allocation until resolved. If capture succeeds after inventory becomes impossible, record fulfillment exception and refund; never silently substitute.
- Seller price floor/discount policy is versioned and not sent to buyer models. Deterministic seller tests prove no response below the policy floor, no aggregate discount beyond policy, and no discount reuse across variants.
- Order fulfillment is an outbox consumer triggered by authoritative confirmed payment, with unique purchase/order keys, not by `proposal.accepted` or client success.
- Returns/disputes/refunds use the shared payment lifecycle. A partial refund updates order line accounting without recreating inventory until return policy confirms physical return.
- Multi-seller marketplace checkout is a later approved multiparty product; do not accept all funds into WalPay and manufacture an escrow/split wallet.

## 11. Implementation phases and acceptance gates

| Phase | Work | Exit evidence |
| --- | --- | --- |
| C0: capability proof | Inventory sandbox credentials, tool schemas, partner permissions, currencies and country rules | Capability manifest with `enabled=false` unless proven; no live actions |
| C1: deterministic store | Products/variants, inventory, quote arithmetic, pricing policy, stable fixtures | Real API can quote and hold stock without a model |
| C2: safe buyer/seller agents | Draft/validate loop, explicit protocol, time/round/token caps, redaction | Every offer obeys constraints; all external fixtures visibly simulated |
| C3: budget and approval | Transactional reservations, immutable digest, step-up integration | Concurrent proposals cannot exceed policy; changed terms invalidate approval |
| C4: PayPal sandbox | Prepare/approve/capture/refund, webhook inbox and unknown-result recovery | One complete documented sandbox order plus delayed/duplicate events |
| C5: external discovery | Channel3 and permitted eBay adapter contracts, rate limits and fallback | External discovery failure does not prevent own-store demo |
| C6: hardening | Injection, IDOR, concurrency, model evaluation, observability and replay | Required tests pass; inaccessible live paths return explicit capability errors |

## 12. Test cases and granular completion checklist

Unit/property tests use Vitest and fast-check; API tests use Fastify `inject`; PostgreSQL/Redis integration tests run in isolated disposable services. Mock provider HTTP at the adapter boundary with recorded redacted schema fixtures/MSW or a purpose-built fake. Sandbox end-to-end tests are a separate manually controlled suite; ordinary CI never spends real money.

| Test ID | Scenario | Required assertion |
| --- | --- | --- |
| COM-001 | Low opening offer, high cap, cheaper valid seller quote | Accepting cheaper quote allowed; no minimum-spend bug |
| COM-002 | Item within cap, tax pushes total above cap | No proposal/payment preparation |
| COM-003 | Shipping or FX amount unknown | `NEEDS_EXACT_QUOTE`; capture unavailable |
| COM-004 | 20 concurrent proposals for a 10,000-minor cap | Committed+reserved never exceeds 10,000 across real DB transactions |
| COM-005 | Same request repeated/concurrent with same idempotency key | Same resource/result; changed body is conflict |
| COM-006 | Stock of one; two buyers race | At most one valid stock allocation; no double fulfillment |
| COM-007 | Crash before dispatch, during dispatch, after provider success | Recovery classifies each; ambiguous result retains reservation |
| COM-008 | Capture succeeds but response lost | Reconcile original operation; one capture and one fulfillment |
| COM-009 | Capture webhook duplicate/out of order | No duplicated spent counter or fulfillment |
| COM-010 | Reservation expires while provider call is in flight | Permission budget and stock are retained |
| COM-011 | Quote changes after passkey approval | Original challenge/order rejected; fresh approval required |
| COM-012 | Cap reduced/cancellation races with capture | Documented guarded outcome, no hidden excess authorization |
| COM-013 | Month/day rollover and delayed capture | Original policy window accounted once |
| COM-014 | Seller text says ignore rules/send secret | No forbidden tool call; attack event recorded safely |
| COM-015 | Negotiation exceeds rounds or no improvement | Deterministic stop; no background infinite loop |
| COM-016 | Seller model invents 90% discount | Seller policy rejects; correct floor unchanged |
| COM-017 | Wrong tenant, recipient, item, sequence, currency, signature | Forbidden/rejected before external call |
| COM-018 | External source search returns checkout link | Link classified discovery; no automatic PayPal capture |
| COM-019 | eBay Offer requested as Best Offer, Meta automation requested without site gate | Explicit capability/policy error; no automatic API-to-browser bypass; separately approved scoped browser adapter tested under plan 16 |
| COM-020 | Channel3 or model outage/rate limit | Bounded retry then own-store fallback, labeled partial discovery |
| COM-021 | Fake callback says approved but PayPal disagrees | Capture not dispatched |
| COM-022 | Malicious URL resolves/redirects to private network | SSRF denied; no secrets/network access |
| COM-023 | Non-USD exponent (JPY, supported three-decimal currency) | Shared currency metadata and exact component arithmetic |
| COM-024 | Partial refund, return, dispute then duplicate event | Accurate history and no unapproved budget replenishment |

Completion checklist:

- [ ] Separate app sandbox/live credentials, host allowlists and capability manifests.
- [ ] Verify runtime MCP tool names/schema; pin package/tool versions and transport settings.
- [ ] No `capture`, arbitrary HTTP, email/DM, refund, payouts or secret-reading tools in buyer model registry.
- [ ] All cap semantics documented in OpenAPI examples; unknown/malformed monetary strings rejected.
- [ ] Own-store seller roles, pricing policy version, inventory hold and exact quote tested.
- [ ] Source/variant/currency/country/condition/warranty constraints persist through every handoff.
- [ ] Every external result includes source, freshness, mode, capabilities and verified merchant identity where available.
- [ ] Agent-to-agent offers are explicitly nonbinding; binding bids remain disabled.
- [ ] Offer sequencing/replay protection and all message limits implemented.
- [ ] Canonical proposal serialization and digest test vectors fixed.
- [ ] Aggregate budget windows and unique reservation constraints created by migration.
- [ ] No Redis-only lock used for financial correctness.
- [ ] Provider timeout recovery, cancellation race and TTL rules implemented and fault tested.
- [ ] Address/terms changes invalidate challenges and require fresh exact quote.
- [ ] User action and provider approval are independently verified and audit linked.
- [ ] Payment confirmation alone triggers fulfillment, exactly once per local unique key.
- [ ] Negotiation trace redacts private caps, tokens and seller floors; trace retention configured.
- [ ] Injection/SSRF/IDOR test corpus executed with deterministic pass criteria.
- [ ] Sandbox demo includes one actual successful capture, one rejected cap, one price-change reapproval and one duplicate webhook.
- [ ] Demo distinguishes `SIMULATED`, `SANDBOX`, `APPROVAL_GATED` and `UNSUPPORTED` without implying live permissions.
- [ ] External integration outages leave the deterministic own-store path usable.
