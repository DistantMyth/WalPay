# Decisions, risks, assumptions and research register

Research baseline: 16 October 2026. Links document provider capability at research time; revalidate before integration and before enabling live traffic. Proposed architecture, schemas, policies and code in this folder are original implementation decisions, not statements that PayPal or sponsors supply those exact APIs.

## Accepted decisions

| ID | Decision | Consequence / evidence location |
|---|---|---|
| ADR-001 | Global product, USD primary; original amounts preserved | User clarification; 15 defines FX and charges |
| ADR-002 | PayPal remains wallet/payment provider; no stored WalPay funds | Avoid fictitious internal spendable balances; 03/04 |
| ADR-003 | Modular TypeScript backend, PG authority, separate API/worker | Smaller invariant boundary; 02 |
| ADR-004 | Exact money strings/BigInt, immutable financial terms | Currency/rounding/property tests; 03/06/15 |
| ADR-005 | Deterministic policy gateway owns all financial actions | Agents propose; permission and budget checks outside model; 05/09 |
| ADR-006 | Provider approval plus transaction-bound local step-up | Local biometrics/PIN does not authorize PayPal itself; 04/09 |
| ADR-007 | Durable operations, inbox/outbox and reconciliation | At-least-once delivery; once-only business effects via uniqueness, not a claim of exactly-once networks |
| ADR-008 | Own-store negotiation is first live sandbox journey | eBay/Meta capabilities remain separately gated; 05/16 |
| ADR-009 | User's selected signed-in browser is a requested execution mode | Local scoped connector; browser tooling/security detailed in16 |
| ADR-010 | Full feature roadmap, narrow hackathon critical path | 01/11; no silent omission of requested capabilities |
| ADR-011 | Sponsor use needs meaningful working evidence | 12 covers all listed sponsors; frontend-only use deferred |
| ADR-012 | P2P FX requires an approved provider/settlement model | Preserve request without inventing a general PayPal exchange API;15 |

## Open decisions with owners and defaults

| Decision | Proposed owner | Default until resolved | Deadline / validation |
|---|---|---|---|
| Human team size | Team lead | Two humans plus coding agents | Before assigning Phase0 tasks |
| Demo countries/account products | Payment lead | Supported sandbox USD corridor, never false residency | Capability probe before payments |
| Lower cap meaning | Product lead | Opening/preferred buying target, not mandatory minimum spend | Mission schema freeze |
| Executable FX provider and fee model | Payment/product lead | Reporting estimates; provider-managed eligible checkout only | Before any conversion claim |
| Browser framework/version and local connector packaging | Agent lead | Playwright MCP extension with restricted gateway,16 | Browser spike and platform tests |
| Facebook/eBay messaging permissions | Integration lead | Read/draft/manual handoff; external sends disabled without support | Before site write scope |
| Cloud/local receipt processing | Privacy/agent lead | User-consented cloud or isolated local OCR; review required | Receipt phase |
| Actual AI model used in product | Agent lead | One schema-capable provider adapter, evaluated | Before mission integration |
| Managed PG version and queue compatibility | Backend lead | PG18 preferred, tested17 fallback via ADR | Phase0 deployment probe |
| Public demo access and funded availability | Team lead | Controlled test users, paid capacity if needed | Before hosted demo |
| Cause beneficiary onboarding/fee treatment | Business lead | Verified supported merchant/charity sandbox only | Cause feature gate |

## Risk register

| ID / severity | Risk | Trigger / mitigation / fallback |
|---|---|---|
| R01 critical | API access assumed from hackathon examples | Capability matrix and account probe; own-store supported checkout fallback |
| R02 critical | Browser content triggers unauthorised financial/message action | Scoped tools, proposal digest and explicit approval; pause/handoff |
| R03 critical | Timeout/double payment | Stable operation keys, reservation retention and provider reconciliation |
| R04 critical | Global/P2P/FX path violates provider eligibility | Corridor allowlist and approved product gate; tracking/estimate only |
| R05 high | Broad scope prevents working submission | Phase11 core gate and timeboxed stretch work |
| R06 high | Provider settlement/payout pending shown as earned cash | Distinct gross/pending/confirmed/item/withdrawal states |
| R07 high | FX or fee increase exceeds cap | Exact payable currency/quote, all-in digest, requote and reapprove |
| R08 high | Contact/receipt/browser data leaks | Minimize, encrypt, local scoped session, redaction/retention tests |
| R09 high | Partner replay or malicious approval | Signatures, task version, source GET verification where possible and unique obligation |
| R10 high | One-click browser action is binding offer | Site-specific action classification, spend reservation and user approval |
| R11 medium | Sponsor access/trial unavailable by judging | Track entitlement expiry; local equivalent; no central financial dependency on optional trial |
| R12 medium | OCR line/tax inference wrong | Confidence and correction draft; confirmation before requests |
| R13 medium | Accessibility claimed without actual client testing | Backend locale/summary contracts now; assistive-device acceptance when clients exist |
| R14 high | Restore starts duplicate external operations | Outbound-off restore, provider reconciliation and unique-resource evidence |

## Primary-source register

The register is a navigation/evidence index, not copied provider documentation. Specialist files cite narrower pages for individual claims.

| Area | Official source | What to recheck |
|---|---|---|
| Competition | [Rules](https://paypalaihackathon.devpost.com/rules), [overview](https://paypalaihackathon.devpost.com/) | Deadline, eligibility, functional submission and license/video requirements |
| Examples/sponsors | [Resources](https://paypalaihackathon.devpost.com/resources) | Current examples, sponsor tools and access paths |
| PayPal AI | [AI docs](https://docs.paypal.ai/), [toolkit source](https://github.com/paypal/agent-toolkit) | Actual tool availability, permissions, sandbox configuration |
| Orders | [Create](https://developer.paypal.com/api/orders/v2/orders-create), [capture](https://developer.paypal.com/api/orders/v2/orders-capture) | Amount/payee/approval/capture states and endpoint constraints |
| Refunds | [Capture refund](https://developer.paypal.com/api/payments/v2/captures-refund) | Partial refunds, pending results and idempotency |
| Idempotency | [REST guidance](https://developer.paypal.com/api/rest/reference/idempotency/) | Product-specific key retention, concurrent duplicate behavior |
| Webhooks | [REST webhooks](https://developer.paypal.com/api/rest/webhooks/rest/), [signature verification](https://developer.paypal.com/api/webhooks/v1/verify-webhook-signature-post) | Configured app/binding, raw body, self vs postback verification, simulator limits |
| Business payouts | [Create payout](https://developer.paypal.com/api/payments.payouts-batch/v1/payouts-post), [batch header](https://developer.paypal.com/api/payments.payouts-batch/v1/definitions/payout_sender_batch_header/) | Account eligibility, funding, receiver types, batch/item states and duplicate window |
| Platforms | [Overview](https://developer.paypal.com/platforms/overview), [seller onboarding](https://developer.paypal.com/platforms/seller-onboarding/before-payment/) | Approval, seller credentials, platform fee currency/account requirements |
| Currencies | [Currency codes](https://developer.paypal.com/api/codes/currency) | Product/country support, exponents, receiving preferences |
| Provider terms | [US agreement](https://www.paypal.com/us/legalhub/paypal/useragreement-full) | Applies to US accounts only; inspect actual user's country/product terms |
| Passkeys | [WebAuthn specification](https://www.w3.org/TR/webauthn-3/), [SimpleWebAuthn](https://simplewebauthn.dev/docs/packages/server) | Server verification, origin/RP/UV, supported library version |
| HTTP errors | [RFC9457](https://www.rfc-editor.org/rfc/rfc9457.html) | Problem details format and safe error disclosure |
| HTTP schema | [Fastify validation](https://fastify.dev/docs/latest/Reference/Validation-and-Serialization/) | Version-specific schema/serialization behavior |
| Database | [PG isolation](https://www.postgresql.org/docs/current/transaction-iso.html), [row security](https://www.postgresql.org/docs/18/ddl-rowsecurity.html) | Lock/retry behavior, owners/BYPASSRLS, migration version |
| Jobs | [BullMQ idempotent jobs](https://docs.bullmq.io/patterns/idempotent-jobs) | External side-effect safety beyond queue retry |
| Deployment | [Render Key Value](https://render.com/docs/key-value) | Valkey compatibility, persistence, network/authentication and eviction |
| Browser tooling | Sources in [16](16-user-browser-agent-and-marketplace-automation.md) | Existing-tab attachment versus cloud sessions, actual scopes and site permissions |
| Sponsors/testing | Sources in [10](10-testing-tools-and-quality-gates.md) and [12](12-sponsor-integration-plan.md) | Tool versions, licenses, quotas, current access and reproducibility |

## Capability evidence record to implement

For every external product store `{provider, product, operation, mode, account_binding, country_pair, currency, entitlement_source, documentation_url, checked_at, checked_by, probe_result, redacted_evidence_ref, expires_at, limitations}`. A documentation page proves a product exists; a successful account probe proves only that tested operation/configuration, not all global corridors or live permissions. A denied probe is useful evidence and must produce a clear fallback rather than a fabricated success.

## Pre-development decision checklist

- [ ] Human owners accept defaults or replace them through recorded ADRs.
- [ ] Every G/U feature has a clear backend unavailable response and user-facing handoff contract.
- [ ] All source-dependent assumptions are rechecked during account integration.
- [ ] Document exact simulator/sandbox/live labels in evidence and API responses.
- [ ] Record team/model/tool/service versions when work starts.
- [ ] Review plan changes for API/data/test/phase implications together.
