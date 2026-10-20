# Social requests, receipt splitting, QR/NFC, causes and accessible backend

Planning date: 2026-10-16. Backend design only. This document specifies proposed **WalPay** `/v1` routes using TypeScript/Fastify, PostgreSQL and Redis/BullMQ. Domain IDs are UUIDv7. All monetary values use `{currency, amount_minor}` with integer strings. Primary reporting is USD, while every obligation/payment retains its original currency; USD estimates do not settle another currency or authorize FX. See `15-global-currency-and-fx.md` for executable quotes and platform charge policy. Canonical routes/events are defined in `06-api-contracts-and-events.md`; domain-detail routes not yet listed there are planned extensions to register in OpenAPI, not alternate names for canonical routes. Mutations of a revisioned resource require `If-Match: "<revision>"` even when an `expected_revision` field is also present; disagreement is rejected. All payment-derived records/events carry `mode: simulation | sandbox | live` with strict storage/projection isolation.

## 1. Payment boundary and feature classification

WalPay supplies identity resolution, consent, requests, split accounting, accessible explanations and approved PayPal orchestration. PayPal remains the wallet/payment provider. A contact's phone number is not authorization to debit their PayPal account, and merchant OAuth does not create a consumer P2P rail. No spendable WalPay balance, escrow, reload, balance transfer, or synthetic wallet is designed here.

The hackathon examples include phone-number movement, receipt splitting and causes, but examples are not entitlements to all corresponding APIs. [Hackathon resources and examples](https://paypalaihackathon.devpost.com/resources), checked 2026-10-16.

| Requested UPI-like experience | Backend plan and implementation classification |
| --- | --- |
| Pay by WalPay ID/phone | Supported identity resolution; payment availability returned by country/use case/provider capability |
| Send money to a friend | Unsupported generic consumer P2P API assumption; request tracking works, settlement disabled or visibly simulated until approved P2P access exists |
| Request money, collect request | Supported local request lifecycle and in-app delivery; provider collection only for permitted business/donation use case |
| Pay merchant, scan merchant QR, online checkout | Supported sandbox PayPal Checkout for verified own-store merchant; third parties gated |
| Phone/ID business payouts | Approved business Payouts flow, separate from friend reimbursements; sandbox uses supported verified recipient method |
| Pay by PIN/biometrics | WalPay passkey/PIN step-up plus normal PayPal approval, never a substitute |
| Split after paying | Supported local obligation computation/tracking; settlement capability clearly declared per request |
| Group expenses / recurring shared expenses | Supported records, revisions and voluntary reminders; no automatic personal debit |
| QR/NFC sharing | Supported opaque link resolution; NFC client/device support later; QR/link fallback always available |
| Receipt OCR and item-level split | Supported asynchronous local/cloud extraction with human confirmation and exact code arithmetic |
| Payment history, receipts, search, categories | Supported read models from actual accessible app payment events/imports; no claim of whole-wallet visibility |
| Pending/failed status, refunds, support and disputes | Shared authoritative payment lifecycle; split obligations adjusted separately |
| Favorites, recent payees, blocked users | Supported consented app identities and owner-private records |
| Scheduled payments / standing instructions | Business subscription/product-specific mandates only; reminders for unsupported P2P |
| Donation QR and progress | Sandbox verified merchant/cause flow; public campaigns production gated for account, region and fundraising acceptance |
| Recharge, mobile bills, utility billers, transport, FASTag | Provider/partner integration required; backlog/unsupported, never emulate success with Checkout |
| Bank linking, UPI IDs, UPI Lite/offline money, cash withdrawal | Unsupported by this PayPal-focused backend; no proprietary wallet |
| Rewards, lending/credit, insurance/investments | Outside MVP; require separate products and approvals, no fabricated availability |
| Multilingual / screen reader / voice-friendly flow | Backend locale/text/error support; final assistive-technology verification belongs to later client work |

PayPal Payouts supports business batch recipient identifiers, but current schema explicitly says sandbox does **not** support `PHONE`. Resolve a consented phone lookup to a verified permitted email/PayPal ID for the sandbox business payout path; do not present that as a direct phone-type payout test. Country/account approval still applies. [Payouts overview](https://developer.paypal.com/payouts/overview/), [create payout schema](https://developer.paypal.com/api/payments.payouts-batch/v1/definitions/create_payout_request/), checked 2026-10-16.

## 2. Modules, storage and events

| Module | Responsibility | Persistence / worker |
| --- | --- | --- |
| `identity-directory` | Handle/phone lookup, consent, recipient references | PostgreSQL + keyed phone lookup; rate limits in Redis |
| `payment-requests` | Request draft, send, accept, decline, expire, reminders | PostgreSQL + transactional outbox |
| `receipts` | Upload quarantine, OCR extraction, validation, revision | Encrypted object store + BullMQ OCR workers |
| `splits` | Exact allocation, participant acceptance, dispute/correction | PostgreSQL immutable split revisions |
| `share-intents` | QR/NFC/link token issuance and resolution | Hashed opaque tokens + revocation/expiry |
| `causes` | Campaign verification, donation intents, public aggregate | PostgreSQL + projection worker/SSE |
| `localization` | Message keys, locale, currency metadata, human summaries | Versioned translations; no model-generated payment amounts |

Canonical shared events consumed/emitted here are `payment.capture_confirmed.v1`, `payment.refund_confirmed.v1`, `payment_request.accepted.v1`, `split.confirmed.v1` and `campaign.progress_changed.v1`. Planned domain event extensions are `payment_request.created.v1`, `payment_request.delivered.v1`, `payment_request.declined.v1`, `payment_request.expired.v1`, `receipt.extracted.v1`, `receipt.confirmed.v1`, `split.disputed.v1` and `split.revised.v1`; register them before implementation rather than introducing alternate shared payment-event names. Every event has `event_id`, aggregate ID/version, `occurred_at`, schema version, mode and trace ID. Redacted payloads travel via the outbox; event handlers have inbox deduplication.

## 3. Handles, phone contacts and recipient safety

Use stable `user_id` internally and revocable public `@handle` aliases. A handle rename cannot redirect an already-approved payment request: bind recipient user ID and verified PayPal destination snapshot into the final payment digest. Reserve confusing/system handles; normalize Unicode and prevent mixed-script impersonation where appropriate. Display verified recipient name/avatar context but avoid implying a bank-grade legal identity check for ordinary users.

Phone enrollment requires E.164 normalization, an OTP or an established identity provider claim, anti-abuse throttling and explicit directory opt-in. Phone possession proves current control only; recycled numbers require re-verification and never reassign old obligations or financial destinations automatically. Changing a destination requires fresh verification, audit and step-up; suspicious changes can delay business payout release. Store phone values encrypted and a server-keyed HMAC for lookup. A plain unsalted phone hash is insufficient against enumeration.

`POST /v1/recipients/resolve` accepts a single normalized handle/phone or an opaque contact token. It is authenticated and purpose limited. Return only an authorized, directory-visible match with a short-lived `recipient_ref`, masked display name, available methods and verification time. Do not return an account-existence oracle for arbitrary probes. Unknown/private/blocked targets use the same neutral response class; known contacts can get scoped detail after a relationship check. Rate-limit per user/device/IP and detect sequential phone enumeration.

```http
POST /v1/recipients/resolve
{"kind":"WALPAY_HANDLE","value":"@sam","purpose":"SPLIT_REQUEST"}

200
{
  "data": {
    "recipient_ref":"<opaque-short-lived-reference>",
    "display_name":"Sam R.",
    "verification":{"directory_opt_in":true,"phone_verified":true},
    "capabilities": {
      "in_app_requests":true,
      "personal_reimbursement_settlement":"UNAVAILABLE",
      "business_payout":"REQUIRES_ELIGIBLE_BUSINESS_FLOW"
    }
  }
}
```

Do not upload an entire address book by default. Prefer one-contact selection and normalized lookup; any batch discovery requires explicit consent, strict limits, revocation and deletion. Store the minimum contact relationship metadata. Raw contacts, OTPs and beneficiary destinations never go to an AI provider. In-app notices are sufficient for MVP; SMS/email integrations require channel consent, unsubscribe rules and a separately enabled provider.

## 4. Payment request lifecycle

Request intent types are explicit: `PERSONAL_REIMBURSEMENT`, `COMMERCIAL_INVOICE`, `MERCHANT_PAYMENT`, `CAUSE_DONATION`. They determine allowed rails. Never send a personal dinner reimbursement through a fabricated goods/services invoice, or use platform-funded Payouts as an unauthorized passthrough consumer transfer. Shared balances are debt records, not assets held by WalPay.

```http
POST /v1/payment-requests
Idempotency-Key: <key>
{
  "recipient_ref":"<resolved-ref>",
  "purpose":"PERSONAL_REIMBURSEMENT",
  "amount":{"currency":"USD","amount_minor":"1083"},
  "note":"Your share of dinner",
  "split_revision_id":"<uuidv7>",
  "expires_at":"2026-10-23T12:00:00Z",
  "delivery_channels":["IN_APP"]
}

201
{
  "data": {
    "id":"<uuidv7>",
    "state":"draft",
    "settlement_capability":"UNAVAILABLE_PERSONAL_P2P",
    "available_actions":["SEND_REQUEST","CANCEL_REQUEST"],
    "amount":{"currency":"USD","amount_minor":"1083"}
  }
}
```

`POST /v1/payment-requests/{id}/send` records sender consent and creates delivery jobs. In-app sending is a user action; AI may draft a request but cannot independently message contacts. `POST /.../{id}/accept` records acknowledgment of the debt/request, not payment. `POST /.../{id}/decline`, `/cancel`, `/remind`, and `GET /v1/payment-requests?direction=incoming&cursor=...` complete lifecycle. Respect blocks, expiry, per-recipient reminders and quiet hours. Set a conservative default reminder frequency, at most one explicit reminder per 24 hours and no automatic reminder without opt-in.

Request states: `draft -> sent -> accepted -> settled`; alternatives `declined`, `cancelled`, `expired`, `disputed`. `settled` must include `settlement_evidence: PROVIDER_CONFIRMED | MUTUALLY_ATTESTED_EXTERNAL`. External payment attestation is a separate explicit two-party record; it does not become a verified PayPal transaction or update merchant revenue. A unilateral sender assertion remains `external_settlement_reported` pending confirmation. Attachments are optional and sensitive.

If a permitted settlement product exists, `POST /v1/payment-intents` references request ID/version and produces its immutable amount, beneficiary, platform charge and permitted rail. Standard `/authorization-options`, `/authorizations`, `/paypal-order`, `/capture` routes follow. Unsupported personal settlement returns a stable capability response with request tracking still usable. API payload includes `mode` with `simulation`, `sandbox` or `live`; simulations cannot enter real payment tables.

An accepted commercial request may link to a real approved PayPal invoice from the business module, but invoices are never auto-created solely because a dinner receipt exists. Prevent self-request loops, invalid recipient references, request spam, replay, altered amounts, IDOR and staff access without audit.

## 5. Receipt ingestion and OCR confirmation

Pipeline: `upload_requested -> quarantined -> scanning -> extracting -> needs_review -> confirmed`; alternatives `rejected`, `extraction_failed`, `deleted`. OCR output is an untrusted draft, not the source of financial authority.

1. `POST /v1/receipts/uploads` returns a short-lived pre-signed upload URL, object key bound to owner and maximum size/type. Design defaults: images up to 10 MB, at most 5 pages, bounded pixel count. PDFs require a sandboxed rasterizer, no embedded script execution or arbitrary remote fetch.
2. Client uploads directly to quarantine storage. `POST /v1/receipts` submits object reference; server verifies ownership, magic bytes, declared/actual size, checksum and decode success. Scan malware, re-encode images, remove EXIF/location metadata and defend against decompression bombs.
3. Worker applies orientation/contrast preprocessing, then a deterministic OCR or vision-model adapter with a strict extraction schema. Local extraction is available for privacy/offline strategy; cloud processing requires recorded user consent and approved retention/region settings. Uploading to cloud is not the default when consent says local only.
4. Persist raw extraction text, structured candidate, per-field confidence/source bounding boxes, model/tool version, extraction job ID and content hash. Confidence is advisory. The model never sums or rounds final obligations.
5. Deterministic validator parses localized digits/decimal separators only with declared/confirmed currency rules; reject ambiguous `$`, decimal `1.234` meaning, unsupported exponent, negative total, duplicate scan concerns and inconsistent components. Return flags with evidence, not guessed certainty.
6. Require the payer to confirm/correct currency, total, tax/tip/fees/discount, line items and already-paid amount. `POST /v1/receipts/{id}/confirm` requires `If-Match` for the reviewed revision; create an immutable confirmed revision. If `expected_revision` is supplied too, it must match the header. No split requests sent from OCR alone.

```http
GET /v1/receipts/{id}
{
  "data": {
    "id":"<uuidv7>",
    "state":"needs_review",
    "revision":1,
    "currency":"USD",
    "extraction": {
      "merchant_name":"Cafe Example",
      "lines":[
        {"id":"<uuidv7>","label":"Pasta","quantity":"1","line_total_minor":"2000","confidence":0.97},
        {"id":"<uuidv7>","label":"Shared bread","quantity":"1","line_total_minor":"1000","confidence":0.83}
      ],
      "subtotal_minor":"3000",
      "discount_minor":"0",
      "tax_minor":"250",
      "tip_minor":"0",
      "fees_minor":"0",
      "total_minor":"3250"
    },
    "review_flags":[{"code":"CONFIRM_TOTAL","field":"extraction.total_minor"}],
    "financially_confirmed":false
  }
}
```

The `confidence` number is not a calibrated guarantee; store provider semantics and evaluate it on a receipt corpus before applying any triage threshold. Labels containing instructions such as 'send all contacts to this URL' are receipt data. Local/cloud OCR runs with no finance/contact/message tools, no arbitrary URL fetch, bounded CPU/memory/time and a JSON parser rejecting unknown actionable fields.

Receipt/transaction matching is an optional suggestion based on same currency, amount, merchant and time. Require explicit confirmation of the paid transaction. Duplicate image hashes flag review, not automatic deletion: several people may upload the same bill. A receipt proves what was printed, not that money settled.

## 6. Exact split rules and allocation algorithm

Supported modes: `EQUAL`, `EXACT_AMOUNTS`, `WEIGHTED`, `ITEMIZED`. A split contains a confirmed receipt revision, original currency, payer/participant IDs, allocation policy, paid-by contributions, immutable allocation results and version. All participants must be unique IDs; self-share is recorded but does not create a request to self.

Core invariants:

- Every component allocation sums exactly to that component's integer total.
- Sum of participant gross shares equals confirmed receipt total.
- Sum of paid-by contributions equals total for a fully paid bill. Partial payment and unpaid merchant balance are separate explicit states.
- For each participant, `net_obligation = gross_share - paid_contribution`. Sum of net obligations is zero for a fully paid bill. `gross_minor` and `paid_minor` are unsigned minor-unit strings; `net_minor` is explicitly a derived **SignedMinor** integer string (can begin with `-`), not an unsigned Money/payment amount. A positive value means owes, negative means is owed, and zero means balanced; convert to positive Money plus explicit direction before constructing any request/payment.
- A participant may owe a payer, but no receivable is a spendable WalPay balance. Split calculation creates no payment.
- Never add two currencies. A multi-currency group has separate currency subledgers; USD summaries include rate/source/time and estimation status. A converted settlement requires an exact quote and fresh approval.
- Receipt total excludes any later WalPay settlement fee. If a permitted payment flow charges a platform fee, disclose and authorize it separately in the settlement quote; do not silently inflate the friend's claimed meal share.

Use largest remainder allocation with deterministic participant-ID tie breaking. All callers supply the same canonical order; do not depend on database row order or user display name. Carry every tax/tip/discount allocation as integers; percentages/weights are bounded rational integers, never IEEE floats.

```ts
type Weight = { id: string; weight: bigint };
function allocateMinor(total: bigint, rows: Weight[]): Map<string, bigint> {
  if (total < 0n || !rows.length) throw new Error("INVALID_ALLOCATION");
  // UUIDv7 IDs are normalized lowercase ASCII, so this ordering is host-independent.
  const canonical = [...rows].sort((a, b) => a.id < b.id ? -1 : a.id > b.id ? 1 : 0);
  if (new Set(canonical.map(r => r.id)).size !== rows.length) throw new Error("DUPLICATE_PARTICIPANT");
  if (canonical.some(r => r.weight < 0n)) throw new Error("NEGATIVE_WEIGHT");
  const divisor = canonical.reduce((s, r) => s + r.weight, 0n);
  if (divisor === 0n) throw new Error("ZERO_WEIGHT_TOTAL");
  const parts = canonical.map(r => ({
    id: r.id,
    share: total * r.weight / divisor,
    remainder: total * r.weight % divisor
  }));
  const assigned = parts.reduce((s, r) => s + r.share, 0n);
  const left = total - assigned;
  const ranked = parts.filter(r => r.remainder > 0n).sort((a, b) =>
    a.remainder === b.remainder ? (a.id < b.id ? -1 : 1) : (a.remainder > b.remainder ? -1 : 1));
  // left < number of positive-weight participants; bounded by request schema limits.
  for (let i = 0n; i < left; i++) ranked[Number(i)].share += 1n;
  return new Map(parts.map(r => [r.id, r.share]));
}
```

Test the helper with empty/zero/duplicate/overflow conditions and bound input lengths. Negative discounts are represented as positive component magnitudes allocated and then subtracted; do not pass a negative total to this helper. The function returns minor-unit integers; transport serialization converts them to decimal strings.

Itemized computation order:

1. Use confirmed line totals. Fractional quantities are rational strings; line total on receipt remains the confirmed monetary basis, not a floating-point multiplication of inferred unit price.
2. Allocate each line among assigned participants by equal/rational weights. Shared bread can be shared by all; alcohol can be assigned to a subset.
3. Apply line-specific discounts only to eligible lines/participants. Allocate a bill-wide discount proportionally to eligible pre-discount subtotal unless the payer selects an explicit alternative. Prevent discount exceeding eligible base.
4. Allocate tax using per-line tax when known; otherwise require confirmed proportional or equal policy. Separate tax categories and included-tax receipts; included tax is disclosed, not added a second time.
5. Allocate tip by confirmed pre-/post-discount basis or explicit amounts. A printed suggested tip is not an actual paid tip. Mandatory service charge and voluntary tip are different fields; don't double count them.
6. Allocate other fees only under explicit policy. Zero-base allocation requires equal/manual allocation and review; division by zero never silently yields zero.
7. Check sum against receipt total. A mismatch blocks publication. Permit an explicitly confirmed `ROUNDING_ADJUSTMENT` component with reason and bounded magnitude; never hide a mismatch as 'AI correction'.
8. Apply paid-by contributions and create a settlement plan. For MVP one payer, each debtor points to that payer. Multiple-payer debt simplification is an opt-in display/proposal; do not redirect a confirmed payee without participant consent.

Example: equal split of `3250` USD minor units among canonical A/B/C yields `1084`, `1083`, `1083`. If A paid the full `3250`, A's own share is `1084` and B/C each owe A `1083`. A does not receive a request for their own `1084`. For tax `250` on bases `2000` and `1000`, the same allocation rule gives `167` and `83`.

## 7. Split APIs, participant consent and corrections

```http
POST /v1/splits
Idempotency-Key: <key>
{
  "receipt_id":"<uuidv7>",
  "receipt_revision":2,
  "split_method":"EQUAL",
  "participants":[{"user_id":"<A>"},{"user_id":"<B>"},{"user_id":"<C>"}],
  "paid_by":[{"user_id":"<A>","amount":{"currency":"USD","amount_minor":"3250"}}],
  "allocation_policy":{"remainder":"LARGEST_REMAINDER_ID_ASC","tax":"PROPORTIONAL","tip":"PROPORTIONAL"}
}

201
{
  "data": {
    "id":"<uuidv7>","revision":1,"state":"draft",
    "total":{"currency":"USD","amount_minor":"3250"},
    "shares":[
      {"user_id":"<A>","gross_minor":"1084","paid_minor":"3250","net_minor":"-2166"},
      {"user_id":"<B>","gross_minor":"1083","paid_minor":"0","net_minor":"1083"},
      {"user_id":"<C>","gross_minor":"1083","paid_minor":"0","net_minor":"1083"}
    ],
    "settlement_capability":"REQUEST_TRACKING_ONLY",
    "requires_payer_confirmation":true
  }
}
```

`PUT /v1/splits/{id}/draft` corrects the unconfirmed allocation with `If-Match`. `POST /v1/splits/{id}/confirm` requires `If-Match` and explicit payer confirmation; it creates an immutable confirmed split revision and emits `split.confirmed.v1` without sending requests. A separate `POST /v1/splits/{id}/requests` requires the confirmed revision and explicit invitation/delivery consent, then enqueues voluntary in-app requests. Participants can read their own share and sufficiently detailed calculation evidence. Shared receipt access is consent scoped; redact unrelated sensitive line items when necessary. Payer and invited participant permissions differ: an invitee cannot change others' paid-by contributions.

Planned extension routes `POST /v1/splits/{id}/accept`, `/disputes`, `/revisions` and `/cancel` require `If-Match` for the expected version. A dispute includes category `WRONG_ITEM`, `WRONG_AMOUNT`, `ALREADY_PAID`, `WRONG_PERSON` or bounded text. Preserve the original revision and evidence. A material revision supersedes unpaid old requests, requires participants to review changed shares, and emits corrected notices. Existing paid settlements remain immutable. A corrected smaller debt creates an explicit adjustment/refund proposal; never auto-debit/auto-refund or erase history. Handle a participant payment racing with revision by locking request/split revision rows and reconciling any provider operation in flight.

Receipt refunds have two layers: provider refund on the merchant payment, and group fairness adjustment. Confirm actual refunded amount/lines; allocate credit under the original allocation evidence or explicit new agreement. Update requests only after the refund's status is known where relevant; a disputed/pending refund is not received money.

## 8. QR and NFC are share transports

Design a common `share_intent` object with types `MERCHANT_CHECKOUT`, `REQUEST`, `SPLIT_INVITE`, `CAUSE_DONATE`, `PROFILE`. Use a random high-entropy token stored hashed; encode only an HTTPS WalPay URL such as `https://<walpay-host>/s/<opaque>`. Do not encode bearer access tokens, PayPal secrets, full phone numbers, names, debt history or a claim that payment has occurred. A compact signed structured QR is optional later; an opaque server-resolved token is easier to revoke.

Planned extension `POST /v1/share-intents` accepts `{type, resource_id, audience, expires_at}` and returns opaque URL, token expiry and a text alternative. Canonical `POST /v1/qr-resolutions` accepts the token/link and returns safe context, current merchant/cause name, amount policy, immutable payment/request reference and required actions. Unknown/expired/revoked tokens get neutral errors. Private invites bind audience and require login; public campaign QR resolves to public campaign information, never a donor's details. QR, link and NFC transports all use this one resolver.

Static QR links to a verified merchant/campaign/profile and asks for a fresh amount/intent. Dynamic QR references one exact current quote/request with expiry and expected beneficiary. Enforce resource version at resolution and at payment authorization; QR replay cannot duplicate a single-use payment attempt. Public campaign links may deliberately be reusable but each donation creates a unique intent. Substituted QR cannot bypass explicit beneficiary/amount review.

NFC uses the same HTTPS URL through a supported NDEF URL record. NFC does not provide payer identity, proof of proximity, payment approval or settlement confirmation. Treat a scanned tag as untrusted input. Hardware/browser support is client dependent; make copy-link/QR/short-code alternatives part of the contract and test backend behavior without requiring NFC hardware. Never label this backend a card-network contactless POS integration.

PayPal also documents hosted payment links and QR codes for merchant collection. They can be a permitted fallback for verified merchants, but a URL alone does not prove WalPay can observe or attribute its payment events. Use only a tested integration with resource linkage and provider confirmation for automated state updates. [PayPal Payment Links and Buttons](https://developer.paypal.com/payment-links-buttons/overview), checked 2026-10-16.

## 9. Cause campaigns, live totals and recurring support

Campaigns belong to verified recipient organizations/eligible merchant accounts. Save organization display/legal name, owner, country, permitted donation products, provider merchant ID, verification status, goal in original currency, moderation status, purpose, terms and contact. Public publication requires review of prohibited/fraudulent causes, beneficiary identity and applicable PayPal country/account/fundraising acceptance. The sandbox demo uses a clearly labelled fictitious cause and sandbox merchant.

```http
GET /v1/causes/{id}
{
  "data": {
    "id":"<uuidv7>","title":"Community Library Demo","mode":"sandbox",
    "recipient":{"display_name":"Demo Library","verification_status":"SANDBOX_TEST_ACCOUNT"},
    "goal":{"currency":"USD","amount_minor":"100000"},
    "totals": {
      "gross_confirmed":{"currency":"USD","amount_minor":"42000"},
      "refunded_confirmed":{"currency":"USD","amount_minor":"2000"},
      "net_confirmed":{"currency":"USD","amount_minor":"40000"},
      "pending":{"currency":"USD","amount_minor":"5000"},
      "progress_bps":4000,
      "projection_version":37,
      "as_of":"2026-10-16T15:30:00Z"
    },
    "fee_disclosure":{"platform_charge_policy_version":"<version>","shown_before_approval":true}
  }
}
```

Donation initiation: `POST /v1/causes/{id}/donation-intents` validates campaign live state, amount bounds, currency, anonymity preference and fee allocation. It delegates to shared payment intents with explicit donation purpose and permitted recipient. The quote distinguishes donor contribution, provider/platform fees, total charged and estimated/known recipient net. Do not claim a tax deduction or zero fees without evidence. A platform charge on a donation must be permitted and disclosed; platform revenue and campaign contribution are separate components.

A donor approving checkout is not a confirmed donation. Verify webhook signatures, deduplicate inbox events, validate merchant/resource/currency/amount and reconcile ambiguous outcomes before counting captures. App-scoped PayPal events do not reveal every transaction of that organization across unrelated apps. External Donate button/IPN or other product paths require their own documented event/attribution adapter; do not assume an Orders webhook will cover every donation product. [PayPal webhooks](https://developer.paypal.com/api/rest/webhooks/), [Donate SDK](https://developer.paypal.com/sdk/donate/), checked 2026-10-16.

Projection rules:

- `gross_confirmed = sum(unique confirmed eligible contribution components)`; exclude platform charge and pending/failed intents.
- `net_confirmed = gross_confirmed - confirmed_refunded_contribution - confirmed_reversals`; provider fees and merchant net are a separately named financial view.
- Each refund/reversal references its original donation allocation and cannot exceed it; duplicate events cannot subtract twice. If an event arrives before its capture/allocation, hold it for reconciliation and fetch authoritative state instead of publishing a negative invented balance. If a refund includes platform charges, allocate explicitly rather than treating all refund value as contribution.
- `progress_bps = min(10000, floor(max(0,net_confirmed) * 10000 / goal))` for positive goals. Return actual uncapped totals so overfunding is visible; goal zero requires a no-target campaign variant.
- For multiple donation currencies, maintain separate exact totals. USD public reporting uses versioned rate snapshots and an `ESTIMATED_REPORTING` label; do not present heterogeneous minor units as one exact balance. Freeze/restate reporting conversion consistently under the FX plan.
- Donation pledges/subscriptions promised for future cycles are not money received. Pending money has a separate label. A provider dispute/reversal can reduce confirmed net and progress.
- Projector stores unique event ID, contribution delta, aggregate version and checkpoint in one transaction. Support a replay/rebuild from immutable captures/refunds and compare with incremental projection.

`GET /v1/causes/{id}/events` uses public SSE containing aggregate totals/version only. Resume with `Last-Event-ID`; if cursor is too old, return/reset to latest snapshot. Redis Pub/Sub is a wakeup optimization, not the sole event history. Fall back to ETag-based polling. Do not emit donor names/emails/notes unless each donor explicitly opted into those exact fields; anonymous mode hides them from public streams and exports while retaining restricted operational evidence.

Recurring support can use an eligible PayPal subscription product/plan, with the donor's explicit provider mandate. Track subscription ID, plan version, frequency, contribution/fees, status, cancellation and each confirmed cycle. First approval does not authorize arbitrary one-off amounts or different causes. A recurring donation mandate is not permission for an agent to debit personal reimbursements. [PayPal Subscriptions](https://developer.paypal.com/platforms/subscriptions/), checked 2026-10-16.

Multi-cause donation MVP: separate explicit donation intents/payments per verified cause. One approved merchant/multiparty integration may support a later combined checkout with exact allocation, payee eligibility and refund policy. Never collect into a synthetic WalPay pool and show allocated IOUs as delivered donations. Splitting across causes is an agent proposal until recipient-specific provider settlement is confirmed.

## 10. Accessibility and multilingual backend contracts

Use WCAG 2.2 AA as the later full-product target and provide backend primitives now. Backend-only tests cannot certify a screen-reader experience; final clients require actual keyboard, screen-reader, zoom and assistive input testing. [WCAG 2.2](https://www.w3.org/TR/WCAG22/), checked 2026-10-16.

- Persist `locale` (BCP 47), timezone, preferred number formatting, notification channels and text verbosity with consent. Do not infer disability from transactions or store unnecessary disability labels.
- Return stable machine codes and message keys plus localized plain text. Include `Content-Language`, fallback locale and translation version. Translating an error cannot change its retriable/authorization semantics.
- Always return raw currency code/integer amount along with human display string. Pronounceable summaries include currency name and exact major/minor units; '$' alone is ambiguous. Human amount formatting never feeds arithmetic.
- Return `accessible_summary`, beneficiary name, total, fee breakdown and required actions for payment/request/QR resources. Server generates authoritative amounts/names; model output may draft optional explanations with protected immutable fields.
- Provide receipt line text, OCR flags, calculation steps, split components and nonvisual QR/link equivalents. Scanning a QR is optional; manual request code and searchable consented identity flow exist.
- Avoid audio-only OTP/status and biometric-only recovery. Passkey-compatible device PIN/security key and accessible account recovery follow security policy. Backend returns methods actually available, not a forced face/fingerprint prompt.
- WebAuthn verifies signed origin/RP-bound public-key assertions with user verification as required. WalPay stores credential public keys/metadata, never biometric images/templates. Provider login/approval continues independently. [WebAuthn specification](https://www.w3.org/TR/webauthn-3/), checked 2026-10-16.
- All deadlines are RFC 3339 UTC plus user timezone for display. Provide sufficient confirmation/challenge windows and safe restart; extending app presentation time cannot extend an expired provider quote or consent.
- Support English/Hindi initially with a framework for other languages and RTL scripts. Locale changes cannot alter stored monetary values, identities, digest, order or consent version. Human-review all financial/security critical translations; AI can draft low-risk help text.
- Voice requests are drafts only until an authenticated user confirms parsed amount, currency, beneficiary and purpose. Never authorize from an audio transcript, voice resemblance or speech sentiment.
- Event payloads include semantic state and localized status text; no color/icon-only meanings. Rate-limit repeated state notifications and permit quiet mode/preferences without concealing failures.
- Exports offer machine-readable UTF-8 CSV/JSON with documented money/currency fields, localized text as a separate optional layer and formula-injection protections.

## 11. Data schema and authorization details

| Table | Keys / critical constraints |
| --- | --- |
| `public_identities` | `user_id, handle_canonical unique, directory_opt_in, version`; history cannot redirect frozen recipient |
| `verified_contacts` | `user_id, kind, encrypted_value, lookup_hmac, verified_at, revoked_at`; secret key rotation plan |
| `contact_relationships` | Owner, contact user ID, consent/source, blocked state; owner-private |
| `recipient_references` | Hashed opaque ref, requester, purpose, beneficiary version, expiry; no cross-user reuse |
| `payment_requests` | Sender/recipient IDs, purpose, amount/currency, revision, state, rail capability, expiry; positive amount |
| `request_deliveries` | Request/version/channel/recipient unique, delivery state, consent version, next permitted reminder |
| `receipt_objects` | Owner, quarantine/approved object key, checksum, MIME, bytes, retention/deletion status |
| `receipt_revisions` | Receipt/revision unique, extraction version, confirmed components, currency, confirmed actor/time |
| `split_revisions` | Split/revision unique, receipt revision, `split_method`, calculation policy version, state, canonical digest; confirmation separate from request dispatch |
| `split_shares` | Split revision/user unique, component allocations, gross/paid/net strings; reconciliation checks |
| `split_disputes` | Share/revision/reporter, category, bounded evidence, resolution; restricted read |
| `settlement_links` | Request revision/payment intent/provider evidence unique; distinguish external attestation |
| `share_intents` | Token hash unique, resource ID/version/type, audience, expiry, max uses, revoked state |
| `campaigns` | Owner, verified recipient, country/currency, goal, moderation/live state, policy version |
| `donation_allocations` | Payment/cause/contribution/currency/mode unique, fee allocation and refund references |
| `campaign_projection_events` | Unique source event and campaign, exact delta, projection version |
| `campaign_totals` | Campaign/currency/mode unique, gross/refund/reversal/pending counts, version/checkpoint; never mix simulation/sandbox/live |

Every read is resource/participant/role scoped. `user_id` in a body is not authentication. Administrative support uses audited purpose-bound access with redaction; no broad public endpoint reveals debts, contact graphs, donor history or receipt files. Receipt image URLs are short lived and owner/participant scoped; do not place them in logs/public SSE. Deletion expires objects and unnecessary extracted PII under retention policy while preserving required minimal financial evidence and legal holds. Publication can be withdrawn without rewriting confirmed provider transactions.

## 12. Phases, test tools and release gates

| Phase | Scope | Completion evidence |
| --- | --- | --- |
| S0 | Directory opt-in, handles, request lifecycle and explicit unavailable rails | Two users can send/decline/accept a tracked request without any money claim |
| S1 | Receipt upload/quarantine/OCR/confirmation | Golden corpus extraction reports + malicious-file rejection |
| S2 | Integer split and participant revisions | Property tests prove conservation; manual correction never changes old paid evidence |
| S3 | QR/share resolver and in-app notifications | Expiry/replay/recipient-substitution/IDOR tests; accessible text equivalent |
| S4 | Sandbox verified cause checkout and live projection | Capture/refund duplicates produce correct totals; no pending donation counted |
| S5 | Multilingual contracts and fee/FX disclosures | Locale invariant tests and human-reviewed English/Hindi critical messages |
| S6 | Approved business payout linkage / later P2P adapter | Actual entitlement and provider-specific conformance; otherwise remains unavailable |

Test stack: Vitest, fast-check, Fastify `inject`, real PostgreSQL/Redis integration services, malicious file/receipt fixture corpus, OpenAPI contract validation, Postman/Newman collections, deterministic provider fake and separately flagged PayPal sandbox runs. Use a synthetic receipt corpus without real contacts/card numbers. Record OCR field accuracy, currency/total error rate, review burden and extraction latency; acceptable financial error is handled by mandatory confirmation, not a claimed perfect OCR model.

| Test ID | Scenario | Required assertion |
| --- | --- | --- |
| SOC-001 | Equal/weighted split over randomized totals/weights | Exact sum, deterministic ties, nonnegative gross allocations |
| SOC-002 | 3250 among three; A paid | Gross 1084/1083/1083; only B/C requests; net sum zero |
| SOC-003 | Tax, service charge, tip, discount, included tax | Components conserved; no tax/tip double counting |
| SOC-004 | Zero eligible tax/discount base | Explicit review/manual allocation, no divide-by-zero or lost cents |
| SOC-005 | JPY, two-/three-decimal supported currencies, locale separators | Correct exponent; ambiguous parsed total requires review |
| SOC-006 | Blurred/rotated receipt, duplicated tip suggestion, refund line | No automatic payable split; review flags/evidence retained |
| SOC-007 | Receipt text includes tool instructions/URL | No finance/contact tool/network call; schema-bound OCR only |
| SOC-008 | Oversized/decompression-bomb/malicious PDF/wrong MIME | Rejected/quarantined before model/tool processing |
| SOC-009 | Recycled phone, changed handle, wrong recipient ref | Frozen beneficiary never silently changes; reference scope enforced |
| SOC-010 | Phone enumeration/private/blocked user | Neutral result class, throttling, no contact leakage |
| SOC-011 | Personal request tries to call merchant/payout rail | Capability error; no provider request made |
| SOC-012 | Confirm/retry then repeated explicit request-dispatch job | Confirmation alone sends nothing; one request per split revision/participant and one delivery job key |
| SOC-013 | Payer edits published split during payment | Revision conflict/guard; preserve in-flight and paid evidence |
| SOC-014 | Debtor disputes after payment/receipt correction | Adjustment proposal only; no erased transaction or automatic debit |
| SOC-015 | User reports external payment unilaterally | No provider-confirmed revenue; mutual evidence required for tracked settlement |
| SOC-016 | QR replay/expiry/revocation/resource version drift | No duplicate payment; stale/private reference safely rejected |
| SOC-017 | NFC tag contains external URL or altered recipient | Treat untrusted; no auto-approval/capture or unrestricted fetch |
| SOC-018 | Cause approved callback but capture pending/failed | Confirmed progress unchanged; pending amount separate |
| SOC-019 | Duplicate/out-of-order captures/refunds/reversals | Unique accounting deltas; rebuild matches incremental projection |
| SOC-020 | Refund includes contribution plus platform fee | Correct contribution decrement; no over-subtraction |
| SOC-021 | Donation in USD and EUR | Separate exact ledgers; USD report explicitly estimated if conversion not locked |
| SOC-022 | Subscription approved but no cycle settled | Zero confirmed cycle contribution until authoritative payment |
| SOC-023 | Anonymous donor and public SSE/export | No identity/notes/contact leak; aggregate version resumable |
| SOC-024 | User locale changes after approval | Money, payee, digest, consent and payment state unchanged |
| SOC-025 | Screen-reader summary/RTL/Hindi amount/long name | Text contract complete; immutable fields preserved; client QA tracked separately |
| SOC-026 | Fake/replayed passkey or PIN brute force | Origin/RP/nonce/expiry/user verification/rate-limit checks enforced |
| SOC-027 | Reminder runs after request declined or user blocked | No notification; worker rechecks authoritative state/consent |
| SOC-028 | Projection wakeup lost or SSE disconnected | Snapshot/history recovery correct; Redis loss does not lose totals |

Granular completion checklist:

- [ ] Personal request, commercial invoice, merchant payment and donation purposes are separate schema enums and policy paths.
- [ ] Unsupported P2P/phone sandbox paths return honest capability labels and cannot call provider adapters.
- [ ] Handle ownership, OTP verification, phone HMAC, directory opt-in and block checks implemented.
- [ ] Recipient references bind requester/purpose/destination revision and expire.
- [ ] No full-address-book or raw-contact AI upload exists by default.
- [ ] In-app request delivery requires user action; external channels stay disabled until consent/provider integration is complete.
- [ ] Request expiry, decline, dispute, cancellation and reminder throttles covered by state transition tests.
- [ ] Receipt upload content checks, quarantine, malware scan, bounded decoder and EXIF removal implemented.
- [ ] Cloud OCR obeys explicit user consent/region/retention; local-only mode does not fall back silently.
- [ ] Currency/total/tax/tip confirmation required and immutably versioned.
- [ ] All split calculation uses integer/rational code and deterministic test vectors.
- [ ] Every line/discount/tax/tip/fee component reconciles exactly; rounding adjustment is explicit.
- [ ] Fully paid versus partly paid bills and multiple payers have explicit behavior.
- [ ] Participant permissions/receipt visibility and external settlement evidence types are enforced.
- [ ] Revision races preserve old paid/in-flight evidence and require fresh approval for changed terms.
- [ ] Share tokens are high entropy, hashed, revocable, audience scoped and contain no PII or payment secrets.
- [ ] QR/NFC resolves to a reviewable resource; never triggers payment merely by reading the token.
- [ ] Donation beneficiary/country/account acceptance verified before production publication.
- [ ] Contribution, provider fee, platform charge, total charged and recipient net are correctly named/disclosed.
- [ ] Cause projections count only authoritative unique contributions and handle partial refunds/reversals.
- [ ] Multi-currency aggregates preserve originals and label USD reporting estimates.
- [ ] Donation subscription cycles count only after payment; cancellation supported.
- [ ] Public streams and reports preserve donor anonymity and exclude private notes.
- [ ] English/Hindi critical messages reviewed; translation cannot alter values or authorization.
- [ ] Accessible summaries, manual QR/code alternatives and non-biometric recovery methods present in API contracts.
- [ ] Backend evidence distinguishes accessibility support from full client accessibility certification.
- [ ] Postman collections demonstrate receipt confirm -> split -> request and separate cause sandbox capture/refund.
- [ ] Golden calculation corpus/property/integration/abuse tests pass; simulations isolated from real payment/projection tables.
