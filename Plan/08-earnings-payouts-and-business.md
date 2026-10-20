# Earnings, partner-triggered payouts, and merchant business tools

Planning baseline: 16 October 2026. Proposed backend design; no partner integration or PayPal approval is claimed to exist. Read [payment reliability](04-paypal-payments-and-wallet.md), [domain and ledger](03-domain-data-and-ledger.md), [security](09-security-identity-and-authorization.md), and [global currency and FX](15-global-currency-and-fx.md) before implementation.

The public route register, money grammar, `simulation/sandbox/live` modes, uppercase problem codes and versioned event conventions in 06 are authoritative. Named commands below explicitly labelled **internal planned extensions** have no public URL yet; expose them only after adding complete contracts to 06/OpenAPI. Provider `/v1` paths are always explicitly labelled PayPal paths and are not WalPay routes.

## 1. Product outcome and funded-money boundaries

Freelancers, creators, gig workers and merchants should be able to identify what they earned, who owes it, which work generated it, what was deducted, and whether PayPal has actually delivered it. An approved task creates a contractual earnings obligation; it does not create funds, prove the partner has funded a payout, or prove payment is delivered.

Distinguish these money paths:

| Path | Who receives or funds money | Demo / release classification |
| --- | --- | --- |
| WalPay's own store | WalPay's first-party merchant account receives customer Checkout capture | Core real sandbox flow |
| Creator tip or freelancer invoice issued by the actual seller | Seller merchant receives the buyer's approved PayPal payment | First-party seller sandbox integration, or approved platform delegation for outside sellers |
| Partner pays its contractors | Eligible partner-owned PayPal business funding account sends Payouts | Public sandbox testing where enabled; live Payouts access, funding and country restrictions apply |
| WalPay pays its own contracted demo workers | WalPay's eligible payout sender uses its own business funds | Truthfully labelled sandbox business-payout demonstration |
| Marketplace buyer pays seller with platform commission | Approved platform integration; seller is payee; fee follows approved arrangement | Sandbox exploration; live partner approval, onboarding and feature permissions required |
| Unconnected real gig platform pays a worker | WalPay may import a statement or expose an adapter contract | Fixture/manual import until the real partner grants API/webhook access |
| Arbitrary friends exchange or convert money | No authority established by these merchant APIs | Future provider/compliance-gated peer-money corridor; no generic P2P workaround |

Never collect a buyer's money into the WalPay store account and automatically forward it to an unrelated seller merely to avoid marketplace onboarding. Never represent a local earnings row as a wallet balance. Do not reuse marketplace partner-fee collection accounts as the own-store merchant or payout funding account without explicit provider confirmation that the account arrangement permits it.

PayPal's Payouts product requires access approval, a business account, completed account checks, and available funding; sandbox experimentation can start before live access is approved. [Payouts prerequisites](https://developer.paypal.com/payouts/use-payouts/overview)

India is listed as “receive and withdraw” in the Payouts country matrix, rather than a sending market. A genuine Indian account must not be treated as a live Payouts funding source. [Payouts country support](https://developer.paypal.com/payouts/supported-features/)

## 2. Backend services and data ownership

| Module | Owned records | Events emitted |
| --- | --- | --- |
| `partner-connectors` | Partner installation, credentials reference, event inbox, task snapshots, cursors | `partner.task.approval_verified`, `partner.task.revoked` |
| `earnings` | Immutable earnings accruals and adjustments, payer/payee relation, contract version | `earning.accrued`, `earning.adjusted` |
| `payout-policy` | Mandates, per-run/day limits, available funding observations, holds | `payout.eligible`, `payout.blocked` |
| `payout-orchestrator` | Payout obligations, batches, attempts, allocations and durable operations | `payout.requested`, `payout.delivered`, `payout.failed` |
| `business-billing` | Invoice drafts, provider invoice references, subscription plans and memberships | `invoice.sent`, `invoice.payment_verified`, `membership.changed` |
| `merchant-onboarding` | Referral IDs, verified merchant IDs, granted permissions and capability snapshots | `merchant.capabilities_changed`, `merchant.consent_revoked` |
| `earnings-reporting` | Query projections, partner attribution, currency summaries and exports | Rebuildable materializations only |

Use the same PostgreSQL/outbox, Fastify REST, BullMQ and provider adapter infrastructure as checkout. The payout worker runs with a narrower secret set than the HTTP API. Reporting and AI assistants can query sanitized projections; neither receives payout credentials or authority to mark payment delivered.

## 3. Core schema and invariants

All local identifiers are UUIDv7. Provider IDs and external partner IDs remain opaque strings. Every table includes tenant/environment scoping where applicable, timestamps and optimistic versioning.

| Table | Key fields and constraints |
| --- | --- |
| `partner_installations` | Tenant, partner type, fixed API base, credential secret reference, allowed event types, status, last verified, signing-key version |
| `partner_event_inbox` | Installation ID + external event ID unique; raw digest; signature metadata; source task/version; processing state |
| `partner_tasks` | Installation ID + external task ID unique; current verified version; worker mapping; contract ID; approval evidence reference |
| `earning_accruals` | Payee, payer legal entity/account, task/version or order/capture reference, currency, gross minor, fee minor, net due minor; immutable contract snapshot |
| `earning_adjustments` | Original accrual ID; credit/debit direction; positive minor amount; reason; external correction reference unique |
| `payout_destinations` | User, provider, encrypted receiver, receiver type, destination version, ownership evidence, verification time, consent, cooldown_until |
| `payout_mandates` | Payer, authorized integration, allowed recipients/countries/currencies, task source, per-item/day limits, expiry, approving actor, revocation |
| `payout_obligations` | Payee/payer, currency, positive amount due, source accruals, status, hold reason, provider mode; no spendable balance field |
| `payout_allocations` | Obligation ID + accrual ID; amount minor; guard allocation sum against source remaining amount under lock |
| `payout_batches` | Payer funding account, environment, currency, local operation UUID, stable sender_batch_id, body hash, provider batch ID, provider state |
| `payout_items` | Batch ID + stable sender_item_id unique; destination/version, amount; provider item ID unique; raw status; normalized status |
| `payout_attempts` | Logical obligation, batch/item, request UUID, attempt outcome, error/debug ID, started/completed timestamps |
| `funding_observations` | Payer/provider account, currency, observed amount if authorized and available, source, freshness; never caller-declared proof |
| `business_invoices` | Seller, invoice number unique per seller, currency, immutable sent version, provider invoice ID unique, amount/paid/refunded snapshots |
| `membership_plans` | Seller, local plan/version, provider product/plan IDs, one currency, billing/cancellation policy hash |
| `memberships` | User/seller, provider subscription ID unique, provider status, entitlement status, paid-through evidence, last verified |

Constraints to implement transactionally:

1. A verified task approval produces one earning accrual for the defined `(installation, task, milestone, payable_revision)` business key. A new arbitrary event ID cannot create a second payment for the same work.
2. Payout allocations cannot exceed approved net due after signed adjustments; no allocation crosses payer, currency or tenant.
3. A provider item belongs to one logical obligation allocation. Distinct attempts cannot concurrently spend the same remaining obligation.
4. A changed payout destination creates a new version and requires reauthorization/cooldown; queued work cannot silently switch recipient.
5. Amounts use exact minor-unit strings at the API and integer-safe arithmetic. `net_due = gross − contractually permitted deductions`; fees cannot be negative or exceed authorized gross.
6. A payout result does not alter the original accrual. Delivered, returned, reversed and refunded facts add journal entries and update projections.
7. Only an authenticated, verified provider item result or approved reconciliation evidence can count as delivered.
8. An earning whose source is an already direct-paid merchant capture must not also produce a Payouts obligation for the same money.

## 4. Partner connector protocol

Create a small adapter SDK with a common contract. Real connectors use documented partner APIs and separately granted scopes; do not promise that Fiverr, Upwork, YouTube, delivery services or other named platforms expose task approval or instant payout APIs to WalPay.

```ts
interface PartnerConnector {
  verifyDelivery(rawBody: Buffer, headers: Record<string, string>): Promise<VerifiedEvent>;
  fetchTask(externalTaskId: string): Promise<VerifiedTaskSnapshot>;
  mapWorker(externalWorkerId: string): Promise<ConsentedPayeeMapping>;
  reconcileSince(cursor?: string): Promise<TaskChangePage>;
  getCapabilities(): PartnerCapabilities;
}
```

Proposed connector modes:

- `real`: authenticated partner webhook/API, documented capabilities and retained contract evidence.
- `sandbox_partner`: WalPay-operated demo partner with signed approval API and fixture tasks; visible as a simulated partner, with real PayPal sandbox settlement where available.
- `import_only`: user-authorized CSV/statement ingestion; informs earnings reporting but cannot auto-trigger payouts.
- `manual_verified`: finance operator records approved work with evidence and required payer authorization; distinct audit trail.

### 4.1 Signed approval intake

```http
POST /v1/partner-events/0199e6c0-2000-7000-8000-000000000201
Content-Type: application/json
X-WalPay-Key-Id: demo-2026-10
X-WalPay-Timestamp: 1792166400
X-WalPay-Signature: v1=<hex-hmac-sha256>

{
  "event_id": "task-event-3481",
  "type": "task.approved",
  "occurred_at": "2026-10-16T16:00:00Z",
  "task_id": "gig-782",
  "task_version": 4,
  "worker_id": "partner-worker-117",
  "contract_id": "contract-2026-09",
  "milestone_id": "delivery-final",
  "amount": { "currency": "USD", "amount_minor": "2500" },
  "approval": { "actor_id": "partner-supervisor-52", "reference": "approval-718" }
}
```

All header names here belong to the proposed WalPay demo connector, not an existing external partner. Each actual partner's signature specification replaces this verifier in its own adapter.

Proposed signature message: UTF-8 `timestamp + "." + exact_raw_body`. Verify the active/rotating key identified by the configured installation and key ID, require a bounded timestamp skew for this controlled protocol, compare fixed-length signatures with constant-time comparison, then persist deduplication and digest. Do not let body values select a signing secret or payer account. The canonical route's `partnerId` maps to the authenticated tenant installation; it is not an arbitrary payer selector.

```json
{
  "event_id": "task-event-3481",
  "status": "accepted",
  "verification": "pending_partner_lookup",
  "receipt_id": "0199e6c0-2000-7000-8000-000000000202",
  "mode": "sandbox",
  "request_id": "0199e6c0-2000-7000-8000-000000000299"
}
```

Return `202` after SQL inbox/outbox commit. A valid delivery is evidence of an event, not sufficient evidence of an earned amount. Worker obtains the current task from a fixed, allowlisted partner endpoint, verifies worker/contract/milestone/amount/version and approval authority, then compares against prior task events. Never GET an arbitrary “evidence URL” supplied inside the event.

If a real partner lacks a GET API, define a documented signed-event trust model with contractual event authority and correction semantics; do not silently skip verification. If neither authority is available, keep the row `unverified` and prohibit automated payout. Duplicate identical delivery returns the original receipt; changed content under the same event ID raises a security case. Lower task versions do not overwrite newer state. Cancellation/revocation events carry adjustment evidence and may place an unpaid obligation on hold.

### 4.2 Verified approval to durable obligation

```ts
async function accrueVerifiedTask(snapshot: VerifiedTaskSnapshot) {
  return db.transaction(async tx => {
    const task = await tx.lockPartnerTask(snapshot.installationId, snapshot.taskId);
    requireMonotonicVersion(task, snapshot.version);
    requireVerifiedApprover(snapshot);
    const contract = await tx.getActiveContract(snapshot.contractId);
    requirePartnerWorkerAndAmountMatch(snapshot, contract);
    const mapping = await tx.getConsentedPayeeMapping(snapshot.workerId);
    const uniqueWorkKey = payableWorkKey(snapshot, contract);
    const existing = await tx.findAccrual(uniqueWorkKey);
    if (existing) return existing;
    const earning = await tx.insertAccrual(exactContractCalculation(snapshot, contract, mapping));
    await tx.appendJournal(accrualJournal(earning)); // Verified obligation evidence, not settlement.
    await tx.insertOutbox({ type: "earning.accrued", earningId: earning.id });
    await tx.updateTaskEvidence(task, snapshot);
    return earning;
  });
}
```

The payout-policy worker evaluates the resulting obligation separately. A model may summarize evidence or flag anomalies; deterministic policy and payer authority govern whether payment may start.

Accrual journals use a separate obligation-evidence/accounting-rule type from provider settlement journals. A verified partner promise can establish a business claim but never provider clearing, delivered cash, or spendable funds. Canonical financial-fact uniqueness prevents different webhook/API observations of the same payout item from posting the same delivered amount twice.

## 5. Authority to pay and funding model

Automatic “pay when approved” means “queue an eligible payout immediately after verified approval under an existing payer mandate.” It does not promise instant availability or bypass provider review. Store a revocable payer mandate created by an authorized business administrator with step-up authentication; bind it to the partner installation, recipient category, currencies, maximum per item/day, contract versions and expiry. Start conservatively with an allowlisted set of sandbox recipients and a USD limit.

Supported implementation choices:

1. **Partner-owned funding:** The real payer configures its eligible PayPal Payouts integration in an approved deployment arrangement. The worker uses only the payer's authorized credential/delegation mechanism supported for that product. Do not assume seller Checkout consent also authorizes Payouts.
2. **WalPay own-business funding:** For WalPay's own obligations, use its separate eligible payout sender business account. This is the clean sandbox demo path for the controlled gig partner.
3. **Reporting only:** When funding authority, access or the corridor is missing, record approved earnings and expose the block reason. Do not claim an unconnected partner has paid.

Never ask users to paste personal PayPal passwords or broad master credentials. Production credential onboarding requires secure secret handling and a supported provider/business relationship; first-party sandbox developer credentials may be configured by the development team.

Payouts funding must cover principal and provider fees, with conversion costs where applicable. Exact provider fees are recorded from evidence, not hardcoded as a universal percentage. [Payouts fees and funding](https://developer.paypal.com/payouts/fees/)

The local funding reservation is an operational commitment for the payer account, not a customer wallet balance. Lock reservations by payer/currency during batch preparation. If no authorized real-time funding balance API is available, use conservative operator-supplied funding limits plus provider acceptance/reconciliation; clearly record that the balance is not verified. Insufficient funding blocks/retries after funding remediation, never borrows from another tenant or silently changes currency.

## 6. Recipient routing by WalPay ID, phone or email

WalPay handles and verified phone contacts are discovery/routing aliases. Resolve them to a consented, verified PayPal destination record, then freeze that destination version on the obligation/item. Do not treat a phone OTP as proof of ownership of a PayPal account. Where available, use consented PayPal identity evidence and destination confirmation. Any payout destination change requires renewed verification and configured cooldown/risk review.

PayPal Payouts accepts email and PayPal account identifiers; the current create-payout reference explicitly states that `PHONE` is unsupported in sandbox. Therefore, the real sandbox flow for a searched phone number resolves to a test recipient's verified `EMAIL` or `PAYPAL_ID`. Raw phone delivery remains a separately verified live corridor. [Create batch payout](https://developer.paypal.com/api/payments.payouts-batch/v1/payouts-post)

Canonical routing endpoint and internal planned destination commands:

| Route | Request / behavior | Response / checks |
| --- | --- | --- |
| `CreatePayoutDestination` — internal planned extension | Provider type, consented receiver and verification session | Masked destination, `verification_pending`; ownership proof and provider capabilities required before activation |
| `VerifyPayoutDestination` — internal planned extension | Verification evidence reference | Verified version or remediation; step-up policy |
| `POST /v1/recipients/resolve` | `{ "alias_type": "walpay_id", "alias": "mira.walpay", "purpose": "business_payout" }` | `200` opaque recipient ID, masked label, eligible modes; rate limits and discoverability consent |
| `DisablePayoutDestination` — internal planned extension | Owner; no unresolved attempts tied to active version | Disabled for future use; retain required transaction evidence |

Prevent directory enumeration: exact-match discovery, per-user/IP limits, opt-in contact lookup, no full address-book upload by default, no provider account existence oracle, masked display data, and a recipient confirmation screen contract for the caller. NFC/QR can convey an opaque WalPay recipient token; it does not authorize movement or prove the recipient is trustworthy.

## 7. Payout HTTP contracts

Canonical routes are proposed WalPay APIs; extension commands remain internal until their routes are registered in 06/OpenAPI. All user/business mutations require tenant authentication, scope checks and idempotency keys; version-dependent commands require `If-Match`.

| Route | Required authority | Success |
| --- | --- | --- |
| `GET /v1/earnings` | Current payee or permitted payer report role | `200` cursor page; filter by partner/date/currency/status |
| `GetEarningsSummary` — internal projection used by canonical earnings read | Payee/report role | Per-currency accrued, payable, pending, delivered, returned and deducted values |
| `CreatePayoutMandate` — internal planned extension | Payer finance admin + step-up | Versioned mandate; caps/expiry/recipients/connector |
| `RevokePayoutMandate` — internal planned extension | Payer finance admin | Future preparation disabled immediately; in-flight outcomes reconciled |
| `POST /v1/payouts` | Payer `payout:write`; approved obligations; step-up or applicable mandate | `202` stable operation and obligation status |
| `GET /v1/payouts/{id}` | Payee or payer scopes | `200` item result, source work attribution, provider evidence freshness |
| `CancelPayout` — internal planned extension | Payer finance role; provider item unclaimed and cancellable | Cancellation is not promised until provider confirms returned funds |
| `ReconcilePayout` — internal operations command | Operations role | GET-only provider reconciliation |

```http
POST /v1/payouts
Idempotency-Key: d52ef3e1-4802-44ea-a65b-91a02f3953dc

{
  "obligation_ids": ["0199e6c0-2000-7000-8000-000000000203"],
  "payout_destination_id": "0199e6c0-2000-7000-8000-000000000204",
  "destination_version": 2,
  "mandate_id": "0199e6c0-2000-7000-8000-000000000205"
}
```

The caller cannot set the principal, fee, funding account, receiver string or provider recipient type in this endpoint. The server derives all of them from verified obligations, contract and destination. A finance-admin manual override still needs an explicit audited adjustment/new obligation, not arbitrary payout JSON.

```json
{
  "id": "0199e6c0-2000-7000-8000-000000000206",
  "status": "queued",
  "amount": { "currency": "USD", "amount_minor": "2500" },
  "recipient": { "display_name": "Mira", "destination": "m***@example.test", "destination_version": 2 },
  "funding_status": "awaiting_provider_acceptance",
  "mode": "sandbox",
  "status_url": "/v1/payouts/0199e6c0-2000-7000-8000-000000000206",
  "version": 1,
  "request_id": "0199e6c0-2000-7000-8000-000000000298"
}
```

Errors include `409 DESTINATION_CHANGED`, `409 FUNDING_NOT_VERIFIED`, `409 APPROVAL_NOT_VERIFIED`, `403 PAYOUT_MANDATE_REVOKED`, and `422 PAYOUT_CORRIDOR_UNSUPPORTED`. A new conflicting action while an earlier operation is unresolved returns `409 OUTCOME_UNKNOWN`; the already accepted operation remains `202`/queryable with its original ID. Errors do not reveal full receiver addresses or credential details.

## 8. Provider batch adapter and idempotency

Batch by payer funding account, environment and currency; freeze recipients and amounts before dispatch. MVP limit: 50 items per batch, deliberately below provider limits to simplify reconciliation. Each item gets a durable UUIDv7 `sender_item_id`; each batch gets a UUIDv7 `sender_batch_id` and separate persisted provider request ID. Retain identifiers and transaction evidence for the finite documented payment/business retention policy, sufficiently longer than provider deduplication and required reconciliation/refund periods. Any longer legal hold has a named reason and review date; do not retain by convenience indefinitely.

Illustrative sandbox provider body:

```json
{
  "sender_batch_header": {
    "sender_batch_id": "0199e6c0-2000-7000-8000-000000000207",
    "email_subject": "Your WalPay contractor payout"
  },
  "items": [{
    "recipient_type": "EMAIL",
    "receiver": "sandbox-contractor@example.test",
    "amount": { "currency": "USD", "value": "25.00" },
    "sender_item_id": "0199e6c0-2000-7000-8000-000000000208",
    "note": "Approved milestone gig-782"
  }]
}
```

Provider call: POST **PayPal** `/v1/payments/payouts`; persist the returned provider batch ID. Use stable sender batch identifiers on ambiguous retries. PayPal documents duplicate batch protection for the preceding 30 days and recovery links for duplicate requests. This is not a global exactly-once guarantee; the local obligation/allocation constraints must outlive that window. [Payout sender-batch header](https://developer.paypal.com/api/payments.payouts-batch/v1/definitions/payout_sender_batch_header/)

```ts
async function preparePayoutBatch(candidateIds: string[], payer: Payer) {
  return db.transaction(async tx => {
    const obligations = await tx.lockEligibleObligations(candidateIds, payer.id);
    requireSameFundingAccountCurrencyAndEnvironment(obligations);
    await tx.lockFundingReservationKey(payer.id, obligations[0].currency);
    const mandate = await tx.lockAndValidateMandate(obligations);
    const destinations = await tx.lockVerifiedDestinationVersions(obligations);
    requireNoActiveAttemptOrUnresolvedOutcome(obligations);
    requireCorridorAndProductCapabilities(payer, destinations, obligations);
    const projectedCost = calculatePrincipalAndConservativeFeeReserve(obligations);
    await tx.reserveOperationalFunding(payer, projectedCost);
    await tx.reserveMandateDailyLimit(mandate, projectedCost.principal);
    const batch = await tx.insertFrozenBatch(obligations, destinations);
    await tx.insertOutbox({ type: "payout.batch.prepared", batchId: batch.id });
    return batch;
  });
}
```

Worker claims a database lease, rechecks revocation/capability state before first send, and sends the persisted body outside the transaction. If it dies or times out after PayPal accepts, recover the same batch operation; never immediately re-batch those obligations with new IDs. When a duplicate response links to an original batch, validate environment/account and query that known resource. Do not blindly follow arbitrary response URLs.

Apply the conservative mutation-error rule from 04: a post-send parse/schema/amount/receiver mismatch is an unknown/discrepancy case with reservations retained, never proof that no payout occurred. Only verified provider evidence proving rejection without financial effect permits definitive failure or a controlled later attempt. A batch/item observation is deduplicated by canonical provider financial fact, not merely by the delivery event ID.

Insufficient funds or account restriction are business blocks, not infinite retries. A definitive failed item may be eligible for a new controlled attempt after remediation, but only after its first item is conclusively failed/returned and the remaining obligation is recomputed. Unknown, pending, unclaimed and held items cannot be retried as new payments.

## 9. Payout and earning state machines

Earning settlement dimensions:

```text
task unverified -> approval_verified -> earning_accrued
earning_accrued -> payable | on_hold
payable -> allocated_to_payout -> partially_delivered | delivered
any accrual -> adjustment record                         (preserve original)
delivered -> returned/reversed evidence + reopened due  (if contract still owes it)
```

Obligation normal form:

```text
approved -> blocked | queued
queued -> submitting -> submitted
submitting -> outcome_unknown
submitted/outcome_unknown -> pending | unclaimed | held | delivered | failed | returned
unclaimed -> delivered | returned
held -> pending | delivered | failed | returned
delivered -> reversed | refunded                         (subsequent facts)
```

Preserve provider raw statuses without inventing unsupported values. Normalize known item status `SUCCESS` to `delivered`; `PENDING`, `UNCLAIMED`, `ONHOLD`, `BLOCKED`, `FAILED`, `RETURNED`, `REFUNDED` and `REVERSED` remain separately distinguishable. Unknown provider enum values go into reconciliation/quarantine, not delivered. [Payout item status reference](https://developer.paypal.com/api/payments.payouts-batch/v1/definitions/payout_batch_items_list)

The provider may return unclaimed payouts after 30 days; only a verified resulting status releases a local funding reservation or reopens an obligation. Store the expected check date, but do not declare returned funds solely from a local timer. [Payout processing](https://developer.paypal.com/payouts/payment-processing/)

Subscribe to the relevant batch and item events. A batch notification does not contain all item outcomes, so the worker GETs batch details with pagination and individual item resources as needed. Reuse the verified inbox pipeline from the payment document. Confirm each item's amount, receiver mapping, `sender_item_id` and account/environment before projecting delivery. [Batch payout webhook semantics](https://developer.paypal.com/api/rest/webhooks/event-names/)

Polling proposal: immediately after batch creation, then jittered 5 s / 15 s / 60 s / 5 min for recent transitional items, backing off for holds/unclaimed payments. A daily reconciliation sweep checks all nonterminal items and recent delivered items for later reversals within configured retention. Provider rate limits and operational budgets override the sample cadence.

## 10. Multi-currency earnings, conversion and platform charges

USD is the product's primary reporting and demo settlement currency. Retain every accrual, invoice, provider fee, payout and adjustment in its original currency. Return currency buckets; a USD equivalent total is an estimate with its rate source/time, never a payable combined balance. Read the dedicated [FX plan](15-global-currency-and-fx.md) for the canonical quote contract and corridor gates.

For any supported conversion, persist source amount/currency, target amount/currency, provider quote/reference if available, estimated or guaranteed classification, rate source, rate value/precision, provider fee, WalPay fee, net recipient amount, expiration and final provider settlement evidence. Store a separate fee policy version and fee payer. Require fresh approval if a promised total/net changes outside the authorized terms.

Three distinct fee arrangements:

1. Own-store pricing or an explicitly disclosed WalPay service invoice can include lawful business charges approved by the buyer. Record the charge transparently as part of the actual business transaction; do not disguise a personal transfer as a sale.
2. An approved marketplace may collect platform commissions using the enabled platform fee mechanism. The provider does not offer an unrestricted fee field for arbitrary P2P/FX transfers.
3. A future peer exchange/conversion market needs its own supported money-transfer/FX provider arrangement, corridor eligibility, compliance design and settlement safeguards. Plan it as a gated extension; ordinary Checkout/Payouts do not establish a generic currency-swap API or customer-to-customer exchange business.

PayPal platform fees must share the transaction currency; first-party integrations do not support this platform-fee feature. The partner fee account has special settlement/account requirements, so separate it from own-store collection and contractor payout funding. [Platform fee constraints](https://developer.paypal.com/platforms/overview)

Do not hardcode an exchange-rate spread, a provider conversion option, guaranteed FX expiry, or exact recipient net when the enabled API has not supplied it. A third-party market-rate feed may support comparisons and AI explanations; it cannot be passed off as a binding PayPal execution quote. Refund/cancellation accounting must distinguish original currency amounts, refund currency, provider-applied conversion, returned fees and nonreturned fees using actual evidence.

## 11. Freelancer invoices

First-party invoice integration starts in sandbox for a controlled business seller. Outside-seller invoice issuance requires the supported delegated/partner permissions, separate from identity login. Multi-currency means invoices in supported currencies with original-currency accounting; never place mixed settlement currencies inside one invoice or automatically charge a rate selected by an LLM.

Proposed routes:

| Route | Request / action | Backend safeguards |
| --- | --- | --- |
| `POST /v1/invoices` | Customer, currency, line items, tax/discount policy, due terms | Merchant role; exact totals; returns canonical `202` creation operation; draft is recorded separately from provider completion |
| `ReviseInvoiceDraft` — internal planned extension | Versioned draft edits | Optimistic version; sent versions immutable locally |
| `POST /v1/invoices/{id}/send` | Final draft version and optional authorized notification mode | `202` explicit merchant command; durable send operation; idempotency |
| `POST /v1/invoices/{id}/remind` | Reason/cadence | `202` merchant-scoped operation; rate limits; no model-spam loop |
| `POST /v1/invoices/{id}/cancel` | Reason | `202` eligible cancellation operation; paid invoice may need refund path |
| `GET /v1/invoices` | Scoped invoice records, state and payment allocations | Owner/customer access as applicable; sensitive data redacted |
| `GetInvoiceDetail` — internal planned extension | One invoice's full authorized projection | Expose a separate public detail URL only after registering it in 06/OpenAPI |

Provider adapter creates a draft via `POST /v2/invoicing/invoices`, records its provider ID, and sends via the separate send endpoint. Draft creation does not notify the customer. [Invoicing reference](https://developer.paypal.com/invoicing/reference/)

Illustrative create-draft provider request:

```json
{
  "detail": {
    "invoice_number": "WP-2026-0017",
    "currency_code": "USD",
    "note": "Approved design milestone",
    "payment_term": { "term_type": "NET_15" }
  },
  "invoicer": { "email_address": "sandbox-seller@example.test" },
  "primary_recipients": [{
    "billing_info": { "email_address": "sandbox-client@example.test" }
  }],
  "items": [{
    "name": "Design milestone",
    "quantity": "1",
    "unit_amount": { "currency_code": "USD", "value": "250.00" }
  }]
}
```

Merchant contact/tax details come from its verified account and invoice configuration. Do not let clients impersonate a different invoicer. Snapshot the exact sent draft and provider result; use the local operation ID and supported endpoint idempotency behavior. [Create invoice API](https://developer.paypal.com/api/invoicing/v2/invoices-create/)

`INVOICING.INVOICE.PAID` alone is not proof of a fully settled invoice: it may describe partial or pending payment. Re-fetch the invoice, inspect paid/outstanding amounts and payment evidence, then allocate collected revenue. `Record payment`/“mark paid” APIs can record an external payment; they do not move money. A manually recorded external payment is separately labelled and cannot masquerade as a verified PayPal capture. Invoice refunds and recorded external refunds must likewise use the correct provider operation and evidence source.

Maintain statuses such as draft, scheduled, sent, partially_paid, paid, overdue, cancelled and refunded as derived local business states; preserve provider states separately. Avoid auto-retrying an invoice send that may already have succeeded without checking current invoice state. Notification delivery is not payment receipt. Explicit send/reminder commands are product actions with audit/authorization; this planning task sends no real invoices or messages.

## 12. Creator tips and memberships

One-time tips are supported commercial creator payments using a real seller/payee checkout and buyer approval in an eligible corridor. Record creator, fan opt-in display preference, campaign/content attribution, gross, provider fee, platform fee where enabled, net and refund/dispute status. Do not classify an individual creator tip as a charitable donation or promise tax deductibility.

Membership subscriptions use a real provider product, a versioned plan in one currency, and a buyer-approved subscription. A local `membership.active = true` flag is not authority to bill. Products/plans are created by the seller service; server-side pricing cannot be replaced by client-supplied `plan_id` from another merchant. PayPal's first-party integration uses Catalog Products and Subscriptions APIs. [Subscriptions integration](https://developer.paypal.com/subscriptions/integrate/)

API proposal:

- `ProvisionMembershipPlan` — internal planned extension: seller scope, name, amount, currency, interval, trials and cancellation policy; versioned plan and provider provisioning state.
- `POST /v1/subscriptions`: user, local plan ID/version; returns canonical `202` setup operation, followed by subscription reference, pending state and provider approval action.
- `GetMembershipEntitlement` — internal read projection: provider state, entitlement state, next verification time and paid-through evidence. A dedicated public detail route must first be registered in 06/OpenAPI; current operation status uses `GET /v1/operations/{id}`.
- `POST /v1/subscriptions/{id}/cancel`: owner or authorized merchant, reason and effective entitlement policy; canonical `202` durable provider cancellation plus reconciliation.

PayPal provider sequence: `POST /v1/catalogs/products` → `POST /v1/billing/plans` → `POST /v1/billing/subscriptions` → buyer approval → verified subscription/transaction evidence. These are PayPal paths, not WalPay API routes. Use provider GET subscription and transaction queries to recover missing webhook evidence. Separate paid cycles from subscription activation; free trials can legitimately activate without a payment. [Subscriptions API](https://developer.paypal.com/api/subscriptions/v1)

Retain provider subscription statuses (`APPROVAL_PENDING`, `APPROVED`, `ACTIVE`, `SUSPENDED`, `CANCELLED`, `EXPIRED`) separately from access entitlement. A cancellation may stop renewal while a paid-through entitlement remains valid under the disclosed policy. Failed payment changes follow a deterministic grace/suspension policy; an AI assistant cannot silently reprice or extend billing. [Subscription status schema](https://developer.paypal.com/api/subscriptions/v1/definitions/subscription/)

Outside-merchant subscription facilitation is separately select-partner gated, and INR is explicitly unsupported in the platform subscription guide. Use USD first and verify seller/country eligibility; do not infer all merchant subscription capability from first-party sandbox access. [Platform subscriptions prerequisites](https://developer.paypal.com/platforms/subscriptions/integrate/)

## 13. Marketplace onboarding and automated platform commissions

Keep this extension behind a `multiparty_enabled` capability flag until the actual platform app and seller capabilities are verified. Merchant linking is different from consumer identity linking.

Internal planned onboarding commands, with public routes deferred until registered in 06/OpenAPI:

- `StartPayPalSellerOnboarding`: seller administrator starts an authorized referral operation; server requests only needed features and records consent evidence.
- `GetPayPalSellerOnboarding`: reports provider referral/onboarding state.
- `RefreshPayPalSellerOnboarding`: produces a fresh supported onboarding action when required.
- Provider callback: validates local correlation; queries seller integration status. Query parameters are not proof of completion.

Use the Partner Referrals API to create the onboarding referral and query merchant integration status for verified `payments_receivable`, email confirmation and required granted permissions/capabilities. Consent revocation events disable new operations and trigger reconciliation. [Seller onboarding before payment](https://developer.paypal.com/platforms/seller-onboarding/before-payment/)

Do not copy `legal_consents.granted=true` into a provider call without actual seller authorization to that action. Persist the applicable seller-consent terms/version and actor. PayPal controls account verification and risk review; WalPay cannot mark provider checks passed itself.

For an enabled marketplace fee, derive seller merchant ID and fee amount from a signed/versioned business contract and trusted quote. Example provider purchase-unit fragment:

```json
{
  "amount": { "currency_code": "USD", "value": "100.00" },
  "payee": { "merchant_id": "SELLER_MERCHANT_ID" },
  "payment_instruction": {
    "disbursement_mode": "INSTANT",
    "platform_fees": [{ "amount": { "currency_code": "USD", "value": "5.00" } }]
  }
}
```

`PARTNER_FEE` must be enabled through the supported seller/platform arrangement. The fee is not an arbitrary extra buyer debit and is not a general multiple-recipient split API. Store gross seller proceeds, platform fee and provider fee separately; reconcile actual amounts. [Immediate capture and partner fee requirements](https://developer.paypal.com/platforms/checkout/immediate-capture)

If one cart contains multiple sellers, phase one creates explicitly linked seller orders with separate payment outcomes; do not promise atomic settlement across them. A later supported multi-purchase-unit integration must test provider limits, currency rules, partial capture/refund outcomes and buyer approval semantics. Split-bill friends' obligations are a different domain from marketplace commission allocation.

Delayed disbursement is a separately enabled feature, not indefinite escrow. If eventually used, model its provider release deadlines, automatic release and refund/dispute behavior. The provider documentation states automatic disbursement after 28 days and requires the delay-disbursement feature. [Delayed disbursement](https://developer.paypal.com/platforms/checkout/delayed-disbursement/)

## 14. Reporting, exports, and AI assistance

`GET /v1/earnings?from=2026-10-01&to=2026-11-01&reporting_currency=USD` returns canonical paginated records and an authorized summary projection with UTC-defined boundaries or explicit requested timezone, per-currency totals and optional clearly estimated USD equivalents. Register these filter/summary fields in its 06/OpenAPI contract before implementation. The following JSON is the summary fragment, not the full pagination envelope. Use date ranges with inclusive `from`, exclusive `to`. Enforce row-level payer/payee permissions before grouping.

```json
{
  "period": { "from": "2026-10-01T00:00:00Z", "to": "2026-11-01T00:00:00Z" },
  "currencies": [{
    "currency": "USD",
    "accrued_minor": "100000",
    "payable_minor": "15000",
    "payout_pending_minor": "10000",
    "delivered_minor": "75000",
    "returned_minor": "0",
    "deductions_minor": "5000"
  }],
  "as_of": "2026-10-16T16:20:00Z",
  "provider_verified_at": "2026-10-16T16:19:40Z"
}
```

Definitions must specify whether accrued is gross or net and how adjustments affect totals. The example uses net accrued after separately reported deductions; gross reporting is available as an explicit field/query. Delivered is a historical delivered amount; net delivered subtracts later reversals/refunds and is separately named. Never present `accrued − delivered` as a provider wallet balance. A daily rollup is rebuildable from immutable accruals, allocations, journals and provider observations.

Business exports include source partner/task/invoice/order, payer/payee display name, original currency, gross, individual deduction categories, net due, actual delivered amount, provider references, status and evidence timestamps. Restrict PII; mitigate spreadsheet formula injection in CSV values beginning with `=`, `+`, `-`, or `@`. Export endpoints are asynchronous for large datasets with expiring owner-scoped download tokens and audit records.

AI features worth implementing after reliable facts exist:

- Explain why earnings are pending, with exact reason codes/evidence links and last verification time.
- Draft invoice lines from approved work, then require deterministic totals and merchant send authorization.
- Flag duplicate task claims, unusual destination changes or amount discrepancies; route to review rather than silently denying earned pay.
- Forecast cash needs per currency using historical obligations, explicitly labelled as estimates.
- Suggest a supported settlement currency/route using validated fee/FX evidence; never fabricate an executable rate.
- Answer multilingual earnings questions through scoped query tools with stable machine codes and localized explanations.

Tool outputs contain only data authorized for the user. Tool schemas do not expose unrestricted SQL, arbitrary partner URLs, payout receiver overrides or provider credentials. Imported task text and invoice descriptions are untrusted content, never instructions to the agent.

## 15. Security and operational controls

- [ ] Separate actual payer, beneficiary, commercial source and provider merchant/funding identities.
- [ ] Authenticate the connector and verify approval authority before accrual.
- [ ] Deduplicate both delivery event IDs and underlying payable-work business keys.
- [ ] Enforce destination ownership/consent and freeze versioned receiver data before submission.
- [ ] Require a payer mandate or explicit finance authorization for each automated payout.
- [ ] Reserve per-item/day mandate limits transactionally across concurrent workers.
- [ ] Stop new operations on mandate/merchant/provider consent revocation.
- [ ] Prohibit payout retries while an earlier attempt is pending/held/unclaimed/unknown.
- [ ] Verify item result; never infer delivered from batch acceptance/success alone.
- [ ] Keep all principal/fee/net/FX arithmetic exact and currency segregated.
- [ ] Protect connector/payout secrets with per-tenant access and rotation; never send them to models.
- [ ] Redact receiver PII from traces while retaining encrypted required evidence.
- [ ] Isolate finance/admin roles; an auditor has read access without payment authority.
- [ ] Record immutable adjustment and reversal entries, including who/why/evidence.
- [ ] Add kill switches by payer, connector, destination, country/currency and provider environment.
- [ ] Set alerts for task-event validation failures, payout unknown age, insufficient funding, held/unclaimed items and reconciliation discrepancies.
- [ ] Confirm product/country/currency permissions for every real go-live corridor, including separate India receiving requirements.

For Indian recipients, official help describes identity confirmation, PAN, Indian bank account and purpose-code requirements for international receipts. Record provider onboarding state and direct the recipient to PayPal for the required steps; WalPay does not collect or invent clearance for those provider checks. [India receipt setup](https://www.paypal.com/in/cshelp/article/how-do-i-receive-payments-through-paypal-HELP667)

## 16. Tests and acceptance criteria

These are specified future tests, not executed tests.

| Test group | Required scenarios | Acceptance evidence |
| --- | --- | --- |
| Connector authenticity | Valid/modified signature, old timestamp, rotating key, wrong installation | No accrual on unauthenticated delivery |
| Payable-work deduplication | Repeated event ID; different event IDs for same milestone; parallel workers | One accrual and one owed amount |
| Approval verification | Forged approver; webhook amount differs from GET; wrong worker; stale version; revoked task | Hold/reject with evidence; no payout |
| Contract calculation | Discounts/commissions/tax where applicable; rounding; net negative; mixed currencies | Exact deterministic amounts and no unsupported deduction |
| Destination risk | Changed destination after queueing; phone reassignment; absent consent; tenant mismatch | No silent receiver switch or directory data leak |
| Payer authority | Expired/revoked mandate, scope/currency/recipient violation, concurrent daily cap | Blocked before provider write |
| Funding | Insufficient balance/fee reserve; stale observation; unsupported sender country | Explicit block; no cross-tenant fallback |
| Provider idempotency | Timeout after accept; duplicate batch response; worker death; retry beyond dedupe retention | Stable IDs; GET reconciliation; no duplicate delivery |
| Item outcomes | Batch succeeds with failed item; unclaimed then returned; held then success; reversal after success | Correct obligation/journal; item-level evidence |
| Invoice semantics | Draft without send; send timeout; partial/pending payment event; external record payment | No false collected revenue |
| Subscription semantics | Approval pending; trial active unpaid; failure/suspension; cancellation with paid-through access | Billing and entitlement remain distinct |
| FX and fees | Expired estimate; actual rate differs; unsupported fee arrangement; refund conversion | Transparent estimated/actual fields; reapproval where required |
| Reports | Multi-currency totals; partial payout; returned item; concurrent projection rebuild | Totals trace to immutable source facts |
| Authorization | Worker reads another worker's earnings; seller invoice impersonation; funding account override | Denial without provider operation |
| Sandbox integration | Supported sender, two EMAIL/PAYPAL_ID recipients, one successful payout and item GET | Redacted provider evidence; raw PHONE not claimed tested |

Tools: Vitest, Testcontainers PostgreSQL/Redis, fast-check for allocation/journal invariants, Fastify injection for API tests, local signed demo-partner server, official provider Postman collections, and a controlled payout-adapter stub for rare failures. Keep real-sandbox tests opt-in with secrets, test-only receivers and a strict environment assertion. Never use real recipients or live money in CI.

Minimal demo acceptance path: signed demo-partner approval → verified task GET → one accrual → valid payer mandate → one sandbox payout item → real provider item success → earnings report attributes the delivered USD amount to partner/task. If Payouts is unavailable for the development account, complete the same path with a labelled provider fixture and demonstrate real PayPal Checkout separately; disclose the capability gap instead of faking a delivered sandbox payout.

## 17. Development phases and completion checklists

### E0 — signed demo partner and attribution

- [ ] Define partner task/approval contract and mapping consent.
- [ ] Build local demo-partner GET/event server with signing keys.
- [ ] Implement authentic intake, inbox/outbox and GET verification.
- [ ] Persist accruals/adjustments with one-work-one-payable constraint.
- [ ] Expose payee/payer-scoped per-partner earnings queries.
- [ ] Test duplicate, out-of-order, revoked and inconsistent approvals.

Exit: no external money moves; an approved work item creates exactly one correctly attributed obligation.

### E1 — business payout in supported sandbox

- [ ] Inspect actual sender Payouts access and supported country/currency.
- [ ] Configure truthful sandbox funding identity and test recipients.
- [ ] Implement destination verification/versioning and payer mandate caps.
- [ ] Implement batch/item operation persistence and stable identifiers.
- [ ] Verify real item result through GET and webhook pipeline.
- [ ] Fault-test ambiguous writes, item failure and concurrent cap enforcement.
- [ ] Update report only after item success evidence.

Exit: one task approval produces at most one delivered payout item, with recovery demonstrated.

### E2 — invoices, tips and membership operations

- [ ] Build first-party seller invoice draft/send flow and partial-payment reconciliation.
- [ ] Connect tips to owned merchant Checkout with creator attribution.
- [ ] Build USD subscription product/plan/approval/reconciliation with explicit entitlement policy.
- [ ] Add earnings exports, stale-evidence indicators and AI explanations.
- [ ] Test notification and billing authorization boundaries.

Exit: each business tool has its own provider evidence and cannot fake payment from a local flag.

### E3 — approved partner extensions and FX

- [ ] Obtain actual partner API contracts and payer funding authority.
- [ ] Complete PayPal platform approval/seller onboarding for delegated merchant operations.
- [ ] Enable and test partner-fee/FX corridors independently of own-store flows.
- [ ] Test revocation, account restrictions, fee settlement and refunds.
- [ ] Keep peer FX/P2P conversion disabled until a supported provider/business arrangement is proven.
- [ ] Run live-readiness review with documented corridor/product evidence and a reversible rollout.

Exit: each enabled real integration has scoped consent, funding, supported fees/currencies, recovery procedures and reproducible tests.
