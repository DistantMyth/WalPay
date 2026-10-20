# Security, identity, authorization and privacy

Proposed controls for a global money-related application. Implement and verify these before enabling external mutations. Separate user authentication, PayPal identity connection, merchant authority, buyer payment approval and partner payout authority. Possessing one never implies the others.

## Security objectives and trust boundaries

Protect money intent integrity, recipient identity, spending limits, user sessions, contact privacy, provider credentials and source evidence. Untrusted inputs include browser pages/DOM/screenshots, seller messages, receipt OCR, model output, partner payloads before verification, webhook bodies before verification and arbitrary QR text. AI reasoning is untrusted for authorization even when it produces valid JSON.

| Threat | Concrete failure | Required control / evidence |
|---|---|---|
| Object authorization bypass | Read another user's invoice or alter their request by ID | Principal-scoped query + tenant-aware FK + route negative tests |
| Recipient substitution | Page or model switches PayPal payee after consent | Immutable payee binding, quote digest and provider payee verification |
| Prompt injection | Seller listing instructs agent to reveal budget/secrets or pay elsewhere | Tainted content, scoped tool gateway, no model payment credential |
| Replay/double effect | Duplicate capture, job approval or webhook creates extra money movement | Durable operation uniqueness, inbox/outbox and stable provider key |
| Concurrent overspend | Several agents each read same remaining budget | Atomic locked reservations across all applicable buckets |
| Account takeover | Stolen session changes recipient or enrolls passkey | Step-up, session rotation, recovery controls and alerts |
| Partner forgery | Attacker marks job approved | Signed exact payload, key binding, task version and funded obligation policy |
| Upload exploitation | Huge/hostile image, PDF or OCR payload | Private quarantine, size/decode limits, patched isolated scanner |
| SSRF | Webhook certificate URL, product URL or agent link accesses local secrets | Egress allowlist and redirect/DNS/IP validation |
| Browser session theft | Remote service receives cookies or debugging control | Local connector, scoped tab grants, no cookie export, capability revocation |
| Contact enumeration | Attacker probes phone numbers | Exact consented lookup, HMAC indexes, uniform failures and abuse limits |
| Fee/FX manipulation | Hidden fee or stale conversion exceeds approved spend | Immutable disclosed breakdown, exact FX quote or fail closed |
| Insider misuse | Operator edits journal or submits manual payout | Restricted roles, append-only audit, separate reviewer for live exceptions |
| Misleading success | Browser return or accepted batch shown as paid | Provider-confirmed resource evidence and item-level state |

Use OWASP API and LLM risk categories as review aids; completion evidence is the actual test/control, not a checked category name. No checklist alone establishes regulatory compliance.

## Sessions and identity lifecycle

Use a cryptographically random >=256-bit opaque session token, hash it in PG, and set Secure/HttpOnly/SameSite cookie attributes. Rotate on login, role/tenant change and recovery. Suggested policy: 30-minute inactivity, 12-hour absolute lifetime for normal sessions; configure accessible warning/reauth without losing draft. High-risk authorization grant lifetime is shorter (default120 seconds to start order), with payment workflow expiry separately controlled.

Authenticate every mutation and enforce origin+CSRF for cookie sessions. CORS is an explicit origin allowlist, never wildcard credentials. Logout revokes session and browser grants, closes SSE and prevents new tools; submitted provider operations continue reconciliation so logout does not lose money evidence. Lost-device recovery uses verified recovery factors with notification/cooldown for recipient changes. Email/phone verification is not itself PayPal funding authority or proof that two accounts belong to the same legal person.

Phone change invalidates discoverability mapping and recovery bindings according to policy. Verify the new phone; never transfer old obligations by matching the reused number. Contact resolution returns a recipient preview requiring confirmation before financial action. Recipient's provider identity is snapshotted when an operation is frozen; later profile edits do not retarget it.

## Transaction-bound WebAuthn verification

WebAuthn verifies scoped public-key credentials; biometrics remain on the authenticator. The server can require user verification but should not promise it can distinguish fingerprint, face or device PIN. [W3C WebAuthn](https://www.w3.org/TR/webauthn-3/). Use the maintained verifier rather than writing signature/CBOR parsing; verify against the exact relying party and allowed origin. [SimpleWebAuthn server guidance](https://simplewebauthn.dev/docs/packages/server).

1. Resolve current user-owned intent and exact quote revision; reject expired or incomplete tax/shipping/fees.
2. Compute a deterministic financial digest over purpose, mode, tenant, payer, merchant/payee binding, quote version, item identities, total currency/amount, all fee components, FX quote ID if executable, shipping-address reference/hash and expiry. Use a reviewed canonical serializer, never ordinary key-order-dependent JSON concatenation.
3. Generate >=32 random bytes for a challenge; persist its hash, raw challenge as required by verifier storage, intent digest, user, session, expiry and `unused`. Opaque challenge is linked server-side to terms; do not claim an authenticator displays transaction text.
4. Return the canonical readable summary and WebAuthn options. Future clients must show this summary before invoking the authenticator. A compromised client remains a risk; conventional WebAuthn alone is not a trusted transaction display.
5. Verify challenge, exact allowed origin, RP ID hash, credential ownership, signature and required UV flag. Apply library guidance for sign counters and backed-up credentials; a zero/nonmonotonic counter is not universally conclusive theft evidence.
6. Atomically mark challenge consumed and issue a one-use grant bound to the digest and local order operation. Replay loses the row-state race and fails.
7. Before order submission, re-read canonical terms, capability, reservation and revocation. Consume grant for the named operation exactly once. Retries of that same operation use the same recorded grant consumption.
8. Provider approval is independently recorded for the same order/payee/amount. Capture may proceed only if frozen terms still match, policy/mandate validity holds and provider approval is confirmed. If the payment sits beyond the configured authorization/consent window, require a new explicit local authorization, not reuse of an expired grant. Record a separate capture permit referencing the authorized frozen intent; it cannot authorize another order.

Illustrative service sketch:

```ts
async function acceptAssertion(cmd: AssertionCommand, principal: UserPrincipal) {
  const expected = await challenges.loadScoped(cmd.challenge_id, principal);
  const verified = await verifier.verify({
    response: cmd.credential_response,
    expectedChallenge: expected.challenge,
    expectedOrigin: config.allowedOrigins,
    expectedRPID: config.rpId,
    requireUserVerification: true,
    credential: expected.credential
  });
  if (!verified.ok) throw unauthorized('ASSERTION_INVALID');
  return db.transaction(async tx => {
    const row = await tx.challenges.lock(expected.id);
    if (row.consumed_at || row.expires_at <= clock.now()) throw conflict('CHALLENGE_EXPIRED_OR_USED');
    const intent = await tx.intents.lockOwned(row.intent_id, principal.userId);
    if (digestFinancialTerms(intent) !== row.financial_digest) throw conflict('TERMS_CHANGED');
    await tx.challenges.consume(row.id);
    return tx.authorizations.createForFrozenIntent(intent, principal, verified.credentialId);
  });
}
```

This is pseudocode with abstract adapter APIs, not runnable SimpleWebAuthn code. Actual library argument names must be pinned and compiled against the installed version. Challenge completion and grant creation are one transaction; cryptographic verification can occur outside the lock, followed by atomic freshness checks.

## PIN meaning and accessible fallback

Preferred PIN path: authenticator/OS device PIN through WebAuthn; WalPay never receives it. If a separate app PIN is later required, it is a low-entropy secondary factor with an established authenticated device/session, not the sole remote payment authenticator. Store with Argon2id and a separately held pepper, rate-limit per account/device/IP, lock with safe recovery after limited attempts, and never log it. Do not collect a PayPal PIN/password or biometric template. Alternative recovery/passkey/security-key paths must exist for people unable to use a particular biometric modality. Accessibility fallback may require stronger assisted verification; it must not silently reduce a payment to an unauthenticated command.

## AI and tool authorization

Every tool request has service identity, user delegation, mission/session ID, purpose, allowed hosts/resources, expiry and single-use or bounded-use nonce. The gateway reloads current policy and validates arguments against code-owned schemas. Read tools get no write credentials. Draft tools cannot send messages. Send-offer tools require explicit user consent appropriate to the bound seller/thread/amount, either exact action approval or a finite standing nonbinding mandate under16 section7.1; the deterministic gateway and local connector derive/consume a one-use permit for each allowed rendered message. Legally binding bids/offers require transaction-specific approval and strict cap reservation. Browser controls are detailed in16.

Do not send the hidden hard cap, full contacts, unrelated history or credentials to seller-facing tools. A source page saying “user authorized” is evidence from the page, never a grant. Store tool inputs/outputs with sensitive-field redaction and content provenance; model chain-of-thought is not required. Retain concise decision summaries and policy outcomes instead.

Model/provider failures can fall back to manual entry or a new proposal; they cannot choose a different payee, model-authorized spend limit or payment provider. Development coding agents and their shell/MCP permissions are separate from the deployed product agent. Never run a production money workflow in a developer harness with broad filesystem credentials.

## Provider and network controls

- Server-side OAuth credentials only; public client IDs may be exposed as documented, secrets never.
- Separate bindings/webhook IDs per environment and merchant; hostname alone does not identify an event's owner.
- Pin provider API base origins from config. Allowlist certificate fetch origins/paths and validate HTTPS, redirects, IP ranges and bounded response size for self-verification; prefer official postback verification for real events initially.
- Reject URLs containing credentials, custom protocols, IP literals/private/link-local hosts and unexpected ports. Re-check resolved address at connection/redirect to prevent rebinding.
- Verify partner signatures over exact raw bytes before parsing effects; canonical format/version, timestamp tolerance and nonce/event ID deduplication are documented per partner.
- Use parameterized SQL, output escaping in downstream clients, spreadsheet CSV injection protection and private object ACLs.
- Compile route schemas from code only; a seller/model cannot supply a schema executed by Ajv.

## Privacy and data governance

Minimize data by purpose: receipt workers receive receipt pixels, not contacts; commerce agents receive shipping fields needed for quote, not full address until required; earnings reports show counterparties only to authorized account owners. Cloud OCR/model transfer requires clear consent and configurable local processing fallback. Strip image EXIF and detect/redact sensitive document content before third-party processing where feasible.

Proposed retention defaults to confirm before release: raw receipts30 days after extraction, routine redacted model traces7 days, security logs90 days, contact invites7 days, operation/idempotency/audit records according to the documented payment/business/legal retention requirement. These defaults are product decisions, not claims of universal legal sufficiency. Exports and deletion jobs require identity verification, audit record and controlled object cleanup. Keep provider IDs needed for refund/reconciliation even when display identities are pseudonymized where retention is permitted/required.

Live launch gate: named owner reviews actual countries, transfer/FX business model, fees/surcharges, charitable fundraising, privacy/data transfers, KYC/AML/sanctions responsibilities and provider agreements. Use qualified review before enabling new regulated activities; sandbox technical success cannot grant a license or provider permission. US provider terms apply to US accounts, not every global user. [PayPal US agreement scope](https://www.paypal.com/us/legalhub/paypal/useragreement-full).

## Roles and operational authority

Roles: shopper, merchant operator, worker, partner approver, cause administrator, support read-only, finance operator, security operator. Route handlers derive role from server records, never a body field. Support cannot submit captures/refunds/payouts. Live manual financial exception requires a recorded reason and a second authorized reviewer; no direct database edits. Emergency kill switches can stop new money submissions, site automation or a single partner while reconciliation remains enabled.

## Security checklist

- [ ] Threat-model each core journey and attach concrete exploit tests.
- [ ] Verify tenant/user scoping on all object and SSE/export routes.
- [ ] Test price/recipient/currency/fee/FX mutation after step-up invalidates approval.
- [ ] Test challenge replay, concurrent assertions, wrong origin/RP, missing UV, revoked credential and recovery.
- [ ] Test connector revocation, malicious page instructions and prohibited payment-page control.
- [ ] Verify secrets never reach model/browser/telemetry/export payloads.
- [ ] Exercise egress, webhook certificate SSRF and upload decompression cases.
- [ ] Set cookie/CSRF/CORS/CSP client contract and rate limits.
- [ ] Run dependency, secret, container and static scans; triage actual reachable issues.
- [ ] Document privacy transfer consent, retention jobs and restore/deletion interaction.
- [ ] Review live corridor/fee/FX permissions before enabling flags.
