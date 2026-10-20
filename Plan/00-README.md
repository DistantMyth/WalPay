# WalPay backend implementation plan

Planning baseline: 16 October 2026. Product direction: global, USD as the primary display/reporting currency, original transaction currencies preserved, explicit conversion fees. Scope: backend planning only. No application, payment, integration, or executable test suite has been built or run by creating these documents. Code and commands below are implementation specifications or illustrative sketches unless explicitly marked otherwise.

## Recommended product

WalPay is a PayPal-connected assistant for buying within a user's limits, getting paid for work, and coordinating shared expenses. PayPal remains the payment provider and holds the wallet. WalPay stores identities, permissions, obligations, business records, audit evidence, and transaction observations; it does not create a spendable WalPay wallet.

First demonstrate one coherent journey: discover a product, negotiate against a WalPay merchant policy, validate the final all-in price, obtain transaction-bound user verification plus PayPal approval, capture in the sandbox, and show a reconciled receipt. Extend with receipt splitting and a signed task-approval-to-business-payout journey. Keep an explicit distinction between a real sandbox transaction and a simulated external marketplace.

## Reading order and responsibilities

| File | Purpose |
|---|---|
| [01-product-scope-and-feasibility.md](01-product-scope-and-feasibility.md) | Requested feature coverage, capability labels, MVP and acceptance journeys |
| [02-backend-architecture-and-stack.md](02-backend-architecture-and-stack.md) | Stack by subsystem, boundaries, job execution, folder structure and architecture decisions |
| [03-domain-data-and-ledger.md](03-domain-data-and-ledger.md) | Entities, database constraints, financial observations, outbox/inbox and migrations |
| [04-paypal-payments-and-wallet.md](04-paypal-payments-and-wallet.md) | PayPal identity, orders, captures, refunds, webhooks and recovery |
| [05-agentic-commerce-and-negotiation.md](05-agentic-commerce-and-negotiation.md) | Shopping missions, negotiation, store/marketplace adapters and AI tool safety |
| [06-api-contracts-and-events.md](06-api-contracts-and-events.md) | Canonical REST conventions, route register, representative contracts and event schemas |
| [07-social-payments-receipts-and-causes.md](07-social-payments-receipts-and-causes.md) | Receipt splits, requests, contacts, QR/NFC, causes, accessibility and UPI-style coverage |
| [08-earnings-payouts-and-business.md](08-earnings-payouts-and-business.md) | Work approvals, partner integration, earnings, invoices, subscriptions and multiparty commerce |
| [09-security-identity-and-authorization.md](09-security-identity-and-authorization.md) | Passkeys/PIN semantics, threat model, privacy, permissions and release security gates |
| [10-testing-tools-and-quality-gates.md](10-testing-tools-and-quality-gates.md) | Backend test tooling, fixtures, failure tests, AI evaluations and CI gates |
| [11-development-phases-and-task-checklists.md](11-development-phases-and-task-checklists.md) | Dependency order, granular tasks, dates and phase exit criteria |
| [12-sponsor-integration-plan.md](12-sponsor-integration-plan.md) | Every listed sponsor, useful integration, evidence and fallback |
| [13-deployment-observability-and-runbooks.md](13-deployment-observability-and-runbooks.md) | Local/staging/demo/live environments, secrets, metrics, backups and incident response |
| [14-decisions-risks-and-source-register.md](14-decisions-risks-and-source-register.md) | Unresolved decisions, research evidence, ownership and revalidation gates |
| [15-global-currency-and-fx.md](15-global-currency-and-fx.md) | USD-first reporting, FX quotes, platform charges, rounding and future P2P FX |
| [16-user-browser-agent-and-marketplace-automation.md](16-user-browser-agent-and-marketplace-automation.md) | User's signed-in browser, Playwright/Hermes/Browser Use comparison, local connector, seller actions and site policy gates |

## How to use the plan

1. Freeze the conventions in 02, 03, 06 and 09 before parallel implementation.
2. Build Phase 0 and the payment vertical slice before adding agent autonomy.
3. Every feature task needs its API operation, permission, persistence, idempotency behavior, failure behavior, test ID and observability evidence.
4. A checkbox means work to do, not a claim that a control or feature exists. Check it only after a reviewer can inspect evidence.
5. Specialist documents may describe additional internal states. The public API state register in 06 is authoritative; adapters translate provider states into it without discarding the original provider status.
6. Preserve all requested features in the roadmap. A capability gate is a dependency with an owner and fallback, never an invented endpoint or a silently omitted feature.

## Non-negotiable backend properties

- Exact integer money arithmetic; no floating-point payment calculations.
- A lower buying target never forces overspending; the hard upper cap includes all known taxes, shipping and disclosed fees.
- Agents cannot approve their own proposals, change recipient identity, read secrets, or call payment providers directly.
- A successful browser return, accepted queue job, or created payout batch is not settlement evidence.
- Local biometric/PIN verification does not replace provider approval.
- Unknown provider outcome holds the reservation and enters reconciliation; it is not an excuse to submit a new payment.
- Signed, authenticated partner approval is an input to a policy check, not unconditional payout authority.
- No universal UPI/PayPal interoperability, global domestic P2P support, or peer FX execution is implied.

## Delivery checklist

- [ ] Confirm team size and assign human owners; schedule currently assumes two people plus coding agents.
- [ ] Create a sandbox capability report using the team's own developer accounts without committing credentials.
- [ ] Select actual supported demo payer/payee countries and currencies; global design does not mean all corridors are enabled.
- [ ] Implement the canonical OpenAPI document and migrations, then update this plan alongside accepted changes.
- [ ] Keep an evidence folder for sandbox operation IDs, redacted webhook traces, test reports and sponsor usage.
- [ ] Finish the working build before expanding stretch scope.
- [ ] Read the current official competition rules again before submission; see 11 and 14.

Harness recommendations are intentionally provided in the chat, outside these planning files, as requested.
