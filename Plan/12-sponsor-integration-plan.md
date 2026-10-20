# Sponsor integration plan

Research baseline: 16 October 2026. This file distinguishes planned adoption from an implemented integration; **none of the integrations below has been implemented or tested by writing this plan**. Third-party capability, quota and license details must be checked again when enabled.

## 1. Exact sponsor set and adoption order

The official resource page currently lists ten sponsor tools: **AG Grid, APIMatic, Astropods, Bryntum, Channel3, Elastic, KERNEL, Postman, Render, and Zapier**. [Hackathon sponsor resources](https://paypalaihackathon.devpost.com/resources)

Use as many as serve a demonstrated user/backend outcome without weakening the central PayPal flow. All ten are considered here; eight have a possible backend/development role. AG Grid and Bryntum are primarily frontend component products, so the backend phase prepares secure read contracts but does not count unbuilt widgets as completed sponsor usage.

| Priority | Sponsor | Concrete role in WalPay | Initial adoption decision | Financial authority |
|---|---|---|---|---|
| P0 | Postman | API design/test collection, partner examples, CI smoke evidence | Core | No direct money authority; calls the normal guarded API |
| P0 | APIMatic | Verified PayPal API context for coding agents; later contract-derived SDK/docs | Core development tool if harness-compatible | Context/generated code does not authorize transactions |
| P0 | Render | Fastify API, BullMQ workers, private PostgreSQL and Redis-compatible queue hosting | Core deployment target | Runs guarded backend, never becomes source of payment truth |
| P1 | Channel3 | Real product discovery/enrichment with normalized offers | First runtime sponsor addition after core capture | Discovery only unless separately approved checkout capability is proven |
| P1 | Elastic | Tenant-scoped product/earnings search and grounded read-only explanations | Useful stretch | Read projections only |
| P2 | Zapier | One consented partner task workflow or post-payment record sync | Conditional stretch | Partner event ingress still passes backend verification/policy |
| P2 | Astropods | Hosted, observed proposal agent with tightly scoped tools | Conditional stretch; avoid duplicating the full worker stack | Proposals only; backend remains action authority |
| P2 | KERNEL | Separately consented cloud browser for authorized own-store workflow/replay evidence; selected existing local browser uses its own connector | Conditional stretch | Cannot inherit a local tab grant or bypass PayPal payer approval/backend policy |
| P3 | AG Grid / AG Studio | Future earnings/donation dashboard backed by scoped analytics data | Frontend deferred | Read-only backend data contract |
| P3 | Bryntum | Future task/milestone/payment schedule for freelancer workflows | Frontend deferred | Scheduling UI cannot self-approve tasks or payouts |

Sponsor resource participation is not the same as a separate cash-prize category, and a package import is not meaningful integration evidence. Check the current category/eligibility requirements before selecting a sponsor prize; the rules limit the combinations that can be won. [Official prize rules](https://paypalaihackathon.devpost.com/rules)

## 2. Integration boundary shared by every sponsor

```text
User/partner intent
  -> authenticated WalPay request + verified consent
  -> domain policy and capability checks
  -> durable intent/operation in PostgreSQL
  -> outbox and controlled worker
  -> permitted external tool/provider
  -> typed validated result + evidence
  -> verified business state and scoped projections
```

External tools receive the smallest useful payload. They do not receive PayPal client secrets, raw local PINs, biometric data, unrestricted service credentials, or unneeded phone/address/receipt details. A tool result is untrusted input. Authentication, ownership, exact amount/currency, fee/FX quote, budget, expiry, recipient and approval are checked again by deterministic backend code at dispatch.

Each adapter must expose:

```ts
type IntegrationCapability = {
  name: string;
  enabled: boolean;
  mode: 'simulation' | 'sandbox' | 'live' | null;
  operations: readonly string[];
  supportedCurrencies: readonly string[];
  supportedCountries: readonly string[];
  maxConcurrency: number;
  timeoutMs: number;
  evidenceVersion: string;
};
```

This is an internal capability record, not a claim that every sponsor defines these fields. SDK-specific data maps into WalPay-owned types. `enabled: false` uses `mode: null`; enabled runtime modes follow the canonical `simulation | sandbox | live` vocabulary. Avoid calling a read-only analytics/search tool a `live` payment integration merely because its service is online; integration evidence records which operation actually ran.

- [ ] Separate dev/sandbox/live credentials and callback URLs where the product supports them.
- [ ] Add timeouts, bounded retries, concurrency/rate/cost ceilings and circuit behavior.
- [ ] Store secret references rather than secret values in database configuration/audit.
- [ ] Validate response schema and restrict returned URLs before fetching/navigating.
- [ ] Add feature-off and dependency-unavailable responses.
- [ ] Add data retention/deletion/export implications to the integration register.
- [ ] Log sanitized request correlation, operation, duration, result category and usage units.
- [ ] Add the provider fake scenario, contract test and a small real-service smoke test.
- [ ] Record actual usage, version, commit, timestamp, evidence and limitations before making a public integration claim.

## 3. Postman — shared API workflows and review evidence

Postman supplies API collaboration/testing capabilities and the event links to PayPal's collection. Newman runs collections from the command line. [Sponsor resource](https://paypalaihackathon.devpost.com/details/postman), [Newman documentation](https://learning.postman.com/docs/reference/newman-cli/command-line-integration-with-newman/)

**WalPay design:** maintain a sanitized WalPay collection derived from the versioned OpenAPI contract, augmented with authored business journeys. Keep official PayPal examples in a separate learning collection so direct provider calls cannot be mistaken for WalPay policy enforcement.

Collection folders:

1. Environment/health and authenticated session setup using dedicated synthetic users.
2. Shopping mission, offer, final immutable quote and policy rejections.
3. Intent, WebAuthn authorization boundary, order, capture and retrieval. Browser PayPal approval remains an explicit external step.
4. Receipt review, split allocation, request/accept/decline/expiry.
5. Cause QR resolution, contribution and progress.
6. Conditional partner/invoice/payout examples with capability prechecks.
7. Negative ownership, mismatched idempotency, expired quote and provider pending/unknown state.

- [ ] Check status, response schema, stable error code, IDs, currency/amount strings and duplicate-effect assertions.
- [ ] Chain only IDs returned by server; do not supply trusted totals from client variables.
- [ ] Separate local fake-provider and real sandbox environments.
- [ ] Ignore secret environment exports and scrub CI request/response reports.
- [ ] Add Newman CI smoke and readable judge instructions.
- [ ] Record a successful run manifest for actual release commit; a collection file alone is insufficient evidence.

## 4. APIMatic — API context and generated interfaces

The sponsor offers a PayPal Context Plugin intended to provide relevant API context to coding agents. APIMatic also produces SDK/documentation outputs from API descriptions. Harness compatibility needs to be verified rather than assumed. [PayPal Context Plugin resource](https://paypalaihackathon.devpost.com/details/apimatic), [APIMatic platform](https://www.apimatic.io/)

**WalPay design:** use the plugin while implementing the PayPal adapter, compare generated request/response handling with the official PayPal reference and actual sandbox behavior, and retain a short integration note identifying the context source/version. In a later step, feed WalPay's public OpenAPI into SDK/docs generation for a partner-facing client; keep generated output separate from handwritten domain policy.

- [ ] Inspect installation/configuration instructions and compatibility with the actual Oh My Pi setup.
- [ ] Use documented standard/plugin interfaces only; do not claim drop-in compatibility without a successful test.
- [ ] If unavailable in the harness, use approved reference context through a compatible development tool and state that path accurately.
- [ ] Generate/review typed adapter calls for only the enabled PayPal products.
- [ ] Keep idempotency, timeout classification, authorization and reconciliation in reviewed WalPay code.
- [ ] Pin generated SDK/plugin version and document regeneration/diff process.
- [ ] Validate generated contract examples with simulator and real sandbox tests.
- [ ] Record actual tool usage and resulting reviewed integration file/commit, without publishing private prompts/keys.

**Scope boundary:** a context plugin is a development integration; it is not a runtime payment gateway or independent reason to let an agent spend.

## 5. Render — deployment with one durable execution model

Render supports long-lived background workers, managed PostgreSQL and Redis-compatible Key Value. New Key Value instances currently use Valkey 8; paid instances can persist data, and queues should use a `noeviction` policy. Check the exact instance configuration and BullMQ compatibility. [Background workers](https://render.com/docs/background-workers), [PostgreSQL](https://render.com/docs/postgresql), [Key Value](https://render.com/docs/key-value)

**WalPay design:** deploy a stateless Fastify web service, a separately scalable BullMQ worker service, private PostgreSQL and an authenticated Redis-compatible Key Value instance in a compatible same-region network. Use the existing PostgreSQL outbox/business operation records for recovery; queue persistence alone is insufficient.

- [ ] Add reproducible infrastructure configuration with secret references.
- [ ] Use separate demo/staging environments and explicit sandbox provider configuration.
- [ ] Provision a continuously available plan appropriate to webhook delivery and judging access; measure cold starts if a sleeping plan is considered.
- [ ] Configure SQL pooling, queue connection limits, `noeviction`, persistence and memory alerts.
- [ ] Restrict public SQL/Key Value access; enable internal authentication where supported.
- [ ] Set health/readiness routes and safe shutdown/deploy behavior.
- [ ] Migrate through one controlled deployment step; don't let every replica race migrations.
- [ ] Test worker/API restart, queue reconnection and webhook redelivery.
- [ ] Test actual backup/restore options of the selected plan; do not promise unpurchased HA/PITR.
- [ ] Track cost and credit balance through the full judging period.

**Optional Render Workflows usage:** use one read-only report/retail-enrichment task if useful for sponsor demonstration. Workflows support TypeScript/Python tasks with managed execution/retries, but do not add a second executor for the same money effect. [Render Workflows](https://render.com/docs/workflows)

If adopting Workflows for the core later, replace the execution adapter deliberately, preserve operation IDs/outbox/reconciliation and rerun race/fault tests. Do not simultaneously dispatch the same capture/payout from BullMQ and Workflows.

## 6. Channel3 — shopping discovery with truthful checkout capability

Channel3 offers structured product search/enrichment across retailers and a developer API/MCP path. The sponsor's eligibility text requires meaningful API or MCP use rather than name-only reference. [Sponsor resource](https://paypalaihackathon.devpost.com/details/channel3), [official search guide](https://docs.trychannel3.com/guides/make-a-search)

**WalPay design:** add a discovery adapter that queries by user constraints and returns normalized offers. WalPay's own-store adapter remains the guaranteed negotiation/PayPal settlement demonstration. A Channel3 retailer result, buy URL, or product price does not by itself establish a right to negotiate with that retailer, complete its checkout, or charge through PayPal.

Proposed normalized result, independent of Channel3's native schema:

```json
{
  "source": "channel3",
  "source_product_id": "provider-product-id",
  "title": "Example headphones",
  "listed_price": { "currency": "USD", "amount_minor": "8900" },
  "observed_at": "2026-10-16T15:00:00Z",
  "fees_complete": false,
  "shipping_known": false,
  "availability": "reported_in_stock",
  "checkout_capability": "external_link_only",
  "negotiation_capability": "unsupported",
  "provider_offer_reference": "opaque-reference",
  "expires_at": "2026-10-16T15:05:00Z"
}
```

Only a separately verified partner capability can change `checkout_capability`/`negotiation_capability`. Requote price, stock, taxes, shipping and currency before creating an intent; no stale search price becomes an authorized total. The provider currently has a Checkout documentation route, but the fetched page did not establish an executable PayPal integration contract; leave that capability unverified until documented access is obtained. [Channel3 docs](https://docs.trychannel3.com/)

- [ ] Obtain key and perform a real product search; store scrubbed evidence.
- [ ] Map source identifiers/prices/currency/availability/URLs safely.
- [ ] Enforce currency allowlist and locale mapping; USD display does not convert settlement silently.
- [ ] Add bounded cache TTL with provenance and observed timestamp.
- [ ] Test empty results, stale stock, missing fees, unsupported currency, 429/timeouts and malformed data.
- [ ] Prevent product descriptions or returned links from selecting arbitrary tools/payees.
- [ ] Keep official affiliate/commission terms and user disclosures where relevant.
- [ ] Demonstrate an actual search/enrichment result affecting the mission shortlist.

## 7. Elastic — useful scoped retrieval and analytics

Elasticsearch supports lexical, vector/hybrid retrieval and aggregations. The sponsor also lists agent/retrieval/observability capabilities. [Elastic search documentation](https://www.elastic.co/docs/solutions/search), [sponsor resource](https://paypalaihackathon.devpost.com/details/elastic)

**WalPay design:** choose one bounded use case first: "show what I earned from each client this month" grounded in minimized verified payment projections, or product policy/catalog retrieval. PostgreSQL remains financial truth. Ingest through an outbox consumer using idempotent document IDs; search lag cannot change payment status.

Document projection fields: tenant ID, authorized subject ID, business resource ID, kind, merchant/client label when consented, original currency and integer minor string, verified state, event time, source evidence reference, projection version. For analytics that require numeric aggregation, validate a safe integer representation/range or aggregate authoritative decimal/integer values in SQL; do not truncate large money values through JavaScript floating point. USD reporting fields include rate source/time and estimate flag.

- [ ] Define minimized index mapping and retention; exclude tokens, raw receipts and unnecessary contact PII.
- [ ] Enforce tenant/user filters server-side on every query; do not accept arbitrary agent-generated DSL.
- [ ] Provide allowlisted query operations with bounded date window/result count.
- [ ] Add real index write/search smoke and a deterministic relevance dataset.
- [ ] Rebuild index from SQL/event source and test duplicate/out-of-order projection events.
- [ ] Test cross-tenant retrieval, deleted/revoked data, stale result and unavailable cluster.
- [ ] Ground AI explanations in returned evidence and show missing/stale data honestly.
- [ ] Keep capture/payout/reconciliation operational if Elastic is unavailable.

## 8. Zapier — one authenticated partner workflow

Zapier MCP exposes connected app actions and handles app connections. Current docs require Streamable HTTP for MCP; a connection token is a powerful credential and must remain private. [MCP overview](https://docs.zapier.com/mcp/home), [quickstart](https://docs.zapier.com/mcp/get-started/quickstart)

**WalPay design:** select one consented test partner, such as a task board, and connect a reviewed task-approval workflow. If using Zapier MCP for read/propose actions, constrain it to the selected partner/app tools. If using a webhook integration to deliver task completion, configure a trusted bridge credential/signature into WalPay's normal `/v1/partner-events/{partnerId}` path. These are separate integration modes; document which actually ran.

An "approved" string in a sheet, chat message or arbitrary MCP return is not sufficient to authorize money. Validate authenticated partner, event ID, task revision, authorized approver, preagreed amount/currency/payee, funding/capability and payout policy. The backend records the operation and uses its single PayPal adapter. Do not enable direct PayPal money actions in a generic Zapier agent that bypasses this path.

- [ ] Connect only a team-owned/consented test workspace and narrowly scoped app actions.
- [ ] Record actual action names and auth/credential model supported by the selected connector.
- [ ] Store Zapier connection credentials in secret storage; never embed token URLs in public docs.
- [ ] Validate signed/authenticated ingress, idempotency and replay rules.
- [ ] Exercise duplicate, spoofed, outdated-revision, edited-amount and revoked-partner events.
- [ ] Handle Zapier rate limits, task quota and outages without false payout success.
- [ ] Prefer a safe record-sync or approved task demo if PayPal payout access is unavailable.
- [ ] Any outbound notifications/updates require the corresponding user/partner consent and are deduplicated through outbox.

## 9. Astropods — isolated hosted proposal agent

Astropods provides a project/spec, versioned blueprint and hosted agent lifecycle with models/tools/observability. [Official overview](https://docs.astropods.com/welcome), [hackathon resource](https://paypalaihackathon.devpost.com/details/astropods)

**WalPay design:** deploy one agent that reads a redacted mission/catalog shortlist and proposes a typed negotiation/search plan. Give it only scoped read/propose tools. The Fastify domain service applies the plan, enforces bounds, handles human approval, and persists all financial state in PostgreSQL. Avoid distributing PayPal credentials across multiple runtimes.

- [ ] Verify CLI/runtime and model API compatibility with the chosen model/provider access.
- [ ] Publish a versioned spec/blueprint and pin the deployed version in the integration manifest.
- [ ] Implement bounded request/response schema with correlation ID and policy reference.
- [ ] Add short-lived scoped backend tool credentials where supported.
- [ ] Limit model calls/time/tokens and store redacted action summaries.
- [ ] Run the same E-AI adversarial dataset against local and hosted adapters.
- [ ] Test agent unavailability, duplicated response and expired mission.
- [ ] Demonstrate a real hosted run and trace; retain local adapter fallback for reproducibility.

Astropods is an optional runtime for a bounded agent, not an additional master of business state. If adoption adds deployment/debugging risk before Core A is green, defer it.

## 10. KERNEL — controlled browser actions for authorized surfaces

KERNEL supplies isolated browser infrastructure with automation and session observation/replay capabilities. [Official documentation](https://www.kernel.sh/docs), [sponsor resource](https://paypalaihackathon.devpost.com/details/kernel)

**Local versus cloud:** the requested selected already-signed-in local Chrome/Edge tab is handled by the paired local companion and verified Playwright MCP extension described in [16](16-user-browser-agent-and-marketplace-automation.md). KERNEL is a separate cloud browser lane, not an automatic attachment to that desktop tab. Kernel profiles persist cloud browser state; they do not prove control of a user's existing local profile. [Kernel profiles](https://www.kernel.sh/docs/browsers/profiles)

**WalPay design:** use a separately consented short-lived cloud session to browse the team's own store fixture, inspect item details, add an item or obtain a permitted nonbinding offer, then return a structured evidence reference. The core also demonstrates actual browser actions through the local connector. A cloud session can be useful for consenting partner test automation/replays, while local signed-in-session access retains its own consent, account/profile/tab and grant boundaries.

- [ ] Allowlist own-store/test domains and permitted navigation/actions.
- [ ] Ask for/record a separate cloud-session grant with account/site/data retention scope; local browser consent never silently authorizes cloud login/state transfer.
- [ ] Never copy local cookies, profile directories, tokens or credentials into Kernel to simulate local tab attachment.
- [ ] Give the session a test user identity scoped to that store/task, without PayPal password/OTP/PIN access.
- [ ] Apply the same typed observation/action/digest/recipient/cap/site-policy checks as the local connector; no unrestricted raw CDP/eval tools for the model.
- [ ] Set action count, deadline and cost limits; stop and return to the user for supported approval steps.
- [ ] Filter screenshots/replays and retention to avoid exposing sensitive data.
- [ ] Treat web page instructions as untrusted content, not tool-policy instructions.
- [ ] Revalidate all observed price/payee/currency on the server.
- [ ] Test changed DOM, stale price, out-of-stock item, redirect, navigation outside allowlist and canceled mission.
- [ ] Test BRO-026: local disconnect or Kernel outage cannot cause a silent fallback across local/cloud lanes; all affected approvals are invalidated/revalidated appropriately.
- [ ] Preserve sanitized replay evidence showing the actual authorized action.
- [ ] Do not turn browser automation into a claim of authorized eBay/Facebook Marketplace messaging, checkout or payout access.

The existence of browser infrastructure or anti-detection features does not supply marketplace permission. External automation stays disabled until the platform's supported interface and authorization are established.

## 11. AG Grid / AG Studio — future dashboard boundary

The event resource emphasizes AG Studio dashboards and its agent framework; AG Grid/Charts alone do not demonstrate all of that suggested experience. AG Studio is a frontend library. [Sponsor resource](https://paypalaihackathon.devpost.com/details/aggrid), [AG Studio quickstart](https://www.ag-grid.com/studio/react/quick-start/)

**Backend work now:** define scoped earnings/cause analytics responses with original amount/currency, estimated USD value/rate metadata, captured/refunded/net totals, date grouping, pagination and freshness. Accept only allowlisted filters and sort fields; cap exports. An analytics agent can select among named read queries, not run arbitrary SQL or payment tools.

- [ ] Document future analytics contract in the canonical API plan before adding routes.
- [ ] Test ownership, date ranges, currency grouping, refunds, stale projection and large exports.
- [ ] Store dashboard preferences separately from financial facts if later required.
- [ ] Verify eventual component trial/license and redistribution terms before frontend work.
- [ ] Mark sponsor usage **deferred** until actual component/agent integration exists.

## 12. Bryntum — future freelancer scheduling boundary

Bryntum's Gantt/Scheduler/Calendar components fit time-based task/milestone experiences; they are frontend components rather than a payment execution backend. [Sponsor resource](https://paypalaihackathon.devpost.com/details/bryntum), [Gantt documentation](https://bryntum.com/products/gantt/docs/)

**Backend work now:** model milestone IDs, dependencies, scheduled/approved/completed times, resource assignments and linked invoice/payout references in the domain backlog. A future schedule mutation has optimistic revision, tenant authorization and validation for dependency cycles. Moving a milestone in a timeline does not approve a financial task or trigger an unchecked payout.

- [ ] Define UTC instants versus local calendar dates/timezone fields explicitly.
- [ ] Test dependency cycles, conflicting revisions and cross-tenant resource references.
- [ ] Separate scheduled due date from authenticated partner approval evidence.
- [ ] Link only read-only financial status into future schedule responses.
- [ ] Verify eventual trial/license/redistribution terms before frontend work.
- [ ] Mark sponsor usage **deferred** until an actual component is included.

## 13. Credits, trials and judging availability

The following are offers described on the current event resource pages, not balances already granted to the team. Claim conditions, region/account restrictions and future changes apply; record actual acceptance privately.

| Sponsor | Current event resource offer | Planning implication |
|---|---|---|
| AG Grid | 45-day trial | Starting 16 October does not reach 15 December judging end; arrange proper licensing or defer |
| APIMatic | Free subscription paths tied to actual Context Plugin usage/winning, via linked claim form | Do not budget business-tier access as already granted |
| Astropods | Free account path | Check actual model/runtime quota, not an assumed unlimited hosted agent |
| Bryntum | 45-day trial | Same judging-period license issue as other 45-day trials |
| Channel3 | Hackathon key credits and promo offer on its sponsor page | Track search credits and terms; don't assume unlimited searches/checkout rights |
| Elastic | 14-day trial with described extension route, plus local deployment option | Check actual granted extension/cost and protect retained indexes |
| KERNEL | Event link offers $50 credits | Bound browser sessions/actions; cost cannot grow with an agent loop |
| Postman | Free start path | Verify collection collaboration/CI/reporting limits actually needed |
| Render | Event link offers $50 credits | Price the web service, worker, SQL, Key Value and judging uptime separately |
| Zapier | 14-day Professional trial, then Free plan described | Check MCP task/connector features after trial; keep core independent |

Offer sources: [AG Grid](https://paypalaihackathon.devpost.com/details/aggrid), [APIMatic](https://paypalaihackathon.devpost.com/details/apimatic), [Astropods](https://paypalaihackathon.devpost.com/details/astropods), [Bryntum](https://paypalaihackathon.devpost.com/details/bryntum), [Channel3](https://paypalaihackathon.devpost.com/details/channel3), [Elastic](https://paypalaihackathon.devpost.com/details/elastic), [KERNEL](https://paypalaihackathon.devpost.com/details/kernel), [Postman](https://paypalaihackathon.devpost.com/details/postman), [Render](https://paypalaihackathon.devpost.com/details/render), [Zapier](https://paypalaihackathon.devpost.com/details/zapier).

## 14. Integration evidence and acceptance checklist

For each enabled sponsor, store a small sanitized `integration-evidence` entry, not a secret-filled trace:

```json
{
  "sponsor": "channel3",
  "status": "planned",
  "operation": "product_search",
  "mode": null,
  "commit": null,
  "testedAt": null,
  "testIds": [],
  "evidenceReferences": [],
  "knownLimitations": ["No execution has occurred while writing the plan"],
  "owner": "unassigned"
}
```

Change status only when evidence exists: `planned → configured → contract-tested → actual-service-tested → demo-enabled`. Development-only context tools use that accurate role instead of pretending to be runtime services. Track frontend-deferred separately.

- [ ] Integration solves a real step in the selected demo story.
- [ ] Actual API/tool/component use is visible in evidence and attributed accurately.
- [ ] There is a minimal reviewed contract and fake response for deterministic tests.
- [ ] Request/response validation, rate/time/cost limit and unavailable behavior exist.
- [ ] Credential scopes, data minimization and deletion/retention are documented.
- [ ] Read-only integration cannot select payees, approve payment, execute arbitrary SQL or bypass domain policy.
- [ ] Duplicate callbacks/jobs cannot duplicate financial or notification effects.
- [ ] All source/destination currencies, fees and estimates remain explicit; no sponsor is implied to provide unverified FX/P2P rails.
- [ ] Real sandbox payment evidence remains central to the demonstration.
- [ ] Trial, license and credits support the judging period or the disabled/fallback state is honestly disclosed.
- [ ] Sponsor-prize language matches current official criteria and actual integration.

Recommended sequence: complete Postman/APIMatic/Render around the core, then Channel3 discovery, then one or two of Elastic/Zapier/Astropods/KERNEL if the quality gates and calendar allow. Prepare AG Studio/Bryntum API contracts now and integrate their components when frontend work begins. Eight possible backend/development integrations and two deferred frontend integrations is a considered plan, not a completed integration count.
