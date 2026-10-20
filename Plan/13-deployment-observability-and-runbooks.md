# Deployment, observability, operations and runbooks

Proposed operational plan. No services are provisioned by this document. The hackathon build uses sandbox credentials and a clearly identified test environment. Live deployment is a separate release gate with actual account/corridor approvals.

## Environments and topology

| Environment | Dependencies | Credentials / outbound permissions |
|---|---|---|
| local simulation | Docker PG18, Redis-compatible queue, private object emulator, fake provider, seeded store | Fake keys only; external money hosts denied |
| CI integration | Ephemeral Testcontainers PG/queue plus fake provider | No developer secrets; deterministic failures |
| developer sandbox | Isolated database, HTTPS ingress/tunnel with access controls, PayPal sandbox | Developer's sandbox app; only permitted test accounts |
| hosted demo | Render API/worker, managed PG, persistent compatible queue, private object storage | Separate sandbox app/webhook; bounded demo identities |
| staging/live candidate | Production-equivalent isolation and read-only capability probes | Actual entitlements, no unintended financial mutation |
| live | Independently approved release and separate data/secrets | Only enabled product/country/currency/site allowlists |

Deployment diagram: API and worker share reviewed domain code but run with different database/secret privileges. Payment worker can access configured PayPal binding; OCR worker cannot. Local user-browser connectors use outbound authenticated connections to the backend and scoped local tools (16), never public unauthenticated debugging ports. Cloud browser sessions are a different consent/deployment mode.

Render Key Value currently runs Valkey8 for newly created instances and offers paid persistence; validate BullMQ compatibility and use queue-appropriate `noeviction`. Do not rely on a free nonpersistent queue for reliable demo operation. [Render Key Value](https://render.com/docs/key-value). PG outbox remains recoverable authority if queue transport is lost.

## Configuration and secret inventory

Proposed required variables (values never committed): `APP_ENV`, `MONEY_MODE`, `PUBLIC_API_ORIGIN`, `ALLOWED_CLIENT_ORIGINS`, `DATABASE_URL`, `QUEUE_URL`, `OBJECT_BUCKET`, `OBJECT_REGION`, `OBJECT_ENDPOINT` where required, `WEBAUTHN_RP_ID`, `WEBAUTHN_ORIGINS`, `PAYPAL_API_ORIGIN`, `PAYPAL_CLIENT_ID`, `PAYPAL_SECRET_REF`, `PAYPAL_WEBHOOK_BINDING_ID`, `MODEL_PROVIDER`, `MODEL_ID`, `MODEL_SECRET_REF`, `CONTACT_LOOKUP_KEY_REF`, `SESSION_PEPPER_REF`, `OBSERVABILITY_ENDPOINT`. Add per-partner signing key references and rotation versions. A schema validates configuration at startup.

Validate combinations: simulation cannot target PayPal hosts; sandbox must use sandbox credentials/API origin/webhook identity; live mode requires deployment-level gate and approved binding. Do not permit a request parameter to override these. Session signing/lookup/encryption keys are distinct; tenant provider secrets use envelope encryption or managed secret storage. API client ID and secret have different exposure rules.

Rotation: create new secret version -> canary capability/read-only test -> switch new operations -> keep old binding usable for reconciling already submitted operations -> revoke only after obligations complete -> audit. Do not rotate webhook IDs or signing keys in a way that makes delayed events unverifiable. Partner overlap windows are finite and versioned.

## Deployment procedure

1. CI compiles/lints/types, validates OpenAPI/examples, runs required deterministic tests and produces immutable image digest/SBOM.
2. Scan image and dependencies; triage reachable high-risk findings. Pin base-image digest and nonroot user.
3. Back up PG and confirm recent restore evidence. Apply expand-only migration with independent migration credentials and bounded lock time.
4. Start canary API/worker with money submission disabled; verify liveness/readiness, DB constraints, queue capability, private storage and secret redaction.
5. Perform capability/read-only provider probe; run controlled sandbox smoke transaction only under explicit test plan using test accounts.
6. Enable new submissions for demo cohort; keep reconciliation workers running through rollout.
7. Observe error/unknown/queue metrics and run acceptance journey. Record image/migration/config/contract/model/fee-policy versions together.
8. Roll back application using backward-compatible schema if needed; never reverse a captured payment or delete evidence as a deployment rollback.

## Observability contracts

Logs: structured JSON containing timestamp, severity, service/version, request/correlation/operation ID, tenant pseudonym, operation kind, mode, canonical status, provider error category and duration. Redact credentials, cookies, OAuth codes, phone/email/address, raw OCR, entire webhook bodies and seller chat. Keep provider debug IDs only if safe and useful. Avoid high-cardinality IDs in metric labels.

Traces connect API -> outbox -> job -> external provider -> webhook -> reconciliation through stored correlation IDs. Model traces record model/version, prompt schema version, tool name, latency/token/cost counters, evaluation class and policy outcome; do not require hidden reasoning. Browser traces record scoped task/action/proposal/version and redacted observation hashes. Access is role-restricted.

| Metric | Proposed alert / action |
|---|---|
| `payment_unknown_count/oldest_age` | Any unknown older than5min warns; older than30min pages operator; never auto-repay |
| `unreconciled_provider_difference` | Any amount/payee/currency mismatch freezes affected binding |
| `outbox_oldest_age`, `queue_wait_p95` | More than60s for money queue warns; sustained5min blocks new promises |
| `webhook_verification_failures` | Sudden increase investigates binding/signature/attack; no bypass |
| `payout_item_failure_rate` | Threshold per partner; pause affected partner and keep other recovery running |
| `budget_invariant_violation`, `journal_unbalanced` | Any occurrence is release blocker and emergency halt |
| `browser_scope_denied`, `unexpected_navigation` | Pause task, present re-consent if appropriate |
| `model_cost_daily`, `agent_rounds` | At configured budget stop new reasoning; preserve payment recovery |
| `fx_rate_age`, `quote_expired_rejections` | Disable stale estimates/executable quoting appropriately |
| `storage_scan_backlog`, `export_age` | Degrade extraction/export while payment core remains usable |

Thresholds are initial policies to calibrate with tests; they are not promised provider SLAs. Latency dashboards distinguish local dispatch from provider completion and withdrawal/bank settlement.

## Runbook A: capture timed out

- [ ] Locate immutable operation by request/intent ID; mark/retain `outcome_unknown` and freeze its terms/reservation.
- [ ] Disable fresh submission for that operation; do not create a new PayPal request ID.
- [ ] Inspect provider order/capture using configured binding and known IDs; correlate verified inbox events.
- [ ] If confirmed completed with exact merchant/amount/currency, persist one observation, complete journal/reservation and fulfil once.
- [ ] If definitively not executed, follow endpoint-specific safe same-operation retry window; if inconclusive keep unknown and escalate.
- [ ] Inform user through state projection with last checked time and next update; never show failed+retry while uncertain.
- [ ] Record evidence/reason for manual resolution and protect against late duplicate events.

## Runbook B: queue/cache outage

- [ ] Keep PG accepted commands/outbox; expose delayed dispatch and stop accepting beyond backlog limit.
- [ ] Restore queue service/configuration and confirm `noeviction`/persistence/authentication.
- [ ] Re-enqueue eligible outbox/operations with stable IDs; workers consult PG terminal state.
- [ ] Reconcile previously submitted operations before considering resubmission.
- [ ] Verify no double capture/payout and no lost accepted obligation; record recovery time.

## Runbook C: bad partner event or compromised seller/browser content

- [ ] Pause partner/site connector and revoke active delegation; do not stop global reconciliation.
- [ ] Quarantine unverified/conflicting event/content; preserve hash, signer and source metadata.
- [ ] List operations initiated by affected principal/time window and inspect provider state.
- [ ] Stop pending unsent actions; already submitted operations need verified status and permitted refund/reversal process.
- [ ] Rotate relevant credentials after evidence capture; review access scope and missed tests.
- [ ] Communicate actual affected state; do not automatically message sellers or disclose personal data.

## Runbook D: payout partially failed

- [ ] Inspect item statuses, not just batch header. Separate succeeded, pending, failed, unclaimed/returned and unknown.
- [ ] Mark earnings obligations accurately; do not retry successful items.
- [ ] Resolve receiver/funding/currency issue with verified account owner and any required fresh mandate.
- [ ] A corrected destination creates a new approved obligation version for unpaid amount only; preserve historical submitted receiver.
- [ ] Use safe documented retry/new-operation policy and record linkage; replay original job approval never creates another obligation.

## Backups and disaster recovery

Proposed demo objectives: DB RPO<=1h and RTO<=4h using chosen paid backup capabilities; live objectives require separate cost/capability decision. Enable encrypted managed backups/PITR where purchased and private versioned object retention where appropriate. Test restoration to an isolated environment, disable all outbound financial mutations before starting workers, validate counts/constraints/inbox/provider-resource uniqueness, then reconcile provider facts before resuming. A backup restored before a captured payment must not resubmit that payment.

Store deployable image, schema version, contract, seed recipe and minimal redacted demo fixture offline/private as appropriate. Do not back up plaintext secrets into the public repository. Restore tests include contact data access/deletion retention interactions. Export projections can be rebuilt, raw verified facts and audit evidence cannot be casually discarded.

## Operating-cost and judging continuity checklist

- [ ] Create a dated budget for API/worker/PG/queue/storage/egress/model/browser/search/monitoring services.
- [ ] Enforce cloud/model/browser spend limits independently of payment spending limits.
- [ ] Confirm credit/trial end dates cover judging; sponsors may require paid renewal.
- [ ] Keep a reproducible local functional build with complete setup if hosted service is unavailable.
- [ ] Protect demo users from arbitrary amounts, open signup abuse, public admin and unrestricted task/browser access.
- [ ] Rehearse seeded reset without deleting financial/audit history; create new demo cohort/namespace instead.
- [ ] Record daily smoke/capability status without repeated financial noise.
- [ ] Freeze financial scope before submission; retain critical security fixes with clear release notes.
