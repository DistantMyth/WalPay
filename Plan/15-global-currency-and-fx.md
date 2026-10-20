# Global USD-primary currency and conversion plan

User direction: global availability where the provider supports it, USD as the primary currency, conversion with disclosed platform charges, and a future platform or P2P conversion experience. Preserve that product direction while making execution conditional on actual payment/FX entitlements.

## Currency concepts that must remain separate

| Concept | Example | Authority |
|---|---|---|
| Catalog currency | Merchant lists EUR40 | Seller's current quote |
| PayPal order/capture currency | Order charges EUR40 | Frozen order and provider capture |
| User funding currency | User's funding account converts USD to EUR | Provider approval/funding flow; not inferred from profile |
| Receiver settlement/holding currency | Merchant receives or holds its supported currency | Provider observed settlement facts/account product |
| Reporting currency | Dashboard estimates all activity in USD | Attributed valuation service, labelled estimate |
| Platform fee currency | Disclosed USD service fee or eligible same-currency platform fee | Contract/product arrangement plus frozen fee policy |

Setting `preferred_currency: USD` changes display defaults, not actual provider account holdings, FX execution or every invoice's denomination. Never present an estimated USD total as the amount PayPal will debit unless a supported provider quote actually fixes it.

PayPal currency support varies by payment type and country, some currencies have zero decimal places, and receiving preferences can leave an unfamiliar-currency payment pending. Maintain product-specific currency metadata and check provider status rather than assuming a universal currency list is enabled. [PayPal currency reference](https://developer.paypal.com/api/codes/currency).

## Four capability levels

1. **Reporting conversion — core.** Convert observed amounts into indicative USD using a dated attributed rate. No money changes hands. Show original currency, rate time/source and estimate flag. Users can filter by original currency and disable estimates.
2. **Provider-managed checkout conversion — eligible flow.** Create the order in a supported transaction currency and let PayPal present eligible funding conversion/fees to the user. Store known facts and provider references. WalPay cannot guarantee a source-currency hard cap from a changing display estimate.
3. **Executable provider FX quote — gated.** Only enable if the selected contracted provider product exposes an executable quote with exact source/target amounts, fees, expiry and supported execution/settlement semantics. Standard Orders/Payouts alone are not assumed to expose a general FX conversion endpoint. Initially the adapter returns `CAPABILITY_UNAVAILABLE` if no such product has been verified.
4. **Peer FX — future gated product.** Match users' conversion preferences only after an approved regulated payment/FX partner and reviewed settlement model support it. Do not create a WalPay escrow wallet, internal credit, netting pool or two unprotected reciprocal transfers as a shortcut.

Core demo: USD checkout plus a EUR/JPY original-currency reporting fixture and USD valuations. Demonstrate a real second-currency sandbox checkout only if that account/currency configuration works. Explicitly label estimated FX and do not claim a sandbox fixed-rate conversion unless measured and supported.

## Rate and amount arithmetic

Store rates as a ratio of positive integer digit strings `numerator/denominator` representing **target major units per source major unit** (e.g. a decimal rate0.92 becomes92/100). Preserve source/provider precision and quote ID. Reject decimal points/exponents/signs in ratio operands before BigInt parsing. Exponents belong to both currencies. To convert source minor units `s` into target minor units:

`target_minor = round(s * numerator * 10^target_exponent / (denominator * 10^source_exponent))`.

Use BigInt and a named rounding policy. Reporting can use half-even; charges use the explicit fee/contract rule, e.g. ceiling to avoid undercollecting a disclosed fee, with tests for small amounts. Never round the exchange rate first and then call it exact. Calculation must not use binary floating point.

```ts
function floorRatio(n: bigint, d: bigint): bigint {
  if (n < 0n || d <= 0n) throw new Error('invalid ratio');
  return n / d;
}
function ceilRatio(n: bigint, d: bigint): bigint {
  if (n < 0n || d <= 0n) throw new Error('invalid ratio');
  return (n + d - 1n) / d;
}
function halfEven(n: bigint, d: bigint): bigint {
  if (n < 0n || d <= 0n) throw new Error('invalid ratio');
  const q = n / d, r = n % d;
  return r * 2n < d ? q : r * 2n > d ? q + 1n : q % 2n === 0n ? q : q + 1n;
}
```

Bound operand sizes and currencies before arithmetic; negative adjustments use separate direction. For refunds, preserve original currency and distinguish provider refund value from current USD reporting estimate. A different exchange rate at refund time may change the user's funding-currency outcome; do not promise original-rate reversal without provider terms.

## Quote and fee contract

Proposed `POST /v1/fx-quotes`:

```json
{
  "source":{"currency":"USD","amount_minor":"10000"},
  "target_currency":"EUR",
  "purpose":"reporting_estimate",
  "fee_policy_id":"0199e000-0000-7000-8000-000000000081"
}
```

```json
{
  "id":"0199e000-0000-7000-8000-000000000082",
  "kind":"estimate",
  "executable":false,
  "source":{"currency":"USD","amount_minor":"10000"},
  "fee":{"currency":"USD","amount_minor":"150"},
  "source_after_fee":{"currency":"USD","amount_minor":"9850"},
  "target_estimate":{"currency":"EUR","amount_minor":"9062"},
  "rate":{"numerator":"92","denominator":"100","direction":"target_major_per_source_major","source":"fixture_demo","as_of":"2026-10-16T12:00:00Z"},
  "fee_disclosure":{"policy_version":1,"basis_points":150,"flat_minor":"0","collection":"illustrative_not_collected"},
  "expires_at":"2026-10-16T12:01:00Z",
  "mode":"simulation"
}
```

This fixture computes USD100 minus USD1.50 = USD98.50 at EUR0.92/USD -> EUR90.62. It is a mathematical example, not a current market rate or fee recommendation. A reporting estimate does not actually deduct a fee; the example explicitly illustrates a prospective fee-bearing conversion. Ordinary dashboard valuations use fee0 and show no fictitious charge. Real fees and quotes come from their approved policy/provider.

For `kind: executable`, require provider quote ID, exact `source_total_debit`, `source_conversion_principal`, `target_gross`, `target_net`, provider fee/spread if disclosed, WalPay fee, fee collection method, eligibility decision, expiry, and provider execution locator. Reject contradictory `executable: true`/estimate fields. Source-fixed and target-fixed quote requests are different schema variants. An unsupported fixed-target guarantee must not be silently approximated.

### Fee collection models

- Own WalPay merchant service: charge an explicitly priced service only under applicable provider terms; define refunds/tax and merchant-of-record responsibilities. Do not disguise a payment surcharge or exchange service as a product.
- Approved multiparty platform fee: use documented partner platform-fee arrangement; 08 records currency/account restrictions. No invented `fx_fee` field in PayPal Orders.
- Partner FX service: partner collects or shares a disclosed fee under contract and returns auditable facts; WalPay records attribution.
- Reporting-only estimate: collect no conversion fee, because no conversion occurred.

Fee policy stores percentage basis points, flat minor amount, minimum/maximum, jurisdiction/product scope, tax treatment, effective date, rounding, refund policy and disclosure version. The server selects the applicable scoped policy; request `fee_policy_id` is only a validated hint and can never choose a cheaper/out-of-scope policy. Compute from the documented basis (gross or conversion principal); never recursively apply a percentage to an undefined total. The final all-in authorization includes fees known to WalPay. Unknown provider funding fees remain explicitly provider-disclosed and cannot be treated as a guaranteed source-currency maximum.

## Budget enforcement across currencies

Preferred MVP: mission cap and final payable quote share the transaction currency. For USD-capped missions, select USD-denominated payable quotes. A listing displayed in EUR may be explored, but no auto-accept occurs merely because its estimated USD value appears under cap.

If actual supported executable FX is enabled, reserve `source_total_debit` exactly once in USD: it already includes all applicable disclosed fees. Validate the equation between conversion principal, fee components and total so no fee is omitted or counted twice. Bind the FX quote to the intent digest and recheck its expiry just before execution. Any change requiring an additional cent invalidates the prior authorization. A separately configured safety buffer can help discovery filtering, but it cannot turn an indicative rate into a guaranteed quote. Quote expiry with uncertain external execution retains reservation until reconciliation.

For reports, never add raw EUR and USD. Return `by_currency` aggregates and optional `reporting_estimate` with coverage/time/source. Missing rates produce partial coverage and unconverted amounts. Historical income uses the configured recognized-at valuation (e.g. provider confirmation time) consistently; current-rate view is a separate labelled projection.

## P2P FX design prerequisites

Requested future capability remains in roadmap with these dependencies:

1. Identify legal entity, countries, personal/business account types, target users and provider-approved peer exchange/transfer product.
2. Obtain explicit provider permission and determine licensing/KYC/AML/sanctions, disclosures, privacy, chargeback and error-resolution responsibilities for each corridor.
3. Use partner-held funds and partner atomic/guaranteed settlement or protected cancellation. One peer's confirmed leg cannot trigger an unsecured promise from WalPay.
4. Bind both parties' approved source/target/fees/expiry and verified receivers. Matching interest is not authorization to send money.
5. Document partial settlement, failure, refund/reversal, dispute, rate movement and liquidity responsibilities before coding execution.
6. Add scoped capabilities only after sandbox/product evidence. Until then allow an explicitly simulated quote/match demonstration or a provider handoff without claiming completed peer exchange.

Do not launch order matching that handles real client funds or net settlement under the assumption that API availability grants permission. Country-specific terms require review; this plan makes no global licensing conclusion.

## FX implementation and tests checklist

- [ ] Version currency exponents and provider product allowlists.
- [ ] Pin a permitted rate source/license; configure rate freshness and missing-rate behavior.
- [ ] Validate rate direction, positive denominator, magnitude and operand bounds.
- [ ] Test USD/EUR, JPY zero decimals and unsupported currencies using exact expected integers.
- [ ] Test half-even ties, fee ceiling, tiny amounts, maximum amount and target-fixed inversion.
- [ ] Test quote expiry, provider requote and fee policy revision invalidating consent.
- [ ] Test source/target currency mismatch at capture and refund reconciliation.
- [ ] Test concurrent FX reservations and unknown execution retaining source budget.
- [ ] Test coverage gaps and history export containing original amounts plus labelled USD estimates.
- [ ] Verify actual fee collection, refund policy and platform account constraints with approved product.
- [ ] Keep P2P FX flag off until all partner/product/legal/settlement prerequisites have named evidence.
