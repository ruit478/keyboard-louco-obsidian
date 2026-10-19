---
type: decision-log
status: locked
date: 2026-10-19
tags: [locked]
---

# Locked Decisions

> Single source of truth for product-level decisions. Extracted from [[Backend Feature Roadmap|backend roadmap]]. Other notes link here — never restate a decision.

1. **"Unique per user" is type-level**: A user can claim each collectible design at most once. Enforced by `UNIQUE (user_id, collectible_type_id)` plus the atomic conditional claim update.
2. **Duplicate claim → reject.** The QR stays unclaimed, so the physical unit remains claimable by someone else — effectively a gift/trade mechanic. No points conversion (avoids farming incentives).
3. **Stock is finished keychains per variant**, ordered from the supplier. Our DB is the source of truth for stock; no assembly, no POD partner.
4. **QR placement: insert card, printed at packing.** Instances are created and bound to order lines at packing time, not at order creation. QR batches are packing runs.
5. **Guest checkout is future work.** Account required to check out; revisit later if conversion suffers.
6. **Account deletion removes claimed collectibles.** Claim rows and that user's claim audit entries are deleted. QR instances stay claimed/voided.
7. **Collectible is tied to the keychain design.** Buying design X always grants collectible X via its QR card