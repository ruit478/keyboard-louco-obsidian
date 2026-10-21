---
type: note
status: active
date: 2026-10-21
---

# Phase 1 Platform Analysis: Shopify vs Custom App

> Re-opens locked decision #1 ([[10 Business/Decisions/locked-decisions|locked decisions]]). Phase 2 is custom regardless ([[10 Business/Strategy/Product Roadmap|product roadmap]]), so the real question is **when to start the custom build**.

## Recommendation up front

**Build a thin-slice custom app in Phase 1**: Catalog, variants, cart, Stripe checkout, orders, basic admin, and grow it into Phase 2 ([[20 Technical/Backend/Backend Feature Roadmap|backend roadmap]]). The store part of the custom build is Phase 2 work done early: Shopify's Phase 1 work is configuration that gets thrown away (only data migrates).

**Flip to Shopify if** speed to first sale is the binding constraint (custom ≈ 8–12 weeks vs ~1–2 on Shopify), nobody is free to build/maintain the app before Phase 2, or Phase 1's only goal is demand-testing and the customization UX can wait.

## What Phase 1 actually is

- Keycap keychains, customization = **variant options only** (base color, keycap count, switch type).
- Goal per roadmap: validate the market and the customization angle with minimal investment: the QR claim moat only arrives in Phase 2.

## Comparison

| Dimension                | Shopify (Basic, PT)                                                                                        | Custom app (thin slice)                                                         |
| ------------------------ | ---------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------- |
| Time to first sale       | ~1–2 weeks                                                                                                 | ≈ 8–12 weeks solo dev                                                           |
| Fixed cost               | €27/mo (€19/mo annual)                                                                                     | ~€5/mo hosting on railway                                                       |
| Per-sale cost            | 1.8% + €0.30 cards, 1.3% + €0.30 MB WAY; +2% if external gateway                                           | Stripe: 1.5% + €0.25 EEA cards; MB WAY ~1.4% + €0.20                            |
| Customization validation | Native variant picker: 3 options / 100 variants (fits today, but a plain dropdown, no visual configurator) | Real configurator (live keychain preview), validates the actual product promise |
| Dropshipping flow        | Turnkey: order apps / CSV / email forwarding to supplier                                                   | Order export (CSV) to supplier                                                  |
| Checkout control         | Below Plus: branding + post-purchase only                                                                  | Full control: Account-required checkout from day 1                              |
| Payments                 | Shopify Payments (PT since 2022): cards, MB WAY, Multibanco, Apple/Google Pay                              | Stripe + own integration                                                        |
| EU VAT                   | Collected at checkout, B2B VAT validation, tax-inclusive pricing                                           | Stripe Tax or manual; **OSS filing is merchant-side either way**                |
| GDPR                     | Hosted: Shopify DPA                                                                                        | Own hosting, DPAs with processors                                               |
| Carries into Phase 2     | Data only (products/orders/customers via CSV/API)                                                          | Code: catalog, stock, cart, checkout, orders, admin are Phase 2 roadmap items   |
| Migration cost           | Export data + rebuild all store logic in Phase 2                                                           | None: Phase 2 extends it                                                        |
| Platform risk            | Fees/limits change at Shopify's discretion; checkout locked down                                           | Full control; you own the ops burden                                            |
| Ops burden               | Zero: hosted, PCI handled                                                                                  | Hosting, backups, fraud, PCI via Stripe, monitoring                             |

## The real trade-off

- **Shopify buys speed and zero ops, paid in throwaway work.** Everything configured there (catalog structure, checkout flow, order management) is rebuilt in Phase 2, with only the data migrating.
- **Custom Phase 1 builds Phase 2 early.** Accounts, catalog, stock, cart, checkout, orders, admin (roadmap sections 1–3, 5–6, 8) are Phase 2 scope regardless. A Phase 1 thin slice skips QR claim, favourites, and notification polish. Accounts-at-checkout can ship from day 1, matching locked decision #6, and the configurator becomes the real test of the customization angle.
- **Fee math barely matters.** At Phase 1 volumes the Shopify-vs-Stripe delta (≈ 0.3% + €0.05/sale) is noise. <u>The deciding factors are time-to-market, carryover, and what actually gets validated.</u>

## Shopify specifics

- **Variant limits:** 100 variants / 3 options native; our 3 options fit. A 2,048-variant model is rolling out but conditional; don't count on it. Neither fixes the UX: no visual configurator.
- **Checkout:** `checkout.liquid` and Shopify Scripts are dead; Checkout Extensibility is sandboxed, and on non-Plus plans customization is effectively branding + thank-you page. Anything rendered before payment (custom fields, upsells) needs Plus (from $2,300/mo). A Shopify Phase 1 cannot test collectible-adjacent checkout UX at all.
- **Payments PT:** Shopify Payments available (online only): MB WAY + Multibanco built in. This is the strongest single argument for Shopify in Portugal; Multibanco reach is otherwise hard to replicate.

## Custom specifics

- **Fees:** Stripe PT 1.5% + €0.25 standard EEA cards; Multibanco 2.95% + €0.25; MB WAY ~1.4% + €0.20. Stripe Tax computes EU VAT at checkout; OSS reporting stays merchant-side (same on Shopify).
- **Effort (solo dev, thin slice ≈ 8–12 weeks):** product + variants + images, server-side cart, Stripe checkout + webhooks, order emails, stock reservation/release (the roadmap's #1 oversell trap), admin CRUD, hosting + monitoring. Roughly half of the Phase 2 roadmap's non-QR surface.
- **Risk:** you own uptime, fraud, and GDPR. Solvable at this scale, but it is real ongoing work that Shopify simply removes.

## Conditions that flip the recommendation

- **Pick Shopify** if: Phase 1 must launch in weeks (season or cash), nobody is free to build the app, or demand-testing is the sole goal.
- **Pick custom** if: the configurator is the product promise being tested, Phase 2 lands within ~2 quarters, and dev capacity exists now. Building twice costs more in total, and Shopify's locked checkout cannot host the Phase 2 UX even as a test.

**Note:** this is decision support, not a decision. If custom Phase 1 is chosen, decision #1 in [[10 Business/Decisions/locked-decisions|locked decisions]] needs updating there.

## Sources

- [Shopify pricing 2026: Litextension](https://litextension.com/blog/shopify-pricing/) · [Shopify pricing 2026: Sellfy](https://sellfy.com/blog/shopify-pricing/)
- [Shopify variant limits: Litextension](https://litextension.com/blog/shopify-variant-limit/) · [2,048-variant rollout: Webtoffee](https://www.webtoffee.com/blog/shopify-2048-variant-limit/)
- [Shopify Payments Portugal: nexostudio.pt](https://nexostudio.pt/shopify-payments-portugal-mb-way-multibanco-ifthenpay/) · [Shopify plans in Portugal: nexostudio.pt](https://nexostudio.pt/sobre-o-shopify/)
- [Shopify custom checkout limits 2026: bogos.io](https://bogos.io/shopify-custom-checkout/) · [Checkout Extensibility explained: cartylabs](https://cartylabs.com/blog/shopify-checkout-extensibility-explained/)
- [Is Hydrogen worth it: Webtoffee](https://www.webtoffee.com/blog/shopify-hydrogen/) · [When headless makes sense: wecanfly](https://wecanflyagency.com/blog/when-headless-shopify-makes-sense)
- [Stripe fees Portugal: doany.ai](https://doany.ai/tools/stripe-fee-calculator/portugal) · [Stripe Portugal 2026: shelf.pt](https://shelf.pt/blog/stripe-portugal)
