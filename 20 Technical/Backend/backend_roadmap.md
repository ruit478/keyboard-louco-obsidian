# Keyboard Louco — Backend Feature Roadmap

> Scope: Stock keycap keychains (ordered from supplier) + collectible QR claim system.
> Format: grouped feature map with critical backend notes.

---

## 1. Accounts & Identity

| Feature | Backend notes |
|---|---|
| Account creation | Email + password, or OAuth (Google/Apple). Email verification required before QR claim is allowed. |
| Login / logout / session | JWT or server-side session. Refresh token rotation. |
| Password reset | Time-limited signed token via email. |
| Profile management | Display name, avatar, shipping address(es). |
| Account deletion (GDPR) | Soft-delete + anonymize PII, but **keep order records** for accounting. **Locked:** claimed collectibles are hard-deleted (claim rows + that user's claim audit entries). QR instances stay **claimed/voided** — never revert to unclaimed, or deletion becomes a claim-farming vector. |
| Roles / admin flag | Customer vs. admin. Needed before building any admin panel. |
| Email preferences | Marketing opt-in separate from transactional email. |

**Add early:** account creation must be gated behind email verification, otherwise QR claim abuse is trivial.

---

## 2. Catalog & Products

| Feature | Backend notes |
|---|---|
| Product catalog | Keycap keychain SKUs. Each SKU has: print area dimensions (mm), base color, switch type, price. |
| Product variants | E.g. base color, LED on/off, packaging type. Variant = separate stock unit. |
| Product images / gallery | Multiple images per product. |
| Product categories / tags | For filtering and SEO. |
| Search & filter | By name, category, price range. |
| Product availability | Derived from stock, not a manual flag. |
| SEO fields | Slug, meta title, meta description per product. |

---

## 3. Stock Management

| Feature | Backend notes |
|---|---|
| Stock per SKU / variant | Integer quantity. Supplier ships finished keychains — nothing is assembled or printed on our side. |
| Stock reservation on order | **Critical:** decrement/reserve stock at order placement, not at payment, or you oversell. Release reservation on payment failure/timeout. |
| Low-stock threshold | Admin alert when quantity crosses threshold. |
| Stock adjustments log | Who changed what, when, why (manual correction, restock, order). Audit trail. |
| Backorder / pre-order flag | Optional per SKU. If enabled, stock may go negative — drop the `stock ≥ 0` constraint (see Cross-Cutting Backend Concerns). |
| Supplier / restock tracking | Optional but useful once bulk-ordering from Alibaba. |
| External supplier | Finished keychains are ordered from an external supplier. No printing, no assembly on our side — our DB is the source of truth for stock. |

**Add early:** stock reservation + release logic. This is the #1 source of overselling bugs in small shops.

---

## 4. Collectibles & QR Claim (the core product)

Treat this as its own subsystem.

| Feature | Backend notes |
|---|---|
| Collectible definition | A collectible "type" (design, name, image, rarity, edition size). **Tied 1:1 to the product design** — buying a keychain of design X always ships a QR card that grants collectible X. |
| Collectible instance | A unique QR card: `collectible_instance_id` + `qr_code` (unique, unguessable) + `status` (unclaimed/claimed/void). |
| QR code generation | Generate at packing time — the QR is printed on an insert card and packed with the unit. Each QR encodes a claim URL (`https://<domain>/claim/<token>`) with a **unique, random, high-entropy token** — never a sequential ID. Store token hash, not the raw token, to prevent DB-leak forgery. |
| QR claim endpoint | `POST /claim` with token. Atomic claim via conditional update. Idempotency comes from the token itself — a re-scan by the same user returns the existing result; no client idempotency key needed. |
| Uniqueness per user | **Locked: type-level.** Each instance is claimed by exactly one user, and a user can claim each collectible type at most once. Enforce with `UNIQUE (user_id, collectible_type_id)` plus the atomic claim update. |
| Claim race condition | Two users scanning the same QR simultaneously must not both succeed. Use an atomic conditional update — `UPDATE instances SET status='claimed', claimed_by=? WHERE id=? AND status='unclaimed'` — and check rows affected. (A unique constraint on `(collectible_instance_id, user_id)` does **not** prevent this: a second user has a different `user_id`.) |
| Claim requires auth | Force login + email verification before claim. Otherwise anyone can burn QRs. **Implementation note:** the claim token must survive the signup → email-verification redirect loop — carry a "pending claim" token through the auth flow so the scan isn't lost. |
| Claim rate limiting | Per-user and per-IP limits on the claim endpoint as DoS/abuse defense — token entropy is the real brute-force protection. Keep per-user limits generous enough for a legit buyer scanning many keychains in a row. |
| Collectible gallery | User's claimed collectibles, with claimed date, rarity, "owned" badge. |
| Duplicate handling | **Locked: reject.** If the user already owns that collectible type, the claim is rejected with an "already owned" response. The QR stays **unclaimed** (the physical unit remains claimable — gift/trade mechanic). No points conversion (avoids farming incentives). |
| Void / revoke claim | Admin ability to void a QR (lost, damaged, fraud) and optionally revoke a claim. |
| Claim audit log | Every scan attempt: token **hash**, user, IP, timestamp, result (success/already claimed/invalid/rate-limited). Essential for fraud investigation. |
| QR batch management | Admin generates QR cards per packing run, matched to each order line's collectible design. Batches tie to packing runs, not factory production runs. |
| Collectible transfer between users | Optional. If gifting/trading is allowed, the "unique per user" rule and ownership history get more complex. Defer unless core. |

**Locked:** uniqueness is **type-level** (a user can claim each design at most once). Duplicates are rejected and the QR stays unclaimed — see [[10 Business/Decisions/locked-decisions|locked decisions]].

---

## 5. Cart & Checkout

| Feature | Backend notes |
|---|---|
| Shopping cart | Guest cart + merge on login. Cart stores line items with variant. |
| Cart persistence | Server-side cart keyed by session/user, so it survives device switches. |
| Cart validation | Re-check stock and price at checkout. |
| Checkout | Address, shipping method, payment. |
| Payment integration | Stripe/PayPal. Handle webhooks for async confirmation. |
| Order creation | Atomic: reserve stock, create order. Collectible instances are **not** bound here — they're bound at packing time, when the QR card is generated (see Collectibles & QR Claim). |
| Payment failure handling | Release stock reservation, keep cart intact, notify user. |
| Idempotent checkout | Prevent double-charge on double-click / retry. Idempotency key per checkout attempt, passed to the payment provider on PaymentIntent creation. |
| Shipping calculation | Flat rate / weight-based / zone-based. Keychains are light — keep it simple. |
| Tax handling | Decide: included, or calculated by region. |
| Discount codes / coupons | Optional but high-value for launch. |
| Guest checkout | **Locked: none.** Account required to check out — no guest checkout for now. Keeps the order → account link simple and every claim authenticated. |

---

## 6. Orders

| Feature | Backend notes |
|---|---|
| Order list (customer) | Status, date, total, items, tracking. |
| Order detail | Line items, shipping address, payment status. |
| Order status machine | `pending_payment → paid → in_production → shipped → delivered`, plus separate `payment_failed`, `cancelled`, and `refunded` states — each with its own invariants (`cancelled`: release components, no QRs generated yet; `refunded`: partial/full money back, QR stays claimed). Explicit states, no free-text status. |
| Order → collectible link | Instances are bound to order lines at packing time (QR card generated and packed with the unit). Orders reference the `collectible_instance_id(s)` after fulfillment. |
| Fulfillment / tracking number | Manual entry or carrier API. |
| Cancellation | Customer-initiated (before packing) and admin-initiated. Must release component stock. Before packing: no QRs exist yet — nothing to void. After packing: void the order's bound QR instances. |
| Refunds | Full/partial. Decide what happens to already-claimed collectibles on refund — likely keep them claimed (revoking feels punitive) but document the policy. |
| Order emails | Confirmation, shipped, delivered. Transactional only. |
| Admin order management | Search, filter by status, bulk actions, export. |
| Invoice / receipt | PDF generation. |

**Add early:** the order status machine and the order→collectible link. Retrofitting these later is painful.

---

## 7. Favourites / Wishlist

| Feature | Backend notes |
|---|---|
| Add/remove favourite | Per user, per product. |
| Favourites list page | Grid view, "add to cart" from list. |
| Favourites on collectibles | Not needed as a separate feature — the collectible is tied to the product design, so the product wishlist covers it. The claimed gallery is the "owned" list. |
| Stock/price change notification | Optional: email when a favourited product is back in stock or on sale. |
| Guest favourites | localStorage-based, merged on login. Optional. |

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

| Feature | Backend notes |
|---|---|
| Transactional email | Verification, password reset, order confirmation, shipping, claim success. Use a provider (Resend/SendGrid/Postmark). |
| Email templates | Versioned, testable. |
| Optional: push / in-app notifications | For "collectible claimed" or "back in stock." Defer. |
| Claim confirmation | When a QR is claimed, show + email a confirmation with the collectible art. This is the emotional payoff — don't skip it. |

---

## 10. Cross-Cutting Backend Concerns

| Concern | Why it matters here |
|---|---|
| **File storage** | Product images + collectible art. Use S3/R2 with signed URLs and lifecycle rules. |
| **Image processing** | Product image thumbnails/sizes (sharp/imgproxy at the CDN/edge). No upload or print pipeline — nothing gets printed. |
| **Idempotency** | Checkout: idempotency key on the outbound payment call. Payment webhooks: dedupe on the provider's event ID. QR claim: the token is the natural key — no client key needed. |
| **Rate limiting** | Especially on `/claim` and `/login`. |
| **Audit logging** | Admin actions + claim attempts. |
| **Database constraints over app logic** | `UNIQUE (user_id, collectible_type_id)` for type-level claim uniqueness; `stock ≥ 0` on SKUs **only if backorders are off** (see Stock Management); atomic conditional updates for claim and reservation. |
| **Backups + restore drill** | The claim ledger is irreplaceable business data. Test restore, not just backup. |
| **Observability** | Structured logs, error tracking (Sentry), uptime monitoring, alert on failed claims / failed payments. |
| **Feature flags** | Ship claim and custom-print features behind flags so you can disable without redeploy. |
| **API versioning** | If you ever build a mobile app or partner integration. |
| **Terms / privacy pages** | Required for accounts. State the deletion policy: claimed collectibles are removed when an account is deleted. |
| **Abuse / fraud controls** | QR brute-force protection, duplicate-account detection for claim farming. |
| **Test strategy** | Concurrency tests for claim and stock reservation are non-negotiable. |

---

## Suggested Build Order

1. **Accounts + email verification + admin role** — everything depends on identity.
2. **Product catalog + variants** — the SKU model.
3. **Stock management with reservation/release** — before cart, so cart can validate.
4. **Cart + checkout + orders + payment webhook** — the money path.
5. **Collectible definitions + QR generation + claim endpoint + claim gallery** — the core differentiator; build after the money path works.
6. **Favourites** — engagement, low risk, build anytime after catalog.
7. **Admin back office** — grow incrementally alongside each feature.
8. **Notifications, audit, observability** — layer in continuously, not at the end.

---

## Locked Decisions

Moved to [[10 Business/Decisions/locked-decisions|locked decisions]] — the single source of truth for product-level decisions.