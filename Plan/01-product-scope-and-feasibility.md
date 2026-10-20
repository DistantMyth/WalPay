# Product scope, priorities and feasibility

Baseline: 16 October 2026. This is a proposed global backend, USD-first, using PayPal sandbox for hackathon execution. Availability depends on account, product, country, currency and partner permissions. A global user directory must never be mistaken for universally available money movement.

## Capability labels

| Label | Meaning | Release behavior |
|---|---|---|
| S — sandbox implementable | Documented provider path, still requires account setup and actual verification | Enable only after recorded sandbox success |
| G — gated | Product approval, partner agreement, account entitlement or jurisdiction review required | Default off; expose reason and next action |
| W — WalPay workflow | WalPay can implement records, calculations and coordination itself | Payment action still chooses an eligible provider path |
| M — simulation | Controlled fake partner or fixture, explicitly labelled | Never appears as a real external deal or transfer |
| U — unsupported assumption | No verified generally available route for requested behavior | Do not create a substitute payment claim |

## Feature inventory and commitment

| User request | Backend implementation | Priority / status | Proof |
|---|---|---|---|
| Autonomous shopping | Mission, discovery adapters, comparison, durable jobs, policy checks | Core W | Mission trace plus selected quoted product |
| Strict upper/lower limits | Lower target/opening strategy, hard all-in cap, budget reservation | Core W | Boundary/property/concurrency tests |
| Negotiation | Own-store seller rules and authenticated offer protocol | Core W/S | Recorded exchange and accepted unexpired quote |
| eBay discovery/checkout | Official APIs and user handoff according to approved access | Later S/G | Actual endpoint and entitlement report |
| eBay/Facebook bargaining | No generic buyer bargaining API assumed; consented manual handoff or fake partner | M/G/U | Simulation flag and partner support record |
| Agent uses user's existing browser | Selected signed-in tab through existing Playwright MCP extension and restricted local gateway; discovery, draft offers and approved site actions | Core browser spike W/G | Browser consent/action traces; own-store end-to-end test;16 |
| PayPal wallet connection | Separate identity connection, merchant/payee onboarding, buyer approval | Core S/G | Scopes and sandbox order/capture evidence |
| PIN or biometrics | Passkey user verification; PIN fallback as described in 09 | Core W | Bound authorization verified by backend |
| Handles/phone payment initiation | Consented recipient resolution plus capability-routed intent | W/G | Recipient confirmation; no generic P2P claim |
| Fast transfers | Reduce application delay; provider settlement time stays visible | W/G | Task-to-dispatch and dispatch-to-settlement separately |
| Creators/freelancers/gig earnings | Partner provenance, invoices, payouts and per-client earnings | Core extension S/G/W | Signed task approval and payout item evidence |
| Partner task-approved payouts | Authenticated event, eligibility/funding gate, exactly-once business effect | Core extension S/G | Replay and race tests plus sandbox item |
| Split after paying | Group obligations tied to confirmed original expense | Core extension W | Conservation and partial repayment tests |
| Scan receipt/QR | Secure upload, extraction draft, human correction, verified QR resolution | Core extension W | Reviewable extraction and revision record |
| Contacts/NFC | Exact consented directory lookup, opaque invitation token transport | W | No phone book enumeration; QR-equivalent payload |
| Requests/reminders | Obligations, consented invitations, expiry, accept/decline/block | W/G | Recipient-specific request state; supported payment rail |
| Multicurrency invoices | Currency-specific immutable invoice snapshots | Later S/W | Provider invoice and paid-state reconciliation |
| Creator tips/memberships | Approved merchant checkout/subscriptions, entitlements after confirmed events | Later S/G | Subscription renewal/cancel event tests |
| Marketplace automatic split | Eligible seller onboarding and approved multiparty settlement | G | Partner access and per-seller settlement evidence |
| Merchant POS add-on | Order/receipt adapter, signed QR, webhook reconciliation | Later W/S/G | Merchant-scoped order and paid receipt |
| Causes/donation progress | Campaign metadata and confirmed payment/refund projection | Later W/S/G | Progress replay and duplicate-event tests |
| Multilingual/accessibility | Stable message codes, locale data, accessible summaries, optional voice | Core W | Language tests and future assistive-client acceptance |
| USD-primary conversion with charges | Original and reporting money, quote disclosure and supported provider execution | Core estimate; execution G | Quote expiry/rounding and settled currency evidence |
| P2P currency exchange | Separate approved-partner product, no custody or self-created matching settlement | Future G | Legal/product/FX entitlement review before enabled |
| Usual UPI conveniences | History, search, favourites, bills, mandates where supported, disputes, exports | W/G | Full inventory in 07; rail-specific capabilities |

## Three demonstration stories

### A. Buy within an agreed budget

Input: a user wants a desk accessory, has a USD 35 target, a USD 50 absolute all-in cap, and shipping destination sufficient for tax/shipping. The mission discovers own-store products and optionally Channel3 catalog matches. Own-store negotiation returns item USD 40 + shipping USD 4 + tax USD 3 + disclosed service fee USD 1 = USD 48. The user reviews exact seller, item, total, expiry and cancellation/refund terms, verifies with a passkey, and separately completes PayPal approval. The backend captures and reconciles before releasing fulfilment.

Acceptance: a USD 50.01 quote is rejected; a price revision invalidates earlier user authorization; repeated capture calls produce one observed capture; agents cannot bypass the checks. A trace makes AI contribution visible without exposing the user's private maximum to the seller.

### B. Turn a receipt into fair obligations

Input: a confirmed expense plus a receipt image. Extraction returns tentative lines and confidence. The user corrects missing tax and assigns participants. Integer allocation conserves the paid total, and the user confirms a new revision. Invited friends see their own obligations and can accept, dispute or settle through an eligible payment route. An unsupported reimbursement rail produces request tracking/manual handoff, not a fake successful transfer.

Acceptance: OCR alone never creates requests; duplicate requests do not double debt; a refund updates obligations using an explicit allocation policy. The original bill and later reimbursements remain different transaction categories.

### C. Approved work becomes an attributable payout

Input: a demo partner approves a specific version of a USD-denominated job through a signed event. The backend validates signer, task, worker, amount, budget, funding and payout capability; it creates one obligation and queues one PayPal business payout operation. The worker's earnings record shows partner, task, gross, fees if applicable, net, currency and item status.

Acceptance: a replay cannot pay twice; a batch accepted response does not show the worker paid; a timeout enters recovery; a reversed or rejected job cannot erase an already executed payout. The demo partner is labelled a simulator unless actually integrated.

## Scope allocation

Core: identity, capabilities, own-store catalog/quote negotiation, durable payment intent, passkey step-up, PayPal approval/capture, signed webhooks, observation journal, history and recovery. These share one database and payment module.

Next: receipt split coordination and one business payout partner. Then add invoices, creator membership, cause campaigns and selected sponsors. Defer broad marketplace write automation, simultaneous multi-seller settlement, arbitrary instant P2P, production peer FX, credit/loans and bank-account aggregation until their dependencies are real.

## Country and currency policy

Store country facts by source: user residence claim, verified merchant provider country, payout receiver eligibility, shipping destination and funding provider. They are different facts. A capability decision records both source and decision version. Never use VPN location or selected UI language to decide a payment corridor.

Maintain an allowlist per operation, payer/payee account type, country pair and currency. Use USD demo accounts in a permitted sandbox configuration. Never tell a user to misstate residence to unlock a product. PayPal's current India guidance disallows domestic payments; see the exact evidence and corridor treatment in 04/08. USD-first reporting does not remove those restrictions.

## Success measurements

Proposed demo gates: zero money invariant violations; 100% of negative approval/budget/replay safety cases pass; all external mutations have durable operation records; every success is backed by a verified provider result; every simulation is labelled in the API. Product metrics: mission-to-valid-quote rate, savings versus independently sourced comparable total, receipt correction rate, payout approval-to-dispatch delay, and unresolved-operation age. These are proposed measurements, not measured achievements.

## Scope checklist

- [ ] Assign every requested feature one of S/G/W/M/U and retain the reason.
- [ ] Confirm global USD-primary behavior and explicit supported corridors.
- [ ] Confirm whether lower cap means preferred target or minimum acceptable quality/price constraint; default to target, never mandatory spend.
- [ ] Record fees before asking for financial authorization.
- [ ] Demonstrate actual sandbox payment at least once before adding stretch integrations.
- [ ] Include unsupported capability responses and manual handoff acceptance tests.
- [ ] Prevent simulated state from entering real provider queues.
- [ ] Keep sandbox and live credentials, accounts, operations and webhook identities separate.

## Research basis

The competition asks for meaningful PayPal and AI usage and a functional build. Its resource examples span shopping agents, task-triggered payouts, expense splitting, business payments and accessible giving; these inform the product's journeys, not a claim that every example is an unrestricted API. [Hackathon overview](https://paypalaihackathon.devpost.com/), [resource examples](https://paypalaihackathon.devpost.com/resources). Exact provider feasibility sources are attached to the specialist documents.
