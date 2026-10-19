---
type: decision-log
status: locked
date: 2026-10-19
tags: [locked]
---

# Locked Decisions

> Single source of truth for product-level decisions. Extracted from [[20 Technical/Backend/backend_roadmap|backend roadmap]]. Other notes link here — never restate a decision.

1. **"Unique per user" is type-level** — a user can claim each collectible design at most once. Enforced by `UNIQUE (user_id, collectible_type_id)` plus the atomic conditional claim update.
2. **Duplicate claim → reject.** The QR stays unclaimed, so the physical unit remains claimable by someone else — effectively a gift/trade mechanic. No points conversion (avoids farming incentives).
3. **No custom printing.** Nothing gets printed on the keycaps or by us — finished keychains are ordered from an external supplier. There are no design uploads; the collectible claim system is the product's differentiator.
4. **Stock is finished keychains per variant**, ordered from the supplier. Our DB is the source of truth for stock; no assembly, no POD partner.
5. **QR placement: insert card, printed at packing.** Instances are created and bound to order lines at packing time, not at order creation. QR batches are packing runs.
6. **Guest checkout — none for now.** Account required to check out; revisit later if conversion suffers.
7. **Account deletion removes claimed collectibles.** Claim rows and that user's claim audit entries are deleted. QR instances stay claimed/voided — deletion must not resurrect collectibles (claim farming via account churn).
8. **Collectible is tied to the keychain design.** Buying design X always grants collectible X via its QR card — no randomness. One favourites feature covers products and collectibles.

> Sanity-check on decision 2: a rejected duplicate means the buyer keeps the keychain and the QR stays claimable by whoever they give it to. If that's not desired, the alternative is to void the QR on duplicate scan.
