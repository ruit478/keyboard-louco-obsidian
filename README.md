# ⌨️ Keyboard Louco

Stock keycap keychains with a collectible twist: every keychain ships with a QR card that unlocks a collectible tied to its design.

> 📖 This README is a distilled overview. The full feature map and rationale live in [`backend_roadmap.md`](Backend%20Feature%20Roadmap.md).

---

## What is it?

Keyboard Louco sells finished keycap keychains — **ordered from an external supplier**, not assembled or printed in-house. The differentiator is the **collectible QR claim system**:

1. A customer buys a keychain of design X.
2. At packing time, a QR card is generated and packed with the unit.
3. Scanning the QR takes the owner to a claim page; after login (with verified email), the collectible for design X is added to their gallery.

**Collectibles are tied 1:1 to keychain designs** — buying design X always grants collectible X. No randomness, no custom printing.

## How the QR claim works

- Each QR encodes `https://<domain>/claim/<token>` with a **unique, random, high-entropy token**. Only the token **hash** is stored, never the raw token.
- Claims are **atomic conditional updates** (`UPDATE ... WHERE status='unclaimed'`), so two users scanning the same QR simultaneously can't both win.
- **Type-level uniqueness**: a user can claim each collectible design at most once, enforced by `UNIQUE (user_id, collectible_type_id)`.
- **Duplicate claims are rejected** and the QR stays unclaimed — the physical card remains claimable by someone else (gift/trade mechanic). No points conversion.
- Every scan attempt is audited (token hash, user, IP, timestamp, result).

## Feature map

| Area | Highlights |
|---|---|
| **Accounts & Identity** | Email/password or OAuth (Google/Apple), email verification required before claiming, refresh-token rotation, password reset, profile + shipping addresses, GDPR soft-delete (order records kept, claimed collectibles removed), admin roles |
| **Catalog & Products** | SKUs with print area, base color, switch type, price; variants as separate stock units; images, categories, search/filter, SEO fields; availability derived from stock |
| **Stock Management** | Integer stock per SKU/variant, **reservation at order placement with release on failure/timeout** (the #1 overselling bug source), low-stock alerts, audit-trailed adjustments, optional backorders/pre-orders |
| **Collectibles & QR Claim** | The core subsystem — see above. Plus: claim rate limiting, claim confirmation email with collectible art, admin void/revoke, QR batch management per packing run |
| **Cart & Checkout** | Server-side cart (guest cart merges on login), re-validation at checkout, Stripe/PayPal with webhooks, idempotency key per checkout attempt, **no guest checkout** — account required |
| **Orders** | Explicit status machine (`pending_payment → paid → in_production → shipped → delivered` + `payment_failed`/`cancelled`/`refunded`), order→collectible link bound at packing, cancellations void bound QRs, transactional emails, PDF invoices |
| **Favourites / Wishlist** | Per-user product favourites; doubles as the collectible "want list" since collectibles are tied to designs — the claimed gallery is the "owned" list. No separate table |
| **Admin / Back Office** | Separate auth with least-privilege roles, product/stock/order/QR/collectible/user management, audit log, dashboard (orders, revenue, low stock, claim rate, failed claims) |
| **Notifications** | Transactional email via provider (Resend/SendGrid/Postmark), versioned templates; push notifications deferred |

## Cross-cutting backend concerns

- **Database constraints over app logic** — uniqueness and atomicity live in the DB, not the application layer.
- **Idempotency** — checkout key on payment calls, webhook dedupe on provider event IDs; the claim token is its own natural idempotency key.
- **Rate limiting** — especially on `/claim` and `/login` (token entropy is the real brute-force defense).
- **Storage** — S3/R2 with signed URLs for product images and collectible art; image processing at the CDN/edge. No upload or print pipeline.
- **Backups + restore drills** — the claim ledger is irreplaceable business data.
- **Observability** — structured logs, error tracking (Sentry), alerts on failed claims and failed payments.
- **Feature flags** — claim and other features ship behind flags so they can be disabled without redeploy.
- **Concurrency tests are non-negotiable** for claim and stock reservation.

## Suggested build order

1. Accounts + email verification + admin role — everything depends on identity
2. Product catalog + variants — the SKU model
3. Stock management with reservation/release — before cart, so cart can validate
4. Cart + checkout + orders + payment webhook — the money path
5. Collectible definitions + QR generation + claim endpoint + gallery — the differentiator, after the money path works
6. Favourites — low risk, anytime after catalog
7. Admin back office — grow incrementally alongside each feature
8. Notifications, audit, observability — layered in continuously

## Locked decisions

*(2026-10-19 — full rationale in [`10 Business/Decisions/locked-decisions.md`](10%20Business/Decisions/locked-decisions.md).)*

1. **"Unique per user" is type-level** — one claim per user per collectible design.
2. **Duplicate claim → reject**, QR stays unclaimed (gift/trade). No points conversion.
3. **No custom printing** — finished keychains are ordered from an external supplier; the claim system is the product.
4. **Stock is finished keychains per variant**; our DB is the source of truth.
5. **QR placement: insert card, printed at packing** — instances bind to order lines at packing, not order creation.
6. **No guest checkout** for now — account required; revisit if conversion suffers.
7. **Account deletion removes claimed collectibles**; QR instances stay claimed/voided — deletion must not enable claim farming via account churn.
8. **Collectible is tied to the keychain design** — no randomness.

## Vault structure

The repo is an Obsidian vault — [`CLAUDE.md`](CLAUDE.md) defines the vault conventions for humans and AI tools.

```
keyboard-louco-obsidian/
├── README.md                    # project overview — GitHub landing page
├── CLAUDE.md                    # vault conventions — the "contract" for AI tools
│
├── 00 Meta/                     # system stuff — never worked in directly
│   ├── Home.md                  # dashboard / MOC of MOCs
│   ├── Templates/               # note templates (decision, supplier, meeting…)
│   └── Attachments/             # images, PDFs, exports
│
├── 01 Inbox/                    # quick capture — file during weekly review
│
├── 10 Business/                 # NON-TECHNICAL side of Keyboard Louco
│   ├── _Index.md
│   ├── Strategy/                # vision, business model, milestones
│   ├── Decisions/               # locked decisions, decision log
│   ├── Product/                 # catalog strategy, collectible concepts
│   ├── Marketing/               # brand, launch plan, social, SEO
│   ├── Finance/                 # pricing, margins, supplier costs
│   ├── Legal/                   # terms, privacy policy, GDPR notes
│   └── Suppliers/               # Alibaba contacts, quotes, restock cadence
│
├── 20 Technical/                # TECHNICAL side
│   ├── _Index.md
│   ├── Backend/                 # API + DB + QR claim system (backend_roadmap.md lives here)
│   ├── Frontend/                # web app, claim page
│   ├── Infrastructure/          # hosting, S3/R2, email provider, observability
│   ├── Design/                  # UI/UX, collectible art specs
│   └── Runbooks/                # ops procedures, restore drills
│
└── 90 Archive/                  # completed/abandoned work — fills itself over time
```

- Each folder has an `_Index.md` map of content; `00 Meta/Home.md` is the vault dashboard.
- Cross-links do the organizing — locked decisions live in `10 Business/Decisions/` and are linked from technical and business notes alike.
