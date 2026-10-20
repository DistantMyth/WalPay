# Development phases and task checklists

Planning baseline: 16 October 2026, Asia/Kolkata. This is a backend implementation plan, not a claim that these tasks or tests are complete. Every checkbox begins unchecked deliberately. Related quality gates and test IDs are in [the testing plan](10-testing-tools-and-quality-gates.md); provider capability gates apply before enabling a feature. Source/target amounts, fees and conversion gates follow [the global currency and FX plan](15-global-currency-and-fx.md).

The selected local-browser requirement is specified in [16-user-browser-agent-and-marketplace-automation.md](16-user-browser-agent-and-marketplace-automation.md). Its B0–B6 stages fit into the dates below. Reuse a verified existing driver first; a custom extension/browser engine is outside the initial delivery baseline. Actual browser action on a controlled store is part of the core demonstration, subject to proving exact selected profile/tab behavior.

## 1. Calendar and deliverable strategy

The current official submission cutoff is **12 November 2026 at 12:00 PST**, which is **13 November 2026 at 01:30 IST**. The working period from 16 October to 12 November is approximately 27 days. Set an internal submission target of **11 November at 18:00 IST**, leaving more than a day for upload, access and timezone problems. Keep the submitted build accessible through the end of judging, currently **15 December 2026 at 08:00 PST / 21:30 IST**. Recheck the official page before submission. [Official hackathon rules](https://paypalaihackathon.devpost.com/rules)

The rules require a real PayPal sandbox integration plus AI, a working build judges can run, a public repository with an open-source license, and a public YouTube demonstration shorter than three minutes. Sponsor tools are optional. Maintain reproducible setup and judge access for the submitted feature set. [Submission requirements](https://paypalaihackathon.devpost.com/rules)

### Scope tiers

| Tier | Deliverable | Meaning of done |
|---|---|---|
| Core A | Selected local-browser own-store discovery and bounded negotiation → immutable deal → local step-up → actual PayPal approval → capture → verified history | Complete actual-browser and real sandbox journey, failure/recovery cases, G0–G4/G3B pass |
| Core B | Receipt extraction with human correction → exact split → request links; QR cause → verified progress | Backend workflows and authorization tested; money moves only through the same proven Checkout path |
| Core C | USD-primary reporting with retained original currencies, indicative FX labels and fee-inclusive totals | No mixed-currency sum; executable conversion enabled only for an approved provider/corridor with bound quote |
| Stretch 1 | Freelancer invoices, verified partner task ingestion and payout operations, earnings attribution | One selected partner/PayPal capability genuinely enabled and sandbox-tested; otherwise clearly simulated/disabled |
| Stretch 2 | Channel3 discovery, Elastic retrieval/analytics, Zapier partner workflow, Astropods isolated agent deployment, Kernel authorized-store automation | Each has a useful, tested integration and evidence; no dependency can bypass money/approval policy |
| Later | Enabling restricted external marketplaces, binding bids/offers, broad P2P/FX, subscription/membership operations, merchant multi-seller settlement, offline NFC, frontend sponsor components | Access, commercial/region support and applicable partner permissions resolved first; user-browser adapter architecture and own-store action ship earlier |

The user-visible ambition remains broad; only advertise the subset actually built. A coherent core with a verified capture and recovery path is the submission baseline. Scheduling many integrations does not make those integrations enabled.

## 2. Team lanes, dependencies, and work packets

Assumption for calendar estimates: at least two focused backend implementation lanes plus one person handling provider/access/demo integration, with review capacity. If only one developer is available, preserve Core A and the shared security/quality gates, then select one Core B workflow. Do not remove approval/idempotency/reconciliation to retain feature count.

| Lane | Ownership | May proceed in parallel after contracts freeze | Requires review by |
|---|---|---|---|
| Platform | Fastify, auth, SQL, outbox, Redis/BullMQ, deployment, CI | Provider simulator, fixtures, OpenAPI rules, observability | Payments/security owner |
| Payments | PayPal adapter, intent/order/capture/refund, webhook verification, reconciliation, fees/FX boundary | Sandbox capability spikes; simulator adapter contract | Platform owner |
| Domain/AI | Store, missions, bounded seller agent, receipt/split/request/cause services | AI eval datasets, sponsor discovery adapter | Payments owner for side effects |
| Quality/integration | Postman workflows, race/fault/security suites, sponsor evidence, judge runbook | Seed data, test cases, submission setup | Someone who did not author the workflow |

Agent-generated work packets must specify allowed files/modules, input/output contract, invariants, test IDs, completion evidence and unresolved assumptions. A worker cannot change shared auth, money types or provider operation semantics as a local convenience. Merge shared schema/contracts before dependent feature branches; do not share one mutable database across development branches.

### Critical path

```text
Capability/region/access decisions
  -> money + auth + schemas + database
  -> payment intent + persisted operation + outbox
  -> actual PayPal approve/capture + real verified webhook
  -> reconciliation + race/fault/security gates
  -> core workflow integration + deployed demo
  -> reproducible judge build + evidence + submission

Parallel once contracts exist:
store/negotiation -> quote service -----+
receipt/split/request -----------------+-> same approved payment primitive
cause/QR/progress ---------------------+
partner/payout/invoice -> separate capability gate, never blocks Core A
```

## 3. Phase 0 — feasibility, access and contract decisions

**Dates:** 16–17 October. **Exit gate:** written capability table with real sandbox access evidence for core Checkout; unsupported paths explicitly disabled. No feature implementation depends on an assumed API.

### P0.1 Repository and planning decisions

- [ ] Inventory repository contents and existing implementation before creating scaffolding.
- [ ] Record team owner for each lane and how much implementation time is available.
- [ ] Select Node LTS, TypeScript, Fastify, PostgreSQL, Redis, BullMQ and package-manager versions that are mutually supported; pin exact resolved dependencies and container digests.
- [ ] Agree modular backend directory boundaries and naming conventions.
- [ ] Approve canonical Money shape with integer-string minor units, currency exponent table and upper size bounds.
- [ ] Resolve lower price semantics: default target/opening offer versus an explicitly requested hard minimum; never force a worse price.
- [ ] Decide first supported country/currency/corridor and sandbox account region based on team eligibility and provider support.
- [ ] Define USD-primary display/reporting, retained original settlement values and separate indicative/executable FX states.
- [ ] Define the complete total: item, shipping, tax, provider fee where disclosed, platform fee, conversion spread/fee where available.
- [ ] Record fee payer, rounding, refunds, tax treatment and net recipient expectations; do not infer legal/provider permission from a mock.
- [ ] Define core feature flags and the structured unsupported-capability response.
- [ ] Record architectural decisions with owner, date, alternatives, reason and revisit trigger.

### P0.2 PayPal sandbox and partner access

- [ ] Create/reuse dedicated sandbox buyer and business accounts; do not use real money.
- [ ] Create sandbox app; store secrets in a private secret manager/local ignored environment.
- [ ] Verify server OAuth token acquisition using sandbox host only.
- [ ] Create an order with a supported own-store currency; retain a scrubbed request/response.
- [ ] Complete real buyer approval and capture in a minimal spike.
- [ ] Register a real app webhook binding and inspect one actual capture event.
- [ ] Check whether required invoice/payout/marketplace/subscription/FX features are available for the account/region.
- [ ] Record unsupported/unapproved products and their fallback demo adapters.
- [ ] Create separate merchant/payee fixtures where the selected integration genuinely supports them.
- [ ] Resolve partner data/action rights for any external marketplace; otherwise own-store adapter remains the active negotiation surface.
- [ ] Verify that platform aliases/phone numbers are WalPay recipient-discovery identifiers, not a promise of arbitrary PayPal peer transfer APIs.
- [ ] Confirm that local WebAuthn step-up supplements PayPal approval.

### P0.3 Tooling and sponsor selection

- [ ] Create a public-safe Postman workspace/collection and private secret environment.
- [ ] Review the APIMatic PayPal Context Plugin compatibility with the actual coding harness; use documented context manually if direct harness support is absent.
- [ ] Select Render deployment/service types and estimate database/worker/Redis costs through judging.
- [ ] Request/test Channel3 access only if discovery is being enabled; record retail checkout capability separately.
- [ ] Record sponsor trial expiry, usage cap, license conditions and reproducible fallback in a private tracking issue.
- [ ] Limit sponsor spikes to 60–90 minutes each before core capture is demonstrated.

### P0.4 B0 existing-browser capability spike

- [ ] Pin official Playwright MCP and compatible browser extension; inspect actual tool schemas on the target Windows/Chrome/Edge versions.
- [ ] Use two team-controlled profiles and an already-open signed-in controlled-store tab to prove exact profile/tab attachment, before first observation.
- [ ] Identify driver defaults that choose the last-used profile; block those defaults until local account/profile/tab consent is verified.
- [ ] Demonstrate existing session reuse without copying browser profiles, reading cookie/password databases or exporting session tokens.
- [ ] Define the local companion/policy gateway as the only holder of raw driver access; cloud agent gets narrow typed tools only.
- [ ] Classify observe/navigation/local draft/reversible cart/send/binding offer/payment effects; MVP binding offers/bids stay disabled.
- [ ] Record site-policy capability separately from user's browser consent, including read/send restrictions for external marketplaces.
- [ ] If selected-tab isolation cannot be proved, record failed B0 and offer an explicitly chosen visible dedicated profile with manual login, or leave attachment disabled; never silently claim an equivalent cloud/profile-copy fallback.

**Required evidence:** capability matrix, safe `.env.example` shape, one scrubbed sandbox transaction record, initial OpenAPI outline, threat boundaries, and selected core demo story. Test target: G0 setup and a manual precursor of T-PAY-001/T-WH-001. If sandbox Checkout cannot work by this gate, the next action is account/configuration support resolution while the local simulator work continues; do not mislabel simulator payment as real sandbox completion.

## 4. Phase 1 — platform foundation and test harness

**Dates:** 18–20 October. **Dependencies:** P0 contracts. **Exit gate:** G0/G1 plus fresh-database integration gate; authenticated API can persist/retrieve a scoped intent draft without calling any payment provider.

### P1.1 Server and contract scaffold

- [ ] Initialize pnpm workspace/backend package and strict TypeScript compiler settings.
- [ ] Add Fastify app factory separated from process startup, so route tests use `inject`.
- [ ] Add environment schema, sandbox/live host allowlists, startup validation and feature-flag registry.
- [ ] Add request correlation, structured errors, request body size limits and redacted structured logging.
- [ ] Define OpenAPI security schemes, reusable Money/errors/pagination/idempotency headers.
- [ ] Add stable operation IDs for each planned core route.
- [ ] Add generated-contract check and Spectral rules for security/errors/money examples.
- [ ] Define health/live/readiness behavior and dependency timeout policy.

### P1.2 Persistence and authentication

- [ ] Add migrations for users, identities, sessions, device credentials, merchants, intents, authorizations, provider operations, webhook inbox, audit and outbox.
- [ ] Add foreign keys, scoped uniqueness, timestamps and revision fields.
- [ ] Add database transaction helper with explicit rollback/connection release behavior.
- [ ] Add session creation, rotation, expiry, revocation and CSRF policy where cookie auth applies.
- [ ] Add role and ownership policies applied by service methods, not only HTTP handlers.
- [ ] Add WebAuthn enrollment/assertion challenge storage, origin/RP ID rules, user verification policy and single-use expiry.
- [ ] Add minimal encrypted secret/token storage where a connected capability actually needs it.
- [ ] Add consent/visibility fields for alias/phone lookup; verification is distinct from possession of a raw contact value.
- [ ] Define object storage upload metadata, private access and scanning/quarantine state.

### P1.3 Durable jobs and evidence

- [ ] Implement transactionally written outbox records.
- [ ] Implement leased outbox dispatcher and uniquely identified BullMQ jobs.
- [ ] Define worker error categories: retryable, non-retryable and ambiguous provider outcome.
- [ ] Implement job trace correlation, backoff ceiling, retry ceiling and dead-letter/manual-review metadata.
- [ ] Ensure business operation state persists independently of Redis.
- [ ] Add graceful shutdown and restart recovery.

### P1.4 First automation gates

- [ ] Add Vitest projects, fast-check and Testcontainers fixtures.
- [ ] Build WireMock-backed provider contract simulator with explicit scenarios.
- [ ] Add auth/tenant fixture users and canary redaction data.
- [ ] Implement T-MONEY, T-OWN, T-DB-001..007 and test isolation teardown.
- [ ] Add CI jobs for lint/typecheck/contract/secret/unit/property/integration checks.
- [ ] Verify checks from a clean checkout on a second machine/CI runner.

### P1.5 B1 local connector and action protocol

- [ ] Add device pairing/confirmation/revocation records and routes from 16, with expiry, challenge binding and per-device credentials.
- [ ] Build signed local companion using structured process arguments and a pinned private stdio driver.
- [ ] Add outbound authenticated transport; test strict origin/authentication/CSRF rules for any unavoidable loopback endpoint.
- [ ] Implement locally approved site/tab/task grants and intersection with cloud mission policy.
- [ ] Define canonical action digest, sequence/session generation, document epoch, expiry, one-use approval and exclusive tab lease.
- [ ] Add protected minimal local command/result journal and stored-result replay behavior.
- [ ] Implement local stop/disconnect/revoke independently of cloud connectivity.
- [ ] Add deterministic fake driver and BRO-001..007/030 protocol/privacy fixtures.

**Required evidence:** passing G0–G2 reports for existing routes, database migration manifest, no-secret scan, ownership denial examples, queue restart example. Do not create a test-only approval bypass usable in deployed sandbox/production configuration.

## 5. Phase 2 — payment primitive and recovery

**Dates:** 21–25 October. **Dependencies:** P1. **Exit gate:** Core A payment infrastructure passes T-PAY/T-WH/T-RACE; one actual sandbox approve/capture/refund journey and clean reconciliation.

### P2.1 Intent, quote and local authorization

- [ ] Implement `POST /v1/payment-intents` with authenticated ownership and request schema.
- [ ] Bind immutable quote snapshot, merchant/payee, source, currency, fees, total, revision and expiry.
- [ ] Add budget reservation per source currency and atomic maximum checks.
- [ ] Add idempotency storage scoped by actor/operation with normalized payload hash.
- [ ] Return stable replay response and conflict for changed payload.
- [ ] Implement `/{id}/authorization-options` and `/{id}/authorizations` with single-use intent-bound challenge.
- [ ] Invalidate authorization after total/payee/quote/policy revision changes.
- [ ] Reject expired/revoked/foreign-user authorization.

### P2.2 Provider order and capture

- [ ] Implement OAuth token caching without exposing tokens to client/logs/models.
- [ ] Persist provider operation identity before external dispatch.
- [ ] Implement `/{id}/paypal-order` from server-authoritative quote values.
- [ ] Return only client-required approval data; validate return/cancel destinations.
- [ ] Obtain PayPal payer approval through supported flow.
- [ ] Implement `/{id}/capture` with policy, ownership, local proof and provider state checks.
- [ ] Preserve stable provider request ID for an operation retry within documented provider limits.
- [ ] Map pending/completed/denied/canceled/unknown without collapsing them into success.
- [ ] Implement server-authoritative capture history and detail responses.
- [ ] Implement `POST /v1/refunds` with authorization, refundable-amount constraints and distinct operation identity.
- [ ] Preserve original currency/amount for refund and reporting; platform fee refund behavior is explicit.

### P2.3 Webhook and reconciliation

- [ ] Implement `/v1/webhooks/paypal/{bindingId}` raw-byte body capture with size limit.
- [ ] Resolve configured app/environment/webhook binding; verify signature safely.
- [ ] Persist verified webhook inbox record before acknowledging delivery.
- [ ] Dedupe matching repeated events and quarantine conflicting duplicates.
- [ ] Correlate provider resource, amount/currency/payee and business operation.
- [ ] Handle out-of-order/delayed events without terminal state regression.
- [ ] Add retrieval-based reconciliation for unknown/pending operations with bounded polling and operator visibility.
- [ ] Add alert and runbook for unresolved operation age, provider failure rate and queue lag.
- [ ] Execute actual configured-app sandbox webhook proof; keep simulator proof separate.

### P2.4 Adversarial checks

- [ ] Implement T-PAY-001..005 and T-REFUND-001.
- [ ] Implement T-WH-001..011, including bad certificate URL and app binding mismatch.
- [ ] Implement concurrent same-intent capture with identical and different keys.
- [ ] Kill worker after provider acceptance and before SQL completion.
- [ ] Simulate accepted capture with dropped response, then retrieval convergence.
- [ ] Verify duplicate events never duplicate financial observations or notification requests.
- [ ] Validate that USD display/indicative FX cannot change provider charge currency.
- [ ] Save scrubbed G4 sandbox run manifest for the reviewed commit.

**Scope checkpoint on 25 October:** if Core A payment primitive is incomplete, defer all new sponsor runtime integrations and Stretch 1 features. Keep working on exactness, authorization and recovery.

## 6. Phase 3 — own-store agentic shopping

**Dates:** 26–29 October. **Dependencies:** P2 payment interface; store/catalog work can start during P1. **Exit gate:** own-store agent has an observable action beyond text recommendation, while every spend remains approved and bounded.

### P3.1 Catalog and seller adapter

- [ ] Add products, variants, inventory reservations, merchant-owned offers and shipping/tax quote interface.
- [ ] Seed at least five products and two seller policy profiles.
- [ ] Implement stock/price revision checks and reservation expiry.
- [ ] Implement authorized seller counteroffer adapter with round limit, minimum seller price and validity period.
- [ ] Store each offer/counteroffer with actor, timestamp, revision and evidence.
- [ ] Clearly label own-store negotiation versus external marketplace capabilities.
- [ ] Add product search with price/currency/category filters and bounded pagination.

### P3.2 Mission policy and AI tools

- [ ] Implement `POST /v1/shopping-missions` with explicit target/upper cap/currency, category, item constraints, deadline and permitted sellers.
- [ ] Define typed read/discover/negotiate/quote tools and separate side-effecting payment tools.
- [ ] Exclude provider secrets and unnecessary contacts/address from model input.
- [ ] Apply schema validation and deterministic policy checks to every tool call.
- [ ] Recheck policy/revocation/quote completeness at each financial action boundary.
- [ ] Bound model tokens, tool calls, negotiation rounds, execution time and per-user cost.
- [ ] Add durable pause-for-user, canceled, expired, retryable-failure and manual-review outcomes.
- [ ] Produce a final deal explanation using stored evidence: price, fees, savings basis, seller, currency, expiry and capability.
- [ ] Require user-selected final intent and supported PayPal approval before capture.
- [ ] Add tool invocation audit with redacted inputs/outputs and policy rejection reason.

### P3.3 Evaluation and first full demo

- [ ] Implement E-AI-001..005, 009..015 and 017 before expanding model/tool permissions.
- [ ] Run task-success and adversarial sets with reviewed ground truth.
- [ ] Demonstrate allowed cheaper-than-target offer and blocked fee-inclusive over-cap offer.
- [ ] Demonstrate prompt injection in seller/product text cannot change budget/payee.
- [ ] Demonstrate cancellation while negotiation is active.
- [ ] Create a full Postman/server journey up to the PayPal approval boundary; use actual browser payer approval for sandbox proof.
- [ ] Verify a final captured payment appears once in the user's history and merchant earnings projection.
- [ ] Test provider timeout and visible reconciliation in the same journey.

### P3.4 B2–B4 actual selected-browser shopping and negotiation

- [ ] Pair the user's selected controlled-store browser profile/tab through `/v1/browser-task-sessions`; never attach all tabs implicitly.
- [ ] Implement bounded observation/navigation/product extraction with local redaction and provenance.
- [ ] Revalidate origin, selected tab, document epoch, recipient/thread/listing and mission revision before every action.
- [ ] Draft seller messages in local/WalPay memory; do not fill an autosaving external composer while calling the action draft-only.
- [ ] Require exact nonbinding message text/amount/currency/scope/recipient digest and one-use user send authorization.
- [ ] Reject generic clicks on binding offer/bid/buy controls and fee-inclusive over-cap proposals; unknown fees allow only an explicitly nonbinding inquiry.
- [ ] Store dispatch before sending; verify actual thread message evidence after send.
- [ ] Simulate lost send acknowledgement, crash and reconnect; reobserve/reconcile or request manual resolution, never blind resend.
- [ ] Pause on human input, origin/tab/thread drift, login/OTP/CAPTCHA and provider approval; require fresh safe state before resume.
- [ ] Hand the final own-store quote to the shared backend intent/approval/capture flow; browser never controls PayPal authentication or captures by clicking.
- [ ] Demonstrate revoke/disconnect stops future actions and preserves already-started effect uncertainty.
- [ ] Run BRO-008..025/027..029 and malicious-page/seller/screenshot evaluations before enabling the browser demo.

**Required evidence:** Core A full workflow report, model/eval manifest, one reviewed safe negotiation trace, one safe policy rejection, actual sandbox evidence, and an honest external-platform capability status.

## 7. Phase 4 — receipt, split, requests, QR cause and currency views

**Dates:** 30 October–2 November. **Dependencies:** shared payment primitive and scoped identity services. **Exit gate:** Core B/C API journeys work with exact totals, recipient confirmation and verified projections.

### P4.1 Receipt processing

- [ ] Implement private upload initialization, metadata and signed object access.
- [ ] Validate size, media type/magic bytes, page/image limits and scan/quarantine state.
- [ ] Add asynchronous extraction with typed provisional output, currency, line items, confidence and source coordinates where available.
- [ ] Check subtotal/tax/tip/discount/total consistency without inventing missing values.
- [ ] Require correction/confirmation endpoint before creating a payable split.
- [ ] Preserve extraction/review versions and avoid transmitting the whole receipt to unrelated services.
- [ ] Implement E-AI-006..008/016 and T-UPLOAD/T-RECEIPT tests.

### P4.2 Splits and money requests

- [ ] Implement equal, weighted and itemized splits with deterministic remainder allocation.
- [ ] Enforce sum equals confirmed receipt total and supported single settlement currency per split.
- [ ] Store participant consent, membership, allocation revision and request lifecycle.
- [ ] Implement opted-in alias/contact search with verification, throttling and block/report policy.
- [ ] Create signed expiring payment-request links; accepting never silently debits a payer.
- [ ] Deduplicate request creation and notification jobs.
- [ ] Add partial settlement/outstanding balance projection by original currency.
- [ ] Implement cancel/decline/expire/remind with per-recipient anti-spam limits.
- [ ] Treat NFC as transfer of the same signed reference; backend does not assume possession is identity/approval.
- [ ] Implement T-SPLIT/T-REQUEST/T-CONTACT/T-QR cases.

### P4.3 Causes and verified progress

- [ ] Add cause ownership, beneficiary mapping, status, currency, goal and moderation/admin fields.
- [ ] Generate signed QR references without embedding mutable trusted financial data.
- [ ] Resolve cause/payee server-side; show current amount/fee/currency/beneficiary before approval.
- [ ] Create contribution intent through the existing Checkout primitive.
- [ ] Update captured/pending/refunded/net projections idempotently.
- [ ] Add a scoped progress read API and event stream/poll cursor for future clients.
- [ ] Exclude donor identity from public data by default; support consented display name separately.
- [ ] Test duplicate capture/refund events and goal-overflow presentation metadata.

### P4.4 Currency and fees

- [ ] Add primary display currency USD and original amount fields to earnings/history responses.
- [ ] Add indicative rate provenance/time and distinguish missing/stale estimate.
- [ ] Keep source, presentment, settlement and reporting amounts separate.
- [ ] Use decimal/integer arithmetic with explicit rounding and exponent rules.
- [ ] Display provider/platform/conversion fees as named line items and avoid undisclosed simulated fees.
- [ ] If executable FX is approved, store immutable source/debit/destination/net/rate/fee/expiry quote, bind authorization and recheck on dispatch.
- [ ] If no provider/corridor is approved, return unavailable for executable P2P FX while preserving display conversion.
- [ ] Implement T-FX cases: expiry, changed fees, insufficient target net, unsupported corridor, concurrent reservations and duplicate execution.

**Scope checkpoint on 2 November:** freeze feature inventory. Choose one stretch integration only if Core A/B/C gates are green and there are at least two full days reserved for quality work.

## 8. Phase 5 — selected business workflow and useful sponsors

**Dates:** 3–6 November. **Dependencies:** enabled capability evidence, Core A/B/C green. **Exit gate:** every selected stretch capability has a tested real integration or is explicitly disabled; no new safety invariant is weakened.

### P5.1 Freelancer and partner earnings, conditional

- [ ] Add clients, tasks/milestones and attribution records with tenant/partner ownership.
- [ ] Add partner service credentials, key rotation and replay protection.
- [ ] Define signed task event schema, immutable event ID, task revision and approved amount/currency.
- [ ] Verify partner authenticity and compare agreed task amount/payee/contract before creating payout obligation.
- [ ] Require partner/platform authorization and funding/capability check; a task approval is not arbitrary payout authorization.
- [ ] Persist payout business operation before dispatch; unknown outcomes enter reconciliation.
- [ ] Implement provider-supported sandbox payout only if enabled; otherwise mark simulated payout state explicitly.
- [ ] Add invoice draft/issue/status adapter only if sandbox access is demonstrated.
- [ ] Aggregate earnings by client/task/partner and original currency; distinguish gross, fees, net, pending, paid and reversed.
- [ ] Show USD report estimates separately from settled original-currency facts.
- [ ] Test task approval duplicate/revision/conflict, revoked partner, wrong recipient, insufficient funds, provider pending/failure and timeout after acceptance.

### P5.2 Sponsor tasks, selected from the integration plan

- [ ] **Postman:** publish sanitized run collection with assertions; run Newman in CI.
- [ ] **APIMatic:** record actual PayPal context usage and reviewed adapter output; do not call generated code evidence of executed integration.
- [ ] **Render:** deploy API, long-lived worker, private SQL/Redis and secret references; test restart/readiness.
- [ ] **Channel3:** call real discovery API, normalize offers and capability flags, test stale price/timeouts; external buy link does not imply PayPal checkout/negotiation support.
- [ ] **Elastic, optional:** ingest minimized tenant-scoped event projections through outbox and implement one useful retrieval/aggregation; test isolation and reindex.
- [ ] **Zapier, optional:** connect one consented test partner workflow; validate ingested event against authenticated partner contract; no generic model-triggered arbitrary payouts.
- [ ] **Astropods, optional:** host one read/propose agent with scoped tools; compare actual trace and fallback; financial state remains in backend SQL.
- [ ] **Kernel, optional/B5:** automate the team's authorized store fixture in a separately consented cloud session with bounded navigation and evidence; this is distinct from the selected existing local-browser connector and cannot import local sessions silently.
- [ ] **AG Grid / Bryntum:** record future authenticated backend read contracts only; actual component implementation remains deferred to frontend planning.
- [ ] Add per-sponsor feature flags, budgets, test evidence and trial-expiry mitigation.
- [ ] Keep optional Browser Use reasoning/Hermes harness behind the same narrow local action gateway; no independent raw browser/terminal authority.

**Cut rule:** if a sponsor requires a second financial execution system or a new wallet/escrow abstraction, defer that part. Postman/APIMatic/Render are useful without changing payment semantics. Read-only integration failures must not interrupt capture reconciliation.

## 9. Phase 6 — hardening and release candidate

**Dates:** 7–9 November. **Dependencies:** feature freeze. **Exit gate:** G0–G5 pass for every advertised capability; open defects are explicitly triaged.

- [ ] Run all monetary, FX, approval, ownership and webhook invariant tests.
- [ ] Run multi-instance capture/refund/payout races against real PostgreSQL and simulator.
- [ ] Inject accepted-provider-response loss, SQL commit ambiguity, Redis restart, worker kill and provider 429/503.
- [ ] Verify reconciliation does not create a new payment/payout after an ambiguous response.
- [ ] Run authenticated Schemathesis and targeted negative business API tests.
- [ ] Run ZAP against the isolated fake-provider staging deployment.
- [ ] Run Gitleaks over the full public repository history; rotate any exposed key before submission.
- [ ] Review dependency/static findings and remove debug/test bypasses.
- [ ] Execute AI adversarial suite on exact shipped model/prompt/tool versions.
- [ ] Complete B6/G3B with actual pinned local driver/profile/tab evidence, BRO-001..030, message/digest/recipient drift, malicious seller and local journal crash/reconnect cases.
- [ ] Verify invalid/revoked grants and unsupported site policies cause zero unauthorized reads/sends; exact send approval never authorizes a binding offer or PayPal auth action.
- [ ] Verify local stop works offline, raw driver/CDP endpoints are inaccessible to agent workers, and no secret browser data appears in cloud traces.
- [ ] Run bounded k6 read/write/webhook/queue-recovery scenarios and record environment.
- [ ] Rehearse backup restore into an egress-restricted environment.
- [ ] Test tenant-scoped export/deletion/retention and sensitive log/search canaries.
- [ ] Review fee/net reporting and partial refund/currency edge cases.
- [ ] Test language/error code negotiation, timezone fields, Unicode input and accessible-notification data contracts.
- [ ] Inspect health/readiness, alert thresholds, error dashboards and dead-letter/manual-review queues.
- [ ] Test rolling deployment compatibility and graceful worker shutdown.
- [ ] Repeat actual PayPal sandbox journey for final adapter commit/configuration.
- [ ] Classify defects: P0 money/security/data-loss; P1 advertised workflow unusable; P2 bounded degraded UX; P3 later enhancement.
- [ ] Close all P0/P1 defects before demo freeze; feature-flag failing optional capabilities off.
- [ ] Create immutable release candidate tag and evidence manifest.

## 10. Phase 7 — judge reproducibility and submission

**Dates:** 10–11 November; 12 November is contingency. **Exit gate:** a fresh user can reproduce the submitted core story without team intervention, with safe sandbox access.

### P7.1 Build and access

- [ ] Check public repository includes source, migrations, synthetic seed data, lockfiles, OpenAPI, scripts and open-source license.
- [ ] Add exact runtime/container prerequisites and clean setup commands.
- [ ] Document sandbox configuration and how judges obtain required access without publishing team secrets.
- [ ] Supply dedicated restricted demo users and sandbox instructions through the appropriate submission access mechanism.
- [ ] Verify hosted demo URL/API availability and SSL certificate health.
- [ ] Verify demo data reset does not interfere with judge sessions or erase unresolved financial operations.
- [ ] Confirm sponsor credits/trials/quotas last through judging or budget for replacement.
- [ ] Test clean local installation from published repository using README only.
- [ ] Include support/runbook for unavailable external AI/provider service and distinguish degraded mode.

### P7.2 Evidence and claims

- [ ] Create feature truth table: real sandbox, local simulator, disabled, or future for every advertised feature.
- [ ] Link scrubbed test/sandbox manifests to exact release commit.
- [ ] Demonstrate one agent action, one actual PayPal approval/capture, one bounded policy rejection and one useful resulting record.
- [ ] Explain original problem/audience and show what the backend actually enables.
- [ ] Record English demonstration shorter than three minutes; verify the public YouTube link in a signed-out browser.
- [ ] Verify public repository URL, hosted URL and setup instructions in the submission form.
- [ ] Select sponsor claims only when actual usage/evidence meets their current prize rules.
- [ ] Name supported countries/currencies/payment routes precisely; do not claim general UPI interoperability or arbitrary PayPal wallet transfer/FX APIs.
- [ ] Have another team member review final text against the actual build.
- [ ] Submit by internal target 11 November 18:00 IST and verify submitted status.
- [ ] Recheck official cutoff and capture submission receipt privately.

### P7.3 Judging-period upkeep

- [ ] Assign a person to maintain availability through the official judging end.
- [ ] Keep submitted release reproducible and preserve its evidence.
- [ ] Monitor provider/AI quota expiry and hosting credits without changing advertised behavior silently.
- [ ] Rotate/restrict demo credentials as necessary while preserving judge access.
- [ ] Track any post-submission operational repair and comply with the competition's modification rules.
- [ ] Remove/disable demo access and unnecessary retained data after the judging requirement ends, according to the published retention policy.

## 11. Feature-wide implementation ticket template

```text
Ticket ID and name:
Tier / owner / reviewer / estimated effort:
Dependency IDs and provider capability:
User-visible outcome:
Allowed modules/files and API operation IDs:
Inputs, outputs, errors and idempotency semantics:
Authorization, currency/fees and policy invariants:
Database migration/constraints and rollback/recovery plan:
Outbox/jobs/webhook/reconciliation behavior:
AI tools/prompt scope, if any:
Test IDs + fixture IDs + expected database/provider effects:
Observability, redaction, usage/cost limit:
Feature-off / provider-unavailable behavior:
Evidence: commit, commands, reports, sandbox manifest:
Known limitations and public demo wording:
```

Ticket completion checklist:

- [ ] Small enough for one reviewable change with an observable outcome.
- [ ] Success and rejection criteria are written before implementation.
- [ ] API schema/example/error behavior and migration are included where required.
- [ ] Negative ownership, idempotency and provider timeout cases are covered.
- [ ] Unit tests verify the rule; integration tests verify the actual storage/network boundary.
- [ ] Meaningful tests run for the change; output is saved accurately, including failures.
- [ ] No unchecked model output can authorize a financial side effect.
- [ ] No secrets, private receipts or personal contact fixtures enter the public repo.
- [ ] Reviewer can explain recovery after a crash at the external action boundary.
- [ ] Feature flag and operational fallback exist when an external dependency can fail.
- [ ] Scope/README/demo truth table is updated.

## 12. Schedule risks and decisions

| Trigger | Required decision | What continues safely |
|---|---|---|
| PayPal product/region not available | Disable affected capability; choose documented supported sandbox account/product | Core own-store Checkout, simulation-only adapter tests |
| External marketplace access not authorized | Defer autonomous external action | Own-store seller agent and permitted discovery |
| Existing signed-in profile/tab isolation cannot be proved in B0 | Keep that connector unavailable; explicitly select supported visible dedicated profile/manual login or defer | Controlled-store API path and honestly labeled alternate browser mode; no silent cloud/profile-copy substitution |
| Browser send outcome ambiguous after disconnect | Pause, inspect exact authorized thread/local journal and reconcile/manual review | No blind resend; future actions remain stopped until resolved |
| Executable FX/P2P provider unavailable | Keep indicative USD views; disable execution for unsupported corridor | Exact original-currency Checkout and earnings |
| Core payment tests fail on 25 October | Drop new stretch work | Recovery/authorization/webhook correctness |
| OCR quality poor | Preserve manual correction; reduce claimed extraction coverage | Exact splits from confirmed data |
| Agent safety eval fails | Reduce tool permissions or disable affected action | Deterministic service/API flow with human review |
| Sponsor trial ends before judging | Obtain sufficient funded plan/license, self-host where permitted, or disable optional feature and align claims | Reproducible core using stable dependency contracts |
| Load target missed | Diagnose index/pool/queue limits; reduce advertised/demo concurrency if justified | Correct bounded traffic; no silent data loss |
| Team time reduced | Retain Core A plus one secondary workflow | Shared safety, reproducibility and evidence gates |

Checkpoints are scope decisions, not requests to bypass provider or user approval. An integration's commercial activation can remain future work while the plan, feature flag, simulator and honest sandbox evidence are complete.
