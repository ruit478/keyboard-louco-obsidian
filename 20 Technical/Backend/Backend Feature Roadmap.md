---
type: note
status: active
date: 2026-10-19
---

> Scope: **Phase 2 — custom platform**: stock keycap keychains (ordered from supplier) + collectible QR claim system. Phase 1 runs on Shopify with no custom backend ([[10 Business/Strategy/Product Roadmap|product roadmap]]).
> Format: grouped feature map with critical backend notes.

---

## 1. Accounts & Identity

| Feature                  | Backend notes                                                                                      |
| ------------------------ | -------------------------------------------------------------------------------------------------- |
| Account creation         | Email + password, or OAuth (Google/Apple). Email verification required before QR claim is allowed. |
| Login / logout / session | JWT or server-side session. Refresh token rotation.                                                |
| Password reset           | Time-limited signed token via email.                                                               |
| Profile management       | Display name, avatar, shipping address(es).                                                        |
| Account deletion (GDPR)  | Soft-delete + anonymize PII, but **keep order records** for accounting.                            |
| Roles / admin flag       | Customer vs. admin. Needed before building any admin panel.                                        |
| Email preferences        | Marketing opt-in separate from transactional email.                                                |

**Add early:** account creation must be gated behind email verification, otherwise QR claim abuse is trivial.

---

## 2. Catalog & Products

| Feature                   | Backend notes                                                      |
| ------------------------- | ------------------------------------------------------------------ |
| Product catalog           | Being able to list the several products we have available          |
| Product variants          | For a specific product, list its variants (e.g. number of keycaps) |
| Product images / gallery  | For a specific product, list its images                            |
| Product categories / tags | Associate metadata to the products                                 |
| Search & filter           | Allow filtering products                                           |
| Product availability      | Derived from stock, query availability in real time                |

---

## 3. Stock Management

| Feature                     | Backend notes                                                                                                                                |
| --------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------- |
| Stock per variant crud      | Simple crud to manipulate stock                                                                                                              |
| Stock reservation on order  | **Critical:** _decrement/reserve stock at order placement_, not at payment, or you oversell. Release reservation on payment failure/timeout. |
| Low-stock threshold         | Admin alert when quantity crosses threshold.                                                                                                 |
| Stock adjustments log       | Who changed what, when, why (manual correction, restock, order). Audit trail.                                                                |
| Supplier / restock tracking | Optional but useful once bulk-ordering from Alibaba. Not automatic restock, but maybe as stock metadata                                      |

**Add early:** stock reservation + release logic. This is the #1 source of overselling bugs in small shops.

---

## 4. Collectibles & QR Claim (the core product)

Treat this as its own subsystem.

| Feature                            | Backend notes                                                                                                                                                                                                                                                                                                                                              |
| ---------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Collectible definition             | A collectible "type" (design, name, image, rarity, edition size). **Tied 1:1 to the product design** (buying a keychain of design X always ships a QR card that grants collectible X)                                                                                                                                                                      |
| Collectible instance               | A unique QR card: `collectible_instance_id` + `qr_code` (unique) + `status` (unclaimed/claimed/void).                                                                                                                                                                                                                                                      |
| QR code generation                 | The QR is printed on an insert card and packed with the unit. Each QR encodes a claim URL (`https://<domain>/claim/<token>`) with a **unique, random, high-entropy token** — never a sequential ID. Store token hash, not the raw token, to prevent DB-leak forgery.                                                                                       |
| QR claim endpoint                  | `POST /claim` with token. Atomic claim via conditional update. Idempotency comes from the token itself (A re-scan by the same user returns the existing result)                                                                                                                                                                                            |
| Uniqueness per user                | **Locked: type-level.** Each instance is claimed by exactly one user, and a user can claim each collectible type at most once. Enforce with `UNIQUE (user_id, collectible_type_id)` plus the atomic claim update.                                                                                                                                          |
| Claim race condition               | Two users scanning the same QR simultaneously must not both succeed. Use an atomic conditional update — `UPDATE instances SET status='claimed', claimed_by=? WHERE id=? AND status='unclaimed'` and check rows affected. (A unique constraint on `(collectible_instance_id, user_id)` does **not** prevent this: a second user has a different `user_id`.) |
| Claim requires auth                | Force login + email verification before claim. Otherwise anyone can burn QRs.                                                                                                                                                                                                                                                                              |
| Collectible gallery                | User's claimed collectibles, with claimed date, rarity, "owned" badge.                                                                                                                                                                                                                                                                                     |
| Duplicate handling                 | **Locked: reject.** If the user already owns that collectible type, the claim is rejected with an "already owned" response. The QR stays **unclaimed** (the physical unit remains claimable, opening the possibility for a future gift/trade mechanic). No points conversion (avoids farming incentives).                                                  |
| Void / revoke claim                | Admin ability to void a QR (lost, damaged, fraud) and optionally revoke a claim.                                                                                                                                                                                                                                                                           |
| Claim audit log                    | Every scan attempt: token **hash**, user, IP, timestamp, result (success/already claimed/invalid/rate-limited). Essential for fraud investigation.                                                                                                                                                                                                         |
| QR batch management                | Admin generates QR cards per packing run, matched to each order line's collectible design. Batches tie to packing runs, not factory production runs.                                                                                                                                                                                                       |
| Collectible transfer between users | Optional. If gifting/trading is allowed, the "unique per user" rule and ownership history get more complex. Defer unless core.                                                                                                                                                                                                                             |

---

## 5. Cart & Checkout

| Feature                  | Backend notes                                                                                                                                     |
| ------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------- |
| Shopping cart            | Guest cart + merge on login. Cart stores line items with variant.                                                                                 |
| Cart persistence         | Server-side cart keyed by session/user, so it survives device switches.                                                                           |
| Cart validation          | Re-check stock and price at checkout.                                                                                                             |
| Checkout                 | Address, shipping method, payment.                                                                                                                |
| Payment integration      | Provider TBD (e.g. Stripe/PayPal). Handle webhooks for async confirmation.                                                                        |
| Order creation           | Atomic: reserve stock, create order. Collectible instances are **bound** when the QR card is generated (see Collectibles & QR Claim).             |
| Payment failure handling | Release stock reservation, keep cart intact, notify user.                                                                                         |
| Idempotent checkout      | Prevent double-charge on double-click / retry.  Idempotency key per checkout attempt, passed to the payment provider on PaymentIntent creation.   |
| Shipping calculation     | Flat rate / weight-based / zone-based. Keychains are light — keep it simple.                                                                      |
| Discount codes / coupons | Optional but high-value for launch.                                                                                                               |
| Guest checkout           | **Locked: none.** Account required to check out — no guest checkout for now. Keeps the order → account link simple and every claim authenticated. |

---

## 6. Orders

| Feature                       | Backend notes                                                                                                                                                                                                                                                                                                          |
| ----------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Order list (customer)         | Status, date, total, items, tracking.                                                                                                                                                                                                                                                                                  |
| Order detail                  | Line items, shipping address, payment status.                                                                                                                                                                                                                                                                          |
| Order status machine          | `pending_payment → paid → in_production → shipped → delivered`, plus separate `payment_failed`, `cancelled`, and `refunded` states, each with its own invariants (`cancelled`: release components, no QRs generated yet; `refunded`: partial/full money back, QR stays claimed). Explicit states, no free-text status. |
| Order → collectible link      | Instances are bound to order lines at packing time (QR card generated and packed with the unit). Orders reference the `collectible_instance_id(s)` after fulfillment.                                                                                                                                                  |
| Fulfillment / tracking number | Manual entry or carrier API.                                                                                                                                                                                                                                                                                           |
| Cancellation                  | Customer-initiated (before packing) and admin-initiated. Must release component stock. Before packing: no QRs exist yet — nothing to void. After packing: void the order's bound QR instances.                                                                                                                         |
| Refunds                       | Full/partial. Decide what happens to already-claimed collectibles on refund — likely keep them claimed (revoking feels punitive) but document the policy.                                                                                                                                                              |
| Order emails                  | Confirmation, shipped, delivered. Transactional only.                                                                                                                                                                                                                                                                  |
| Admin order management        | Search, filter by status, bulk actions, export.                                                                                                                                                                                                                                                                        |
| Invoice / receipt             | PDF generation.                                                                                                                                                                                                                                                                                                        |

**Add early:** the order status machine and the order→collectible link. Retrofitting these later is painful.

---

## 7. Favourites / Wishlist

| Feature              | Backend notes                                  |
| -------------------- | ---------------------------------------------- |
| Add/remove favourite | Per user, per product.                         |
| Favourites list page | Grid view, "add to cart" from list.            |


**Resolved:** favourites = products (wishlist). Since each collectible is tied to its product design, the wishlist doubles as the want list; the claimed gallery is the "owned" list. No separate table.

---

## 8. Admin / Back Office

| Feature | Backend notes |
|---|---|
| Admin auth + roles | Separate from customer auth. Least-privilege roles (support, ops, admin). |
| Product management | CRUD products, variants, images. |
| Stock management UI | Adjust stock, view log. |
| Order management UI | View, update status, add tracking. |
| QR batch management | Generate, export, void. |
| Collectible management | CRUD collectible types, view claim stats. |
| User management | View accounts, reset, ban, view claimed collectibles. |
| Audit log | Every admin mutation logged. |
| Dashboard | Orders today, revenue, low stock, claim rate, failed claims. |

---

## 9. Notifications & Comms

| Feature                               | Backend notes                                                                                                             |
| ------------------------------------- | ------------------------------------------------------------------------------------------------------------------------- |
| Transactional email                   | Verification, password reset, order confirmation, shipping, claim success. Use a provider (Resend/SendGrid/Postmark).     |
| Email templates                       | Versioned, testable.                                                                                                      |
| Claim confirmation                    | When a QR is claimed, show + email a confirmation with the collectible art. This is the emotional payoff — don't skip it. |

---

## 10. Cross-Cutting Backend Concerns

| Concern                    | Why it matters here                                                                                                                                                         |
| -------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **File storage**           | Product images + collectible art. Use S3/R2 with signed URLs and lifecycle rules.                                                                                           |
| **Image processing**       | Product image thumbnails/sizes (sharp/imgproxy at the CDN/edge). No upload or print pipeline — nothing gets printed.                                                        |
| **Idempotency**            | Checkout: idempotency key on the outbound payment call. Payment webhooks: dedupe on the provider's event ID. QR claim: the token is the natural key — no client key needed. |
| **Rate limiting**          | Especially on `/claim` and `/login`.                                                                                                                                        |
| **Audit logging**          | Admin actions + claim attempts.                                                                                                                                             |
| **Observability**          | Structured logs, error tracking (Sentry), uptime monitoring, alert on failed claims / failed payments.                                                                      |
| **API versioning**         | If you ever build a mobile app or partner integration.                                                                                                                      |
| **Abuse / fraud controls** | QR brute-force protection, duplicate-account detection for claim farming.                                                                                                   |
| **Test strategy**          | Concurrency tests for claim and stock reservation are non-negotiable.                                                                                                       |

---

## Phase 2 Build Order

1. **Accounts + email verification + admin role**
2. **Product catalog + variants**
3. **Stock management with reservation/release**
4. **Cart + checkout + orders + payment webhook** 
5. **Collectible definitions + QR generation + claim endpoint + claim gallery**
6. **Favourites**
7. **Admin back office**
8. **Notifications, audit, observability**

---

## Locked Decisions

Moved to [[10 Business/Decisions/locked-decisions|locked decisions]] — the single source of truth for product-level decisions.