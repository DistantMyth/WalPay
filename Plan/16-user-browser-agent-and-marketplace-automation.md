# User-browser shopping agent and marketplace automation — backend plan

Planning/research date: 2026-10-16. This extends the shopping mission design in [05-agentic-commerce-and-negotiation.md](05-agentic-commerce-and-negotiation.md). It specifies an **actual browser-capable workflow** in the user's selected browser, rather than restricting product discovery to available APIs. This is a plan only: no browser has been attached, no profile read, no seller contacted, and no purchase executed.

## 1. Recommended existing tools and exact implementation boundary

Use **official Playwright MCP extension mode as the first browser driver**, behind a **local TypeScript WalPay companion/policy gateway**. It fits the existing TypeScript backend and can work with the user's selected Chrome/Edge session. Keep the existing WalPay shopping mission and payment services as the authorities. The cloud agent proposes narrow actions; the locally paired companion decides whether those actions are permitted and invokes the existing driver. Do not build a new browser engine or expose the complete browser MCP toolset directly to the model.

This recommendation is an engineering choice for the stated requirement, not a claim of comparative benchmark superiority. Browser drivers can change without changing shopping/payment policy.

| Tool | Verified documented capability | WalPay role / limitation |
| --- | --- | --- |
| Playwright MCP extension | Attaches to existing Chrome/Edge tabs and uses existing profile sessions | Primary local driver; prove exact profile/tab attachment in B0 before enabling it |
| Playwright `connectOverCDP` | Attaches to a reachable Chromium CDP browser; documented lower fidelity than native Playwright connection | Controlled testing/fallback only; no public raw debug endpoint or default-profile copying |
| Browser Use | Agent/browser orchestration, system-Chrome profile support and CDP configurations | Optional Python worker when autonomous exploration warrants it; not needed for initial deterministic adapter and not an independent security boundary |
| Hermes Agent | Browser tools, local/cloud backends and explicit local CDP attach | Optional harness for a self-hosted user; use the WalPay tool gateway through MCP, not unrestricted native browser/terminal tools for a payment mission |
| Kernel | Separate cloud browser infrastructure, profiles, authentication, live view and replay | Sponsor lane for consenting-store automation/testing; not presumed control of a user's already-open local Chrome tab |

Official evidence checked 2026-10-16:

- [Playwright browser connections](https://playwright.dev/mcp/configuration/browser-extension): extension mode and existing browser/session support. The unpinned documentation example is not a deployment lockfile; pin a tested package release and extension compatibility in implementation.
- [Playwright BrowserType](https://playwright.dev/docs/api/class-browsertype#browser-type-connect-over-cdp): Chromium CDP attachment and its fidelity limitation.
- [Browser Use real-browser guide](https://docs.browser-use.com/open-source/customize/browser/real-browser): system-Chrome profile selection; examples may require Chrome to close. A profile launch/reuse is distinct from selecting an arbitrary currently open tab.
- [Hermes official browser guide](https://github.com/NousResearch/hermes-agent/blob/main/website/docs/user-guide/features/browser.md): explicit connection to local Chromium-family browsers exists; ordinary local mode uses a packaged browser. Its real-profile option can copy profile state, which is excluded from this design. `/browser connect` is an interactive CLI feature, so it is not an embeddable website connector by itself.
- [Kernel concepts](https://www.kernel.sh/docs/), [profiles](https://www.kernel.sh/docs/browsers/profiles), [Managed Auth](https://www.kernel.sh/docs/auth/managed-auth), [live view](https://www.kernel.sh/docs/browsers/live-view), [replays](https://www.kernel.sh/docs/browsers/replays): these support a separately consented cloud-session lane. Loading an extension inside a cloud browser is not the same as attaching to the user's desktop tab.
- [Chrome remote debugging changes](https://developer.chrome.com/blog/remote-debugging-port): default-profile debugging flags changed in Chrome 136 after cookie-theft concerns; the plan does not instruct users to defeat those protections.

**MVP:** existing Playwright extension + signed local helper + typed WalPay action gateway. **Hardened V1 option:** a small custom Manifest V3 consent extension with native messaging for explicit per-tab local grants, while retaining existing Playwright/browser primitives where compatible. This custom extension is proposed WalPay work, not an undocumented Playwright or Kernel feature. Do not imply upstream extension permissions themselves enforce WalPay's tenant, spend or seller-message policy.

If Playwright extension mode cannot prove the required profile/tab isolation on supported versions, fail B0. Provide a dedicated visible browser profile with manual user login as an explicit alternative or defer attachment. Do not silently switch to a copied profile, all-tabs automation or a cloud login session.

## 2. User and platform permissions are separate gates

User-started access to their browser supplies account-holder consent. It does not automatically grant a site's permission for automated collection, messaging, binding offers or checkout. Maintain a versioned site-policy registry and require **both** site capability and user grant for each action. Site access is a product capability gate with evidence, not a prompt asking the model to interpret terms every time.

| Site/integration | Default classification | Working demonstration / enablement |
| --- | --- | --- |
| WalPay own store | `SUPPORTED_SANDBOX` | Real browser discovery, cart actions, seller-chat test workflow, final quote and PayPal sandbox handoff |
| Consenting partner/test marketplace controlled by team | `SUPPORTED_TEST` | Actual browser sends to test seller account, deterministic negotiation/cap checks, trace evidence |
| eBay product discovery | `APPROVAL_GATED` | Prefer entitled Browse API; browser mode only after applicable access/policy review |
| eBay Best Offer/bid/checkout clicks | `BINDING_ACTION_GATED` | Separate explicit user authorization and proven site permission/terms; disabled in MVP |
| Facebook Marketplace read/search/messages | `AUTOMATION_PERMISSION_UNVERIFIED` | Browser adapter architecture present, live automation disabled until allowed; human browsing + user-supplied listing/text can support drafting |
| Other stores | `SITE_REVIEW_REQUIRED` | Exact origin, permitted workflow, accounts/countries, rate limits and checkout constraints recorded |
| Kernel cloud session | `CLOUD_CONSENT_REQUIRED` | Separate session/location/profile grant, scoped API key, approved site; no migration of local sessions by default |

Meta states that automated collection from Facebook requires its permission. A signed-in browser and a human-like browser tool do not resolve that requirement. The initial Meta policy gate therefore blocks automated page observation as well as sends; a user can manually view the site and supply content for a local draft. Enable the live adapter only after a documented allowed path is established. [Meta official explanation](https://about.fb.com/news/2021/04/how-we-combat-scraping/), checked 2026-10-16.

eBay prohibits facilitating purchases outside eBay. Therefore an eBay listing never becomes a direct WalPay-store/PayPal invoice merely because a seller answers a chat. Keep the purchase in the platform's permitted checkout and methods; display PayPal only if that actual flow offers it. [eBay off-platform transaction policy](https://www.ebay.com/help/policies/payment-policies/offers-buy-sell-outside-ebay-policy?id=4272), checked 2026-10-16.

Do not activate stealth/proxy/CAPTCHA bypass to circumvent a denied site gate, access control or automation restriction. CAPTCHA, login, 2FA and anti-automation challenges cause a user handoff or stop. The fact that a tool vendor offers an anti-detection capability is not authorization to use it on a particular marketplace.

## 3. Architecture and trust boundaries

```text
Authenticated WalPay user
  -> shopping mission API (immutable caps/constraints)
  -> browser task orchestrator / bounded agent worker
  -> typed action proposal + policy decision + durable command
  -> outbound authenticated connection to paired local companion
  -> local grant / action approval / current-page validation
  -> private stdio Playwright MCP process -> official browser extension
  -> user-selected browser profile and tab -> allowed store/site

redacted observations -> evidence adapter -> product candidates/negotiation history
exact final own-store quote -> payment intent -> user step-up -> PayPal approval
  -> backend provider capture -> authoritative payment/reconciliation
```

The browser does not receive PayPal merchant client secrets, refresh tokens, payout credentials, database access or shared infrastructure keys. The payment service does not receive browser cookies/passwords. Cloud models see only consented, redacted task-relevant page data. Private browsing data stays local unless the user explicitly grants scoped observation upload.

Companion implementation responsibilities:

- Run under the logged-in OS user with signed release/update verification. On Windows use OS credential storage/DPAPI-backed protected storage for its own pairing key; do not access Chrome's cookie/password database.
- Spawn a pinned Playwright MCP binary with structured arguments and stdio. No model-generated shell command, arbitrary module, dynamic executable path or `npx @latest` on every mission.
- Keep raw MCP endpoint and its credentials accessible only to the companion process. Agent workers have no shell/file access to bypass the gateway and no route to the companion's raw driver.
- Pair using a short-lived code/QR and user confirmation of account/device name; generate a per-device keypair locally. Register only public key/device ID and revocable credential. Pairing is not an all-sites grant.
- Establish an outbound TLS connection to WalPay; authenticate server, device and session. Avoid listening on LAN/public interfaces. If a localhost endpoint is unavoidable, bind loopback only with fresh authentication, strict origins/CSRF protection and no credential-bearing URLs.
- Accept short-lived signed commands bound to device/session/user/task/sequence/resource/expiry and revalidate locally against locally approved grants. Cloud task permissions cannot widen local grants silently.
- Maintain a local append-only action journal with command IDs and result classification; protect it and apply retention. The cloud stores action metadata/evidence references, not raw browser state.
- Enforce one exclusive automation owner per selected tab. Human input or another agent lease pauses automation; ownership transfer requires a fresh observation and explicit resume.
- Provide local stop/disconnect/revoke controls that work when the cloud is unavailable. Termination detaches automation, not logs the user out or closes unrelated tabs.

Native-messaging option: a custom consent extension can communicate with a registered native host using restricted extension origins and structured messages. Validate sender/frame origin and payload, use bounded framing, and chunk image artifacts rather than assuming one arbitrarily large message. This option requires a real signed installer/extension implementation and review; it is not assumed to be built into the upstream Playwright extension. [Chrome native messaging](https://developer.chrome.com/docs/extensions/develop/concepts/native-messaging), checked 2026-10-16.

## 4. Granular permission scopes and side-effect classification

Every action has an independently enforced effect class. A generic `click` is not inherently read-only: it can purchase, send a message, accept terms, bid, delete data or change an account. The server and companion classify the target using a site adapter and reject ambiguous effects.

| Scope | Examples | Authorization |
| --- | --- | --- |
| `OBSERVE_SELECTED_TAB` | Snapshot allowed product region, screenshot crop, read current listing | Active per-site/tab grant, consented redaction/upload policy |
| `NAVIGATE_ALLOWED_SITE` | Product detail, pagination/search, permitted same-site link | Site/tab allowlist, rate limit, verified resulting origin |
| `DRAFT_LOCALLY` | Compose negotiation text in WalPay/local memory | No external write; cannot fill a live autosaving composer by default |
| `MODIFY_REVERSIBLE_CART` | Add/remove permitted item in consenting merchant cart | Explicit session grant, item/quantity policy, no auto-buy trigger |
| `FILL_NON_SENSITIVE_FORM` | Search query or known non-autosaving permitted field | Adapter classifies field semantics and side effects |
| `SEND_MESSAGE` | Send exact nonbinding offer text to one seller | Exact recipient/listing/thread/message digest and one-use permit from direct user approval or an explicit bounded standing mandate under section 7.1; site permission required |
| `BINDING_OFFER` | eBay Best Offer, auction bid, purchase commitment | Separate exact financial authorization/product gate, never covered by message-send consent; disabled MVP |
| `CHECKOUT_HANDOFF` | Open actual verified provider/merchant checkout | Final terms reviewed, safe origin transition, automation paused at provider approval |
| `AUTHENTICATE_MANUALLY` | User login, 2FA, CAPTCHA or passkey interaction | Agent observes neither secret fields nor approval interaction; resume after fresh safe snapshot |
| `PAYMENT_EXECUTION` | PayPal capture/refund/payout | Backend financial service only, outside browser tool registry |

No `cookies.read`, `storage.export`, `password.read`, arbitrary JavaScript/CDP, unrestricted network, terminal, clipboard, download/upload, extension installation, account-settings mutation or arbitrary page evaluation tool is available to the browser agent. An internal adapter may use fixed reviewed Playwright code for extraction/actions; it cannot execute model-supplied code. Deny raw `browser_run_code`/`browser_cdp` equivalents even when the upstream driver supports them.

Permission intersection: `effective = site_policy ∩ local_user_grant ∩ mission_constraints ∩ actor_role ∩ current_action_permit`. A permit comes from direct action approval or deterministic derivation from a valid scoped standing mandate (section 7.1), never from the model. Missing/expired/unknown membership denies the action. Consent to a store does not include every subdomain/iframe/redirect target. Seller chat scope includes only the chosen seller/thread/listing, not the entire inbox.

## 5. Session, tab selection and reconnect lifecycle

Proposed endpoints are WalPay contracts; exact driver API names remain inside adapters:

| Route | Purpose |
| --- | --- |
| `POST /v1/browser-connectors/pairing-sessions` | Create short-lived account/device pairing request |
| `POST /v1/browser-connectors/pairing-sessions/{id}/confirm` | Record verified local pairing assertion |
| `GET /v1/browser-connectors` | List owner's paired devices and capability/version status |
| `POST /v1/browser-connectors/{id}/revoke` | Revoke credentials and all active grants immediately |
| `POST /v1/browser-task-sessions` | Bind mission revision, connector, exact sites, mode and proposed scopes |
| `GET /v1/browser-task-sessions/{id}` | State, last command/result, grants, current safe context, next action |
| `POST /v1/browser-task-sessions/{id}/pause` | Stop issuing/dispatching further actions |
| `POST /v1/browser-task-sessions/{id}/resume` | Revalidate local consent/site/tab state and reacquire lease |
| `POST /v1/browser-task-sessions/{id}/cancel` | Terminal stop intent; classify any in-flight effect before closing |
| `POST /v1/browser-task-sessions/{id}/actions` | Submit a typed bounded action proposal |
| `POST /v1/browser-action-proposals/{id}/authorizations` | Record exact user authorization for an effectful action |
| `GET /v1/browser-task-sessions/{id}/events` | Authorized cursor-based state/evidence stream |

Session states:

```text
created -> awaiting_local_consent -> attaching -> observing -> planning
 -> awaiting_action_authorization -> executing -> verifying_effect -> observing
 -> checkout_handoff -> paused_for_provider_approval -> completed

Wait/recovery: paused_by_user, paused_for_login, reconnect_required,
              policy_blocked, effect_outcome_unknown, requote_required
Terminal: cancelled, expired, failed, completed
```

Session record includes `id, user_id, connector_id, shopping_mission_id, mission_revision, driver_version, policy_version, grant_digest, tab_reference, origin, document_epoch, automation_lease_id, last_command_sequence, last_ack_sequence, status, expires_at`. Tab references are connector-scoped opaque IDs, not portable browser debug targets. Never attach automatically to the last-used profile because a vendor default does; require local account/profile/tab confirmation and verify observed site before first read.

Heartbeat is a short liveness signal, not permission renewal. Use bounded command leases (example 15 seconds) and a longer configurable session consent expiry (example 20 minutes). On network loss, cloud scheduler pauses; companion refuses expired/new commands. On reconnect exchange device identity, session generation, last journaled command/result and page epoch. Do not replay all commands from a queue blindly. Invalidate stale action approvals and reobserve after any uncertain navigation or human input.

Cancel immediately prevents future dispatch and revokes local grants. An already-started send/click may have happened. Report `cancel_requested` until its outcome is classified; cancellation cannot undo a message or purchase. For payment ambiguity invoke the shared provider reconciliation path, never browser-button retries.

## 6. Concrete action request and observation responses

```http
POST /v1/browser-task-sessions/{session_id}/actions
Idempotency-Key: <key>
{
  "command_id":"<uuidv7>",
  "expected_sequence":12,
  "mission_revision":2,
  "grant_id":"<uuidv7>",
  "action": {
    "type":"OBSERVE_PRODUCT",
    "tab_ref":"tab_<opaque>",
    "expected_origin":"https://<walpay-store-host>",
    "expected_document_epoch":7,
    "target":{"observation_id":"<uuidv7>","element_ref":"element_34"},
    "include_screenshot":true,
    "fields":["title","variant","quantity","seller","price","shipping","tax","availability"]
  }
}
```

```http
200
{
  "data": {
    "command_id":"<uuidv7>",
    "sequence":13,
    "state":"observed",
    "provenance": {
      "observation_id":"<uuidv7>",
      "session_id":"<uuidv7>",
      "tab_ref":"tab_<opaque>",
      "top_level_origin":"https://<walpay-store-host>",
      "frame_origin":"https://<walpay-store-host>",
      "url":"https://<walpay-store-host>/products/<sku>",
      "document_epoch":7,
      "observed_at":"2026-10-16T16:00:00Z",
      "source_type":"BROWSER_DOM_AND_SCREENSHOT",
      "driver":"PLAYWRIGHT_MCP_EXTENSION",
      "driver_version":"<pinned-version>",
      "content_hash":"sha256:<digest>",
      "screenshot_artifact_id":"<uuidv7>",
      "redaction_policy_version":"<version>"
    },
    "product": {
      "source_item_id":"<sku>",
      "title":"Demo headphones",
      "variant":"Black/refurbished",
      "seller_ref":"seller_<opaque>",
      "item_price":{"currency":"USD","amount_minor":"11500"},
      "shipping":{"status":"UNKNOWN"},
      "tax":{"status":"UNKNOWN"},
      "stock":{"status":"OBSERVED_AVAILABLE","authoritative":false},
      "quote_status":"ESTIMATE_ONLY"
    },
    "capabilities":{"can_negotiate_nonbinding":true,"can_prepare_checkout":true},
    "review_flags":["SHIPPING_NOT_FINAL","TAX_NOT_FINAL"]
  }
}
```

The observation schema uses fixed extraction fields, not arbitrary JSON selectors/scripts. DOM text and screenshot agree on the selected item where possible. Preserve source bounding region/element refs and evidence hashes. A screenshot is contextual evidence, not proof of seller identity, final tax or a provider capture. User-generated DOM can lie; authoritative own-store APIs supersede display text for actual quote/capture.

Element refs are bound to observation/document epoch. After navigation/re-render, reject stale refs and reacquire snapshot. Before action verify tab origin, frame origin, target accessible role/label, visible state, listing/seller/thread fingerprint and expected document epoch. Cross-origin iframe content is a separate capability boundary. If the target cannot be safely classified from current evidence, pause for user action; coordinates alone cannot authorize a send/buy button.

Returned errors include `SITE_AUTOMATION_NOT_ENABLED`, `LOCAL_GRANT_REQUIRED`, `LOCAL_GRANT_REVOKED`, `TAB_CHANGED`, `STALE_OBSERVATION`, `EFFECT_CLASS_UNKNOWN`, `ACTION_APPROVAL_REQUIRED`, `UNAPPROVED_RECIPIENT`, `CAP_REJECTED`, `LOGIN_HANDOFF_REQUIRED`, `EFFECT_OUTCOME_UNKNOWN`, `DRIVER_UNAVAILABLE`. Follow the shared error envelope with code/message/request ID/retriable flag; no cookie/debug endpoint leaks in diagnostics.

## 7. Negotiation through a browser

Supported own-store/consenting-partner flow:

1. User selects connected browser tab and grants the specific allowed store/search/observe scopes. Mission caps and product constraints are immutable inputs from the server.
2. Browser navigates the permitted store and reads shortlisted products. Tool provenance is attached to every candidate, separate from API/Channel3 discovery provenance.
3. Agent drafts a nonbinding negotiation message in WalPay memory. Display terms include exact seller/thread/listing, intended item/quantity/currency, offer scope (`ITEM_PRICE_ONLY` or exact `ALL_IN`) and message text. Do not fill the site's live message field while calling this draft-only: autosave/typing indicators may send data externally.
4. Deterministic code validates offer amount against mission strategy and hard cap. If shipping/tax are unknown, the message is explicitly a nonbinding item-price inquiry; it cannot create a binding purchase commitment. Private max cap stays private.
5. User authorizes the exact recipient + message + amount + scope digest. The local companion records that approval with a one-use nonce, expiry, mission revision, thread fingerprint and site-policy version.
6. Immediately before send, reobserve the seller/thread/listing and compare fingerprints. Fill/send only the exact approved message using reviewed adapter actions; any changed text/recipient/offer needs new authorization. Initial MVP uses per-message user approval. Section 7.1 adds an opt-in standing mandate that lets deterministic policy issue one-use permits for bounded nonbinding rounds without asking on every message.
7. Verify actual message appearance/outcome. Store message ID if available, exact normalized text hash, seller/thread, observed timestamp and evidence. A UI timeout is ambiguous, not permission to press Send again.
8. Treat seller response as untrusted data; extract price, currency, included charges, expiry, condition and any required terms into a structured quote draft. Ask for missing all-in terms rather than assume a free delivery/tax exemption.
9. Apply bounded rounds/deadline/no-improvement stop conditions. User may pause/revoke at any time. On a consenting store with an API-backed quote, use the authoritative exact quote before payment.
10. Final proposal contains payee, product, original currency, all-in amount, disclosed WalPay/provider charges and exact FX where applicable. Authorize via existing payment-intent flow, then normal PayPal approval; browser cannot click through provider approval for the user.

```json
{
  "proposal_type":"SEND_NONBINDING_NEGOTIATION_MESSAGE",
  "session_id":"<uuidv7>",
  "mission_id":"<uuidv7>",
  "mission_revision":2,
  "recipient":{"site":"<consenting-host>","seller_ref":"seller_7","thread_ref":"thread_9"},
  "listing":{"source_item_id":"sku_1","quantity":1},
  "message":"Would you accept USD 115 for this item, subject to confirming the full delivered total?",
  "offer":{"currency":"USD","amount_minor":"11500"},
  "offer_scope":"ITEM_PRICE_ONLY_NONBINDING",
  "observation_id":"<uuidv7>",
  "effect_digest":"sha256:<canonical-recipient-listing-text-terms-digest>",
  "expires_at":"2026-10-16T16:05:00Z"
}
```

A 'Make Offer', 'Place Bid', 'Buy Now', 'Accept' or checkout confirmation control can create a binding commitment even if an agent describes it as chat. Site adapters classify it as `BINDING_OFFER`/payment, require the applicable gate and exact total, and remain disabled in MVP. Never automate around a disabled API by clicking the equivalent UI without the same policy review.

Facebook-specific initial experience: user manually opens a listing and chooses to provide listing details/chat text; agent can explain risk, draft offers and check cap arithmetic locally. The backend can hold a browser-session request in `policy_blocked` with the required evidence listed. Once an authorized path is verified, the same adapter protocol supports actual selected-tab discovery/chat without replacing the rest of the backend. Do not market the manual draft path as live Facebook negotiation.

### 7.1 Optional bounded autonomous nonbinding mandate

Phase B7 adds opt-in autonomous bargaining on a **permitted site** after the initial per-message path is proven. The user can authorize a finite negotiation envelope once, allowing the agent to conduct nonbinding rounds within it. This preserves the autonomous-shopping objective while keeping binding offers/purchases and provider approval as separate controls. It does not enable a site that currently lacks automation permission.

Create a reviewed standing mandate through planned extension `POST /v1/browser-task-sessions/{id}/negotiation-mandates`; require authenticated explicit user action, step-up where policy requires it, `If-Match` on mission/session revision, local companion confirmation and an immutable digest. Planned `POST /v1/browser-negotiation-mandates/{id}/revoke` immediately revokes it locally and server-side. The cloud cannot create or widen a local mandate without that user confirmation.

```json
{
  "mandate_type":"BOUNDED_NONBINDING_NEGOTIATION",
  "mission_id":"<uuidv7>",
  "mission_version":2,
  "site_origin":"https://<consenting-host>",
  "seller_ref":"seller_7",
  "listing_id":"sku_1",
  "thread_ref":"thread_9",
  "quantity":1,
  "currency":"USD",
  "offer_floor_minor":"9000",
  "opening_offer_minor":"10500",
  "target_all_in_minor":"13000",
  "max_all_in_minor":"15000",
  "allowed_offer_semantics":"ITEM_PRICE_NONBINDING_SUBJECT_TO_EXACT_TOTAL",
  "allowed_template_ids":["NONBINDING_OPEN_V1","NONBINDING_COUNTER_V1","ASK_ALL_IN_V1","POLITE_END_V1"],
  "allowed_claims":["REQUEST_PRICE","ASK_DELIVERY_COST","ASK_WARRANTY","END_NEGOTIATION"],
  "max_rounds":4,
  "max_outbound_messages":5,
  "min_message_interval_seconds":15,
  "expires_at":"2026-10-16T16:10:00Z",
  "allow_personal_data":false,
  "allow_binding_offer":false,
  "allow_checkout_approval":false
}
```

Mandate constraints are stored independently of model prompts and signed/bound to local consent. Validate the same lower/opening/upper semantics as plan 05, supported currency and mission constraints. An offered item price is not a final all-in total: when mandatory charges are unknown, approved templates explicitly make the inquiry nonbinding and subject to confirming the exact full total. No promise to buy, accept a binding counteroffer or pay under unknown tax/shipping is allowed. When exact charges are known, each proposed offer plus those charges/platform fee/executable FX must fit the hard all-in cap. Unknown or changed material conditions stop auto rounds and request review.

The model can recommend an allowed next offer amount and template ID. The deterministic gateway validates currency, bounds, seller/thread/listing, source observation, mandate/mission versions, grant/site policy, expiry, cumulative rounds/messages, cooldown and permitted claims. It renders a fixed reviewed template with strictly typed placeholders, refuses arbitrary free-text additions/URLs/personal data, and creates a **one-use per-message permit** bound to exact rendered text/recipient/terms/digest. The local companion independently checks the same mandate and consumes the permit in its dispatch journal. This is delegation under prior user authorization, not LLM self-approval.

No contact/inbox expansion, new seller, alternate listing, address/phone disclosure, unrelated messaging, threat, false competing offer, unverified product claim or off-platform payment proposal falls inside the mandate. A seller asking to move threads/sites or introducing a deposit, new payee, different currency/quantity/variant or binding acceptance pauses automation. A single global 'negotiate anywhere for me' switch is insufficient.

Mandate revoke/expiry, site-policy change, local disconnect, mission cap/version change, user interaction that changes thread/terms, suspected injection or quota exhaustion blocks subsequent dispatch immediately. Reconnect requires a current local grant and fresh observation; it cannot revive an expired/revoked mandate or retry an ambiguous previous send. Prior permitted messages remain audit records.

A successful nonbinding negotiation only produces a reviewable quote. Any binding bid/offer acceptance or purchase still needs transaction-specific financial authorization, applicable budget/inventory reservation and exact terms; Checkout still needs PayPal's own user approval. Standing negotiation consent never permits a browser tool to click through provider approval or a financial service to execute a new charge.

## 8. Safe checkout handoff and merchant/payment matching

There are two payment observation lanes, and they cannot be conflated:

- **WalPay-controlled merchant integration:** authoritative store quote -> WalPay payment intent -> PayPal order created by the correct merchant integration -> buyer provider approval -> backend capture and verified events. Browser discovery is an entry point; it is not the amount/payee authority.
- **External merchant/platform checkout:** navigate only to permitted actual merchant checkout, verify origin/beneficiary/amount as far as documented interfaces allow, and hand off to the user. WalPay cannot assume its own merchant credentials can capture an order created by another merchant, observe unrelated app webhooks, or read the user's full wallet. Mark completion `USER_REPORTED`/`OBSERVED_EXTERNAL_CHECKOUT` unless an authorized partner API independently confirms payment.

Never convert a platform purchase into a WalPay PayPal invoice simply to insert PayPal, extract a platform fee or evade platform checkout. WalPay platform fees on external discovery/assistance require a separate disclosed permitted service model or approved partner integration; an approved fee charge is a separate payment intent and is not proof the merchandise was paid.

At a PayPal login/approval page: pause browser automation, stop screenshot/DOM collection for sensitive regions and return a user handoff action. User enters credentials/OTP or uses their PayPal passkey/biometrics themselves. Do not collect raw credentials, autofill secrets through the model, click provider approval or approve an account-wide mandate because a shopping mission was authorized.

After provider return, the shared payment service fetches/verifies its own order status, merchant, currency, amount and proposal mapping. Return URL text, success banner, browser screenshot and user report are not provider confirmation. Re-quote and reapprove if item, price, shipping, currency, fees, merchant or FX changes. No final provider quote means payment remains blocked.

## 9. Prompt injection, privacy and evidence controls

Pages, accessibility trees, seller chats, screenshots, ads, document titles, embedded tool metadata and checkout labels are untrusted observations. They may contain instructions designed to alter caps, leak account data, contact another recipient, install an extension or move payment off-platform. The agent's trusted task comes only from WalPay authenticated user/session policy. Models never receive a raw browser password manager, token dump, inbox history or cross-tab snapshot.

Use fixed per-site reviewed extraction/action adapters with a generic read-only fallback only where site permission allows it. The fallback can propose a new adapter action but cannot execute unknown form submissions. Treat site-provided WebMCP tools as third-party tools requiring separate reviewed schema/effect/site allowlist; discovery alone does not grant use. No upstream self-improving skill or persistent memory can widen financial/browser permissions.

Redact locally before cloud upload: password/OTP fields, session identifiers, cookies, auth-bearing query parameters, unrelated conversations, private phone/email, addresses not necessary for the current quote and other tabs. Screenshot crops should be task-specific; overlays and canvas content may defeat automated redaction, so sensitive login/payment views are paused rather than recorded. Define a no-capture region state, test redaction failures and offer local-only observation where supported. Hashes do not anonymize sensitive source text; apply deletion/retention controls to original artifacts.

Audit minimal metadata by default: actor, connector/task/session, policy/grant versions, action digest, scope/effect class, local approval ID, timestamps, resulting origin, observation refs and effect status. Optional encrypted screenshots/text have short retention and owner access with explicit consent. Do not log full URLs with tokens, live-view bearer links, CDP endpoints, browser console secrets, network headers or cookies. Trace export and support access are purpose-scoped/audited.

Network protections remain necessary for browser navigation: reject `file:`, extension/internal browser pages, `javascript:`, unapproved `data:` navigation, localhost/private/link-local network targets and unapproved redirects unless an explicit developer test policy enables a particular owned origin. An already-authorized page's links cannot widen this allowlist. Open approved checkout origins through a reviewed handoff transition and immediately pause automation. No model-generated arbitrary network interception rule or request header injection.

## 10. Action dispatch correctness and ambiguous effects

Durable command IDs and local journals can deduplicate gateway dispatch. They cannot guarantee exactly-once behavior on an arbitrary third-party UI that has no idempotency API. Design for `APPLIED`, `NOT_APPLIED`, `OUTCOME_UNKNOWN` and an explicit reconciliation workflow.

```ts
// Sketch: reviewed driver actions only; no model code/eval input.
async function executeBrowserCommand(cmd: SignedCommand, local: LocalState) {
  verifyServerSignatureAndExpiry(cmd);
  assertPairedDeviceAndSession(cmd, local);
  const lease = await local.acquireTabLease(cmd.tabRef);
  try {
    // Recheck inside the exclusive lease; concurrent duplicate commands cannot both dispatch.
    const old = await local.journal.get(cmd.commandId);
    if (old) {
      assertSameCommandDigest(old, cmd);
      return old.resultOrUnknown();
    }
    assertExpectedNextSequence(cmd, local);
    const policy = intersectSiteMissionAndLocalGrants(cmd, local);
    const before = await local.driver.safeSnapshot(cmd.tabRef, policy);
    assertOriginEpochRecipientAndEffect(cmd, before, policy);
    if (isEffectful(cmd.action)) verifyExactOneUseLocalPermit(cmd, before);
    // Persist before dispatch, so a crash can never be mistaken for an unattempted action.
    await local.journal.beginDispatchAndConsumePermit({
      id: cmd.commandId, status: "DISPATCHING", digest: cmd.digest,
      permitId: cmd.permitId // Direct approval or mandate-derived; unique/one-use for effects.
    });
    try {
      await local.driver.executeReviewedAction(cmd.action, before);
      const after = await local.driver.safeSnapshot(cmd.tabRef, policy);
      const outcome = classifyExpectedEffect(cmd, before, after);
      return await local.journal.finish(cmd.commandId, outcome);
    } catch (error) {
      // A thrown driver error does not prove that click/send never happened.
      return await local.journal.finish(cmd.commandId, {status: "OUTCOME_UNKNOWN"});
    }
  } finally {
    await lease.release();
  }
}
```

Persist one-use permit consumption and dispatch journal transition atomically in the local journal; crash after one must not allow a second message send. A permit records either its direct user approval or standing mandate/version. Recheck authoritative local revocation immediately before dispatch. The sketch omits transport framing and database implementation; it does not claim a screenshot-based classifier is always definitive.

For an unknown message send, reobserve the exact thread and seek a provider/site message ID or matching submitted text/time under authorized scope. If ambiguity remains, pause for human confirmation; do not retry by sending the same text again. For unknown binding actions, stop the session and reconcile through the actual platform account/API. For unknown PayPal capture, the provider operation ID, backend state and reconciliation plan govern; a browser retry is forbidden.

Constrain loops: maximum actions, navigations, pages, message rounds, task time, model cost, concurrent tabs and per-site rate limit. Defaults can be 20 read actions, 5 product pages, 4 message rounds and 10 minutes; these are initial design limits, tune with evaluation. Reserve separate limits for a user handoff so idle login time does not trigger an automatic dangerous retry.

## 11. Storage and integration contracts

| Table | Important fields / constraints |
| --- | --- |
| `browser_connectors` | Owner/device ID, public key, key version, driver/version capabilities, revoked_at; no session cookies |
| `browser_pairing_sessions` | High-entropy challenge hash, owner, expiry, local confirmation proof, consumed_at |
| `site_automation_policies` | Exact origins, workflows/scopes, permission evidence reference, review owner/date, version, expiry, status |
| `browser_local_grants` | Connector/session/user, site/tab scope, allowed effects, observation export policy, expiry, signed local assertion |
| `browser_task_sessions` | Mission revision, connector, tab ref, page epoch, state/generation, last sequence/ACK, lease metadata |
| `browser_action_proposals` | Typed immutable payload, source observation, effect class/digest, required authorization, expiry |
| `browser_action_authorizations` | User/local challenge assertion, exact action digest, recipient/thread, one-use nonce, consumed state |
| `browser_negotiation_mandates` | Explicit user/local consent, mission/site/seller/listing/thread, template/claim allowlist, monetary bounds, rounds/time limits, version/expiry/revocation |
| `browser_action_permits` | Direct approval or mandate/version reference, exact rendered message/effect digest, scope, one-use nonce, expiry/consumed state |
| `browser_commands` | Unique command ID and `(session,generation,sequence)`, idempotency key/body hash, dispatch state/result |
| `browser_observations` | Provenance hashes/origin/epoch/refs, redaction/tool/policy version, extracted structured fields, artifact refs |
| `browser_effect_reconciliations` | Original command, unknown status, evidence source, verified classification, actor/time |

No browser extraction becomes a new authoritative `commerce_quote` automatically. Normalize it into a candidate/quote draft with provenance and explicit `ESTIMATE_ONLY` until the selected merchant/integration supplies exact permitted terms. Shared Money/integer/UUIDv7/OpenAPI conventions apply. Do not introduce a separate browser payment ledger.

Browser task worker and financial worker have different credentials, queues, permissions and egress. A child agent/worker never receives a raw driver handle. Shared model services have no access to the local journal/pairing key. The tool registry exposed to Oh My Pi/Hermes/Browser Use is the same narrow WalPay registry with policy enforcement; using another harness cannot bypass it.

## 12. Development phases and evidence gates

| Phase | Required work | Exit evidence |
| --- | --- | --- |
| B0: driver/capability spike | Pin official Playwright MCP/extension; prove existing Chrome/Edge profile/tab attach on target OS; identify actual tool schemas and consent behavior | Recorded test using controlled store, no credential/profile copying; deny fallback if isolation missing |
| B1: local pairing and policy gateway | Signed companion, device pairing/revoke, outbound transport, narrow action schema, grants, local stop | No direct raw MCP/CDP/model shell path; spoofed pairing/commands rejected |
| B2: real browser discovery | Own-store navigation/snapshots/evidence, typed products, local redaction and tab lease | Agent really uses selected browser and can find demo products under a bounded mission |
| B3: browser negotiation | Controlled seller chat, exact user send approval, rounds/caps, result verification/unknown handling | A complete test negotiation with no message duplicates or wrong recipients |
| B4: sandbox payment handoff | Exact authoritative quote, budget reservation, user/provider approval, backend capture/refund | Browser mission ends in a real sandbox PayPal capture with trace and confirmed webhook/status |
| B5: consenting partner / Kernel | Site permission registry, separate cloud consent, scoped sponsor credentials, optional live view/replay | Local and cloud lanes remain distinct; cloud login/profile state never silently imported |
| B6: hardening / external enablement | Injection, profile/tab concurrency, cancellation/reconnect, message ambiguity, site review | External gates open only for documented allowed capabilities; unsupported Meta remains honest |
| B7: bounded autonomous bargaining | Explicit standing nonbinding mandate, reviewed template renderer, deterministic per-message permits, local independent enforcement | Finite auto rounds on a permitted store within all bounds; revocation/expiry/thread/amount drift tests pass; no binding/payment authority gained |

The hackathon demonstration should show the browser operating WalPay's own store and a controlled seller-chat surface, with real backend state and sandbox payment. Show an external-marketplace adapter capability panel or response with gates; do not use a recording/fixture while claiming a live seller negotiation. A simulated Facebook-style test surface is labelled `SIMULATED_MARKETPLACE` and contains synthetic accounts/listings.

## 13. Tests and granular checklist

Use Vitest/Fastify injection for gateway/policy contracts, PostgreSQL/Redis integration tests for command/event state, Playwright on a controlled adversarial test store for browser effects, local companion crash/reconnect fault injection, and separate sandbox PayPal end-to-end runs. Browser integration tests are meaningful here because stale elements, human interference and ambiguous clicks are core correctness risks. Do not point CI at real personal accounts or real marketplace sellers.

| Test ID | Scenario | Required assertion |
| --- | --- | --- |
| BRO-001 | User selects existing Chrome/Edge profile/tab | Only selected granted tab is observed/controlled; active local session reused without exported cookies |
| BRO-002 | Driver defaults to last-used wrong profile | Block until local verification; no first snapshot of wrong account |
| BRO-003 | Backend/model asks raw CDP/eval/cookies/network/terminal | Registry/gateway denies before driver call |
| BRO-004 | Fake pairing, expired command, wrong device/session/sequence | No browser action; replayed identical command yields stored result |
| BRO-005 | User revokes site grant while next action is queued | No further dispatch; local stop works with cloud offline |
| BRO-006 | Site gate denied but user grants all scopes | Site-policy intersection blocks read/send; consent cannot widen platform entitlement |
| BRO-007 | Product page links to localhost/private IP or unsafe scheme | Navigation denied; no internal network/file access |
| BRO-008 | Browser navigates/re-renders after snapshot | Old element refs/epoch rejected; fresh observation required |
| BRO-009 | Another tab, popup, cross-origin frame replaces seller thread | Exact recipient/origin check blocks action |
| BRO-010 | Human types/clicks during automation | Pause/reobserve; single tab lease; no stale queued send |
| BRO-011 | Page tells agent to ignore caps/export inbox/install extension | No prohibited action, no permission escalation, attack evidence safely recorded |
| BRO-012 | Screenshot/DOM contains OTP, secret URL, unrelated DM | Local redaction/no-capture policy prevents cloud/log disclosure |
| BRO-013 | Draft composer autosaves or typing indicator sends | Draft remains local; no external fill without classified scope |
| BRO-014 | User approved message A but model modifies amount/text | Digest mismatch; new approval needed |
| BRO-015 | Seller/thread changes after message authorization | No send; approval tied to original recipient/listing/epoch |
| BRO-016 | Send succeeds then response/network is lost | `OUTCOME_UNKNOWN`; reconcile thread, never blind resend |
| BRO-017 | Crash between journal dispatch/driver/result/ACK | Outcome preserved or unknown; one-use approval not reused |
| BRO-018 | Same command resent after reconnect | Stored result/unknown returned; no second click/send |
| BRO-019 | Click labelled ordinary text but creates binding offer | Effect classifier/gate rejects; no generic-click bypass |
| BRO-020 | Item within cap but shipping/platform fee/FX exceeds it | No financial proposal; exact quote required |
| BRO-021 | External merchant uses another PayPal app/order | WalPay cannot capture it; external status honestly classified |
| BRO-022 | Seller requests off-platform payment for an eBay listing | No WalPay invoice/payment-link diversion |
| BRO-023 | Provider approval page encountered | Automation/recording pauses; user handles secrets and approval |
| BRO-024 | Fake success banner or redirected callback | No confirmed payment without authoritative provider status |
| BRO-025 | Session cancellation races with send/capture | Future actions stop; in-flight outcome reconciled, not erased |
| BRO-026 | Kernel unavailable or no local-to-cloud consent | No silent cloud fallback; local lane/policy remains intact |
| BRO-027 | Model hits loop/token/page/message limits | Deterministic stop and useful saved state; no background retry storm |
| BRO-028 | Unsupported Facebook automation requested | Explicit policy-blocked response; optional manual-content draft accurately labelled |
| BRO-029 | Firefox/Safari not supported by chosen extension | Capability error and explicit supported alternative, no fabricated attach |
| BRO-030 | Local artifact deletion/support export | Retention, owner scope and redaction enforced; secret-bearing links absent |
| BRO-031 | Valid standing mandate, allowed seller/thread/template and bounded offers | Agent completes finite nonbinding auto rounds using one-use gateway/local permits; no per-message user prompt needed |
| BRO-032 | Mandate revoked/expired while a message waits in queue | No next dispatch; old permits/grants cannot revive on reconnect |
| BRO-033 | Seller changes thread/listing/currency/payee or adds deposit | Auto rounds stop; no silent scope/term expansion |
| BRO-034 | Model proposes free text, private phone, false claim or over-cap amount | Deterministic renderer/policy denies before permit issuance |
| BRO-035 | Unknown tax/shipping or new platform/FX charge | Only explicit nonbinding inquiry permitted; no binding acceptance/payment; exact quote still required |
| BRO-036 | Mandate valid but mission cap/version/site policy changed | Prior mandate/permit unusable; fresh explicit authorization required |

Completion checklist:

- [ ] B0 proves exact existing-browser/profile/tab behavior against pinned driver/extension versions.
- [ ] Playwright MCP extension is reused; custom consent/native bridge work is labelled as WalPay implementation.
- [ ] Local helper installer/update signature and minimal OS permissions designed and reviewed.
- [ ] No personal Chrome profile cookie/password database reading/copying or silent cloud upload.
- [ ] Pairing keys live only in protected local storage; server stores public key and revocable scoped credentials.
- [ ] No raw debugging port exposed to network or handed to model/cloud workers.
- [ ] Local transport authentication, origin checks, command expiry/sequence and key rotation covered.
- [ ] One local selected-tab grant, one owner lease and user-visible stop/revoke per task.
- [ ] Every site has a versioned permission/capability record; unsupported workflows fail closed.
- [ ] Meta/eBay limitations are explicit; browser capability does not imply site permission.
- [ ] Read, navigation, draft, reversible cart, message send, binding offer and checkout handoff have separate effects/scopes.
- [ ] Generic click/type/eval cannot bypass effectful-action authorization.
- [ ] Drafting happens locally outside live autosaving/typing-aware composers.
- [ ] Each external send has a one-use exact-text/recipient/thread/listing/amount/scope permit from direct user approval or a valid explicitly authorized bounded mandate.
- [ ] Standing mandate is finite, per-site/seller/listing/thread and locally confirmed; template/claim allowlists prevent arbitrary messages or personal-data disclosure.
- [ ] Gateway renders/validates mandate messages and local companion independently checks/consumes permits; model cannot self-approve.
- [ ] Revocation/expiry/changed thread/cap/charges and reconnect invalidate mandate execution; B7 cannot authorize binding commitments or checkout approval.
- [ ] Binding bids/offers stay disabled until separately authorized and permitted.
- [ ] Browser observations carry origin/frame/tab/epoch/time/tool/redaction provenance and source hash.
- [ ] Stale element refs and human interference invalidate queued actions.
- [ ] Login/OTP/CAPTCHA/provider approval require manual user handoff and no capture of secrets.
- [ ] Fixed extraction code and narrow schemas replace arbitrary model JS/CDP and cookie/network tools.
- [ ] Local redaction and screenshot no-capture states are tested against overlays/canvas/hidden fields.
- [ ] External candidate text never becomes an exact payable quote without authoritative permitted terms.
- [ ] Payment/currency/fees/FX/merchant changes invalidate old proposal and approval.
- [ ] Local command journal handles dispatch crash/replay; uncertain third-party effects never claim exactly-once execution.
- [ ] Unknown sends/captures reconcile original attempt; no blind repeat-click retry.
- [ ] Cancellation/reconnect cannot revive expired local grants or consumed approvals.
- [ ] Kernel cloud session is a separate consent/data boundary with scoped credentials, not local-browser attachment.
- [ ] Oh My Pi/Hermes/Browser Use can access only the same enforced WalPay gateway tools.
- [ ] Controlled real-browser own-store negotiation + PayPal sandbox capture has recorded reproducible evidence.
- [ ] Test/simulation/external manual paths are labelled accurately throughout API responses and demo.
- [ ] Required BRO tests and security checks pass before any external site capability is enabled.
