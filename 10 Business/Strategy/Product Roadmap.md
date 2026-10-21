---
type: note
status: active
date: 2026-10-19
---

# Product Roadmap

> Two-phase plan, locked in [[10 Business/Decisions/locked-decisions|locked decisions]]. This note is the map; the decision log is the source of truth.

## Phase 1: Shopify store (dropshipping)

Sell keycap keychains through **Shopify** with no custom backend.

- **Dropshipping**: orders are fulfilled by the external supplier. No in-house assembly or printing.
- **Customization via variant options**: users pick from set options (e.g. base color, keycap count, switch type) using Shopify's variant picker. No custom text or per-order printing.
- **No QR claim cards yet**: the collectible system arrives with the custom platform in Phase 2.

## Phase 2: Custom platform with collectible QR claim

The full custom platform, specced in [[Backend Feature Roadmap|backend roadmap]]:

- Accounts, catalog, stock, cart, checkout, orders: a complete store backend.
- The **collectible QR claim system** as the differentiator: every keychain ships with a QR card that unlocks a collectible tied to its design.
- Phase 2 decisions (claim rules, stock source of truth, no guest checkout, etc.) live in [[10 Business/Decisions/locked-decisions|locked decisions]].

## Phase 1 → Phase 2

- Phase 1 validates the market and the customization angle with minimal investment.
- Phase 2 is where the moat lives; the QR claim system is only possible on the custom platform.
- Shopify data (products, orders) is a candidate to migrate when Phase 2 launches.
