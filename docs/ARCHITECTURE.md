# POS System — Architecture

> **This is a living document.** Everything here is a starting proposal, not a final decision.
> We expect to change it as we learn more. When something changes, update this file in the
> same pull request as the code change, and add a line to the [Decision log](#decision-log).

- **Last updated:** 2026-11-17
- **Status legend:** ✅ Decided · 🟡 Proposed (default unless someone objects) · ❓ Open (needs a decision)

---

## 1. Goals and constraints

| Constraint | Meaning |
|---|---|
| **Zero budget** | Everything we use must be free: free/open-source tools, GitHub, local machines. No paid hosting, paid APIs or paid services. |
| **Local first** | We build and run everything on our own PCs. Deployment is a later concern. |
| **PC first** | The primary screen is a desktop/laptop browser. Tablet and phone layouts come later. |
| **Two-person team** | Keep the stack small and boring so both of us can work on any part. |
| **Money must be correct** | A POS that miscalculates or loses a sale is useless. Correctness beats features. |

**Who is the real user?** ❓ Not decided yet (café, clothing shop, grocery, etc.). The design below is
generic on purpose. Once chosen, it affects: variants (size/colour), weighted items, tables/tips,
barcodes. See [Open questions](#10-open-questions).

---

## 2. Tech stack

| Layer | Choice | Status | Notes |
|---|---|---|---|
| Backend framework | **Laravel** (latest stable at project creation) | ✅ | PHP 8.3+ |
| Database | **PostgreSQL** | ✅ | Free, strong with transactions and money data |
| Frontend | **Inertia.js + Vue 3 + Tailwind CSS** | 🟡 | Ships with Laravel's starter kits. Gives a fast, app-like cashier screen without building a separate API/SPA. Alternatives: Inertia + React, or Livewire (less JavaScript, but a snappy cart screen is harder). |
| Auth | **Laravel starter-kit auth** (Breeze/Fortify) | 🟡 | Email + password, sessions |
| Roles/permissions | Simple `role` column + Laravel **Policies** | 🟡 | Upgrade to `spatie/laravel-permission` only if we need fine-grained rules |
| Testing | **Pest** (or PHPUnit) | 🟡 | Tests run against a real PostgreSQL database |
| Code style | **Laravel Pint** | 🟡 | Free, one command |
| CI | **GitHub Actions** | 🟡 | Free for public repositories (this repo is public) |
| Task tracking | **GitHub Issues + Milestones + Projects** | 🟡 | Free; one milestone per roadmap phase |
| Hosting | None for now | ✅ | Runs on localhost. See [Deployment](#9-deployment-later) |

### Local development environment (Windows, all free)

Install once on each machine:

1. **PHP 8.3+ and Composer** (Laravel's official Windows installer at `php.new`, or Laragon)
2. **PostgreSQL** (official installer, includes pgAdmin)
3. **Node.js LTS** (for Vite, Vue and Tailwind)
4. **Git** (already installed)

Docker/Laravel Sail is an alternative, but native installs are simpler on Windows 10 Home.
Each of us uses our **own local database**. We share *schema* through migrations and *sample data*
through seeders, never by sharing a database file.

---

## 3. High-level architecture

```
┌─────────────────────────────────────────────┐
│  Browser (PC)                               │
│  Vue 3 pages rendered through Inertia       │
│  - Cashier / checkout screen                │
│  - Back-office (products, stock, reports)   │
└──────────────────┬──────────────────────────┘
                   │ HTTPS/HTTP (localhost for now)
┌──────────────────▼──────────────────────────┐
│  Laravel application                        │
│  Routes → Controllers → Form Requests       │
│        → Actions (business logic)           │
│        → Eloquent models                    │
│  Policies (who may do what)                 │
└──────────────────┬──────────────────────────┘
                   │
┌──────────────────▼──────────────────────────┐
│  PostgreSQL                                 │
│  products, sales, payments, stock, users…   │
└─────────────────────────────────────────────┘
```

A single Laravel app serves both the cashier screen and the back-office. No separate API server
at the start. If we later need a mobile app or an offline client, we add a JSON API on top of the
same Actions.

### Where business logic lives

Controllers stay thin. Rules that matter (completing a sale, refunding, adjusting stock) live in
**Action classes** so they are reusable and easy to test:

```
app/
  Actions/
    Sales/CompleteSale.php
    Sales/VoidSale.php
    Sales/RefundSale.php
    Stock/AdjustStock.php
    Shifts/OpenShift.php
    Shifts/CloseShift.php
  Models/
  Policies/
  Support/Money.php          # helpers for cents <-> display
resources/js/Pages/          # Vue pages (Checkout, Products, Reports, ...)
database/migrations/
database/seeders/            # sample products, demo users
tests/Feature/
docs/
```

---

## 4. Core rules (non-negotiable)

1. **Money is stored as integer cents** (`price_cents`, `total_cents`). Never floats. Display
   formatting happens only at the edge.
2. **Sales are never edited or deleted.** A mistake is corrected by a **void** or a **refund**, each
   recorded as its own transaction with a user and a reason.
3. **Snapshot prices on the sale.** `sale_items` copies the product name and unit price at the time
   of sale. Changing a product's price later must not change past receipts.
4. **Stock is a ledger.** Every change goes through a `stock_movements` row (sale, refund,
   purchase, adjustment). `products.stock_qty` is a cached total that is updated in the same
   database transaction as the movement.
5. **A sale is all-or-nothing.** Creating the sale, its items, its payments and its stock movements
   happen in **one database transaction**. If any step fails, nothing is saved.
6. **Prevent double submission.** The checkout request carries a unique key so a double-click or a
   retry cannot create two sales.
7. **Every sensitive action is attributable.** Who did it and when (voids, refunds, price changes,
   stock adjustments, discounts above a limit).
8. **No secrets in Git.** This repository is public. `.env`, keys and real customer data are never
   committed.

---

## 5. Data model (first draft)

All tables have `id`, `created_at`, `updated_at` unless noted. Amounts are integer cents.

### Users and settings
- **users**: `name`, `email` (unique), `password`, `role` (`admin` | `manager` | `cashier`), `is_active`
- **settings**: `key`, `value` (shop name, currency, receipt footer, default tax mode)

### Catalog
- **categories**: `name`, `parent_id` (nullable)
- **tax_rates**: `name`, `rate_bps` (basis points, 1500 = 15.00%), `is_active`
- **units**: `code` (`pcs`, `kg`, `g`, `l`, `dozen`), `name`, `allows_decimal` (bool)
- **products** (the parent item): `category_id`, `name`, `name_ur` (nullable), `unit_id`,
  `tax_rate_id` (nullable), `track_stock` (bool), `is_active`, soft deletes
- **product_variants** (the thing actually sold and counted; every product has at least one,
  a simple product has a single "Default" variant): `product_id`, `name`, `attributes` (json),
  `sku` (unique), `barcode` (unique, nullable), `price_cents` (per unit), `cost_cents`,
  `stock_qty` numeric(12,3), `reorder_level` numeric(12,3), `is_active`

### Stock
- **stock_movements**: `variant_id`, `type` (`sale` | `refund` | `void` | `purchase` | `adjustment` | `initial`),
  `quantity` numeric(12,3) (signed: negative = stock out), `reference_type` + `reference_id`
  (e.g. the sale), `user_id`, `note`

### Selling
- **shifts**: `user_id`, `opened_at`, `closed_at`, `opening_cash_cents`, `expected_cash_cents`,
  `counted_cash_cents`, `note`
- **sales**: `receipt_no` (unique, sequential), `shift_id`, `user_id`, `customer_id` (nullable),
  `status` (`completed` | `voided`), `subtotal_cents`, `discount_cents`, `tax_cents`,
  `total_cents`, `idempotency_key` (unique), `voided_by`, `void_reason`
- **sale_items**: `sale_id`, `variant_id`, `product_name` + `variant_name` + `unit_code` (snapshots),
  `unit_price_cents` (snapshot), `quantity` numeric(12,3), `discount_cents`, `tax_cents`,
  `line_total_cents`
- **payments**: `sale_id`, `method` (`cash` | `card` | `bank_transfer` | `wallet` | `other`),
  `amount_cents`, `tendered_cents`, `change_cents`, `reference` (nullable)
- **refunds**: `sale_id`, `user_id`, `reason`, `total_cents`
- **refund_items**: `refund_id`, `sale_item_id`, `quantity`, `amount_cents`

### Later phases
- **customers**: `name`, `phone`, `email`, `loyalty_points`
- **suppliers**, **purchase_orders** (receiving stock)
- **audit_logs**: `user_id`, `action`, `subject_type`, `subject_id`, `before`, `after`

> ✅ Weighted items and variants are needed (decided 2026-11-17), so quantities are decimals and
> price/stock/barcode live on the variant. Pakistan (PKR) is the likely target: money is stored as
> whole paisa, and a cash-rounding rule (nearest rupee) is a settings option.

---

## 6. Key flows

### Checkout (the most important flow)

1. Cashier adds items (scan barcode, search, or tap). The cart lives in the browser until payment.
2. Server recalculates all totals itself. **The browser's numbers are never trusted.**
3. On "Pay", the server runs `CompleteSale` inside one transaction:
   1. Lock the product rows being sold (`lockForUpdate`) to prevent overselling.
   2. Check stock rules (block or warn if stock would go negative, depending on setting).
   3. Create `sale`, `sale_items` (with price snapshots), `payments`.
   4. Create negative `stock_movements` and update `products.stock_qty`.
   5. Assign the next `receipt_no`.
4. Show the receipt and open a new empty cart.

### Void
Manager-only, same shift or same day only (rule is configurable). Marks the sale `voided`, reverses
stock through new movements, records who and why.

### Refund
Creates `refund` + `refund_items`, returns stock through new movements, records who and why.
Refunds are partial-capable (some items of a sale).

### Shift
Cashier opens a shift with an opening cash count. At close, the system computes expected cash
(opening + cash sales − cash refunds), the cashier enters the counted cash, and the difference is
stored.

---

## 7. Roles and permissions (first draft)

| Action | Cashier | Manager | Admin |
|---|:--:|:--:|:--:|
| Make a sale, print receipt | ✅ | ✅ | ✅ |
| Open/close own shift | ✅ | ✅ | ✅ |
| Apply small discount | ✅ | ✅ | ✅ |
| Apply large discount (above limit) | ❌ | ✅ | ✅ |
| Void a sale / refund | ❌ | ✅ | ✅ |
| Manage products and prices | ❌ | ✅ | ✅ |
| Adjust stock | ❌ | ✅ | ✅ |
| View reports | own shift | ✅ | ✅ |
| Manage users and settings | ❌ | ❌ | ✅ |

---

## 8. Hardware (PC-first, free approach)

- **Barcode scanner:** most USB scanners behave like a keyboard, so they work in the browser with
  no special software. The checkout screen just needs a focused input that accepts scans.
- **Receipts:** start with a **print-friendly receipt page** (browser print dialog, CSS sized for
  80 mm paper). Direct thermal-printer commands (ESC/POS) come later if needed.
- **Cash drawer / card terminal:** out of scope until a real shop is chosen. Card terminals usually
  cost money and have their own fees, which conflicts with our zero-budget rule, so card payments
  are initially recorded manually as a payment method ("card", with a reference number).

---

## 9. Deployment (later)

For now: **runs on localhost.** Nothing to host, nothing to pay for.

When we want a real shop to use it, the free-or-nearly-free options are:
- Run it on **one PC in the shop** (Laravel + PostgreSQL installed locally; other devices on the
  same network open it in a browser). This costs nothing and works without internet.
- Free hosting tiers exist but are limited and change often. Check current limits before relying
  on one, and never put real customer data on a service we haven't reviewed.

A real shop needs **automatic database backups** (a scheduled `pg_dump` to another drive). This is a
requirement before any real use, not a nice-to-have.

---

## 10. Open questions

1. ❓ **Target business type** (café, clothing, grocery, pharmacy…). Not decided. Drives tables/tips, scales, expiry dates.
2. 🟡 **Country/currency**: Pakistan (PKR) is likely but not confirmed. Tax: ❓ inclusive or exclusive prices, one rate or several?
   Verify current rates with an accountant. Also check whether the chosen business must integrate with FBR.
3. ❓ **One store or multiple branches?** Not decided. Proposal: single store for v1. Adding `branch_id`
   later is possible but touches many tables.
4. ❓ **Frontend choice**: not decided. Inertia + Vue (recommended), Inertia + React, or Livewire.
5. ❓ **Language**: English only, or Urdu names and bilingual receipts?
6. ❓ **Customer credit (udhaar)**: will the shop sell on credit? Decide once the shop type is known.

See also the single-page version of this plan: [pos-plan.html](pos-plan.html).

---

## 11. Team workflow

- `main` is protected (changes only through pull requests). `develop` is the working branch.
- Work on a branch named `feature/<short-name>` or `docs/<short-name>`, branched from `develop`.
- Open a pull request into `develop`. The other person reviews before merging.
- Keep PRs small. One feature or fix per PR.
- Run tests and Pint before opening a PR (CI will check too).
- Plan work as GitHub **Issues** grouped into **Milestones** matching the [roadmap](ROADMAP.md) phases.

---

## Decision log

| Date | Decision | Who |
|---|---|---|
| 2026-11-17 | Laravel + PostgreSQL, web app, PC first | Team |
| 2026-11-17 | Zero-budget: GitHub + local environment only | Team |
| 2026-11-17 | Block sales when stock is zero | Team |
| 2026-11-17 | Weighted items and variants are needed (units, product_variants, decimal quantities added) | Team |
| 2026-11-17 | Likely target: Pakistan (PKR), not confirmed | Team |
| 2026-11-17 | Initial draft of this document | Proposed, pending review |
