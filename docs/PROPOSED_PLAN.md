# POS System: Proposed Plan

- **Date:** 2026-11-27
- **Status:** proposal for review by Murslin and Hashim. Nothing is built yet.
- **Relation to the other documents:** [ARCHITECTURE.md](ARCHITECTURE.md), [ROADMAP.md](ROADMAP.md)
  and [pos-plan.html](pos-plan.html) are unchanged. Where this file differs from them, this file
  describes the change we propose. Section 13 lists the differences.

**Labels:** **Decided** = chosen by Hashim, still needs Murslin's agreement. **Proposed** = default
unless someone objects. **Open** = needs an answer.

---

## 1. Summary

A point of sale and stock system that several shops can use (multi-tenant), where each shop can
later have several branches. We build the backend first. The first release serves one shop with one
branch: register products and vendors, receive stock with a GRN, sell for cash, close a shift and
see sales reports.

## 2. Decisions

| Topic | Choice | Label |
|---|---|---|
| Backend and database | Laravel + PostgreSQL | Decided |
| Build order | Backend first | Decided |
| Frontend (after the backend) | Inertia + React | Decided |
| Tenancy | Multi-tenant, branches in the design from the start | Decided |
| First release scope | One shop, one branch | Decided |
| Languages | English, Roman Urdu, Urdu | Decided |
| Pack selling | Sell a full pack, part of a pack or a single piece, each with its own price | Decided |
| Catalog | Registered categories; every product has a code | Decided |
| Purchasing | Registered vendors; stock arrives through a GRN | Decided |
| Reports | Sales by day, week and month | Decided |
| Customer credit (udhaar) | Full credit and debit ledger | Decided |
| Owner portal | Separate portal on its own domain for us, the product owner | Decided |
| Tax | Not now; planned later | Decided |
| Zero-stock sales | Blocked | Decided |
| FBR integration | Later, together with tax | Proposed |
| Stock costing | Weighted average cost | Proposed |

## 3. Scope

### First release (MVP)
- Shops, branches, users and roles
- Categories, products, variants, pack sizes, barcodes
- Vendors and GRN
- Stock per branch with a stock ledger and manual adjustment
- Shifts, cash in and cash out
- Cash sales with line discounts, receipt numbers
- Sales reports by day, week and month

### Next release
- Void and refund
- Vendor return
- Customer credit (udhaar)
- Order-level discount
- Other payment methods (card, bank transfer, wallet) and split payment
- Owner portal

### Later
- Audit log screen and audit reports
- Low-stock reminders
- Urdu and Roman Urdu screens and receipts
- Tax and FBR
- Several branches per shop in daily use, stock transfer between branches
- Purchase orders, expiry dates and batches

The first release still records who did each sensitive action and when. Only the audit screens
wait.

## 4. Architecture

### Multi-tenancy and branches (Proposed design)
- One application and one database. Every business table has `tenant_id`.
- Stock, shifts, sales and GRNs also have `branch_id`.
- A global query scope limits every query to the current tenant.
- Unique rules are per tenant, for example product code and barcode.
- Automated tests prove that one shop cannot read or change another shop's data.

### Backend first
Business rules live in Action classes (for example `PostGoodsReceipt`, `CompleteSale`,
`CloseShift`). Each phase is verified with feature tests against PostgreSQL and with seeded demo
data. The React screens call the same Actions through thin controllers.

### Owner portal
The portal records which shops use the product and what each one owes us. We enter payments by
hand. It takes no online payment. It uses its own user table and login, separate from shop users.
Locally it runs on a separate host name such as `owner.localhost`.

### Languages
All screen and receipt text goes through Laravel translation files from the first day
(`en`, `ur-Latn`, `ur`). Products can store an Urdu name. Urdu also needs a right-to-left layout.

### Cost risk
Development stays free and local. A multi-shop product with its own domain needs hosting and a
domain before real clients can use it, and both usually cost money. This conflicts with the
zero-budget rule and needs a decision before the first real client.

## 5. Roles and permissions

| Role | Scope | Release |
|---|---|---|
| Platform owner (us) | Shops and their dues | Next (owner portal) |
| Shop owner | Everything in their own shop | MVP |
| Manager | Their branch: products, stock, GRN, reports | MVP |
| Cashier (any number) | Selling and their own shift | MVP |
| Stock keeper, accountant | Stock only, or reports only | Later |

| Action | Cashier | Manager | Shop owner |
|---|:--:|:--:|:--:|
| Make a sale, print a receipt | Yes | Yes | Yes |
| Open and close own shift | Yes | Yes | Yes |
| Discount up to the limit | Yes | Yes | Yes |
| Discount above the limit | No | Yes | Yes |
| Categories, products, prices, vendors | No | Yes | Yes |
| GRN and stock adjustment | No | Yes | Yes |
| View cost and profit | No | Yes | Yes |
| View reports | Own shift | Own branch | All branches |
| Manage users and settings | No | No | Yes |
| Void and refund (next release) | No | Yes | Yes |

Each cashier has their own login and shift. There is no public user registration. We create each
new shop and its owner account (Proposed).

## 6. Catalog

- **Category:** registered once, then selected on products. Sub-categories are optional.
- **Product:** a code that is unique within the shop (generated automatically, can be changed),
  name, optional Urdu name, category.
- **Variant:** the thing that is stocked. It has a base unit (piece, kg, litre). A simple product
  has one "Default" variant.
- **Pack size:** a way of selling a variant, with its own price and barcode.

Stock is always counted in the base unit.

| Sell as | Base quantity | Price (example) |
|---|---|---|
| 1 pencil | 1 | Rs 15 |
| Half pack | 12 | Rs 170 |
| Box | 24 | Rs 320 |

Selling one box from 100 pencils leaves 76 pencils.

The base unit sits on the variant, so a 1 L bottle and a 5 L tin of oil are two variants counted in
pieces. Loose oil is a variant counted in litres.

## 7. Purchasing

### Vendors
Code, name, phone, address.

### Goods Received Note (GRN)
1. Select the vendor and branch.
2. Enter the vendor invoice number and date.
3. Add lines: product, pack size, quantity, cost.
4. Save as a draft and check it.
5. Post it.

Posting happens in one transaction: GRN number assigned, stock increased, stock ledger rows
written, average cost updated. A posted GRN is never edited or deleted.

### Weighted average cost

| | Quantity | Cost per piece |
|---|---|---|
| Existing stock | 100 | Rs 10.00 |
| New GRN | 48 | Rs 12.00 |
| Result | 148 | Rs 10.65 |

Average and unit costs are stored with four decimals. If existing stock is zero, the average becomes
the new GRN cost.

## 8. Selling

### Shifts
A cashier opens a shift with a cash count. Only one open shift per cashier. Every sale belongs to a
shift. Cash taken out of or put into the drawer is recorded with a reason.

**Expected cash** = opening cash + cash sales + cash in − cash out. The next release adds cash
refunds and udhaar collections.

### Checkout
1. The client sends pack IDs, quantities, discounts, payment and a unique request key.
2. The server finds the prices and calculates all totals itself.
3. In one transaction it locks the branch stock rows (always in the same order), checks stock,
   writes the sale, items, payment and stock ledger rows, and assigns the receipt number.
4. A repeat of the same request key returns the first receipt.

Lines for the same variant in different pack sizes are added together before the stock check.

### Sales reports
By day, week and month: number of sales, total sales, discounts, per cashier and per branch, and
gross profit for managers and owners. Dates follow the shop's time zone (Proposed: `Asia/Karachi`,
week starts Monday).

## 9. Customer credit (udhaar), next release

- A customer has a credit limit.
- A ledger holds debit entries (credit sale) and credit entries (payment received).
- Ledger entries are added, never edited. The balance is calculated from them.
- A sale on credit above the limit needs a manager.

## 10. Data model (first release)

Every table has `id` and timestamps. Business tables have `tenant_id`.

| Table | Main columns |
|---|---|
| tenants | name, status |
| branches | name, address, timezone |
| users | name, email, password, role, branch_id, is_active |
| categories | name, parent_id |
| units | code, name, allows_decimal |
| products | code, name, name_ur, category_id, track_stock, is_active |
| product_variants | product_id, name, unit_id, reorder_level, is_active |
| variant_packs | variant_id, name, base_quantity, price_paisa, barcode, is_active |
| vendors | code, name, phone, address |
| goods_receipts | branch_id, grn_no, vendor_id, vendor_invoice_no, received_on, status, posted_by, posted_at |
| goods_receipt_items | goods_receipt_id, variant_id, variant_pack_id, quantity, base_quantity_per_pack, unit_cost, line_total_paisa |
| branch_stock | branch_id, variant_id, quantity, average_cost |
| stock_movements | branch_id, variant_id, type, quantity, reference, user_id, note |
| shifts | branch_id, user_id, opened_at, closed_at, opening_cash_paisa, expected_cash_paisa, counted_cash_paisa |
| cash_movements | shift_id, direction, amount_paisa, reason, user_id |
| sales | branch_id, shift_id, user_id, receipt_no, subtotal_paisa, discount_paisa, tax_paisa, rounding_paisa, total_paisa, idempotency_key |
| sale_items | sale_id, variant_id, variant_pack_id, product_name, variant_name, pack_name, base_quantity_per_pack, quantity, unit_price_paisa, unit_cost, discount_paisa, tax_paisa, line_total_paisa |
| payments | sale_id, method, amount_paisa, tendered_paisa, change_paisa |
| document_counters | branch_id, type, last_number |

`tax_paisa` stays zero until tax is built.

## 11. Core rules

1. Money is stored as whole paisa. Unit costs use four decimals. No floating-point numbers.
2. Quantities use exact decimals.
3. Posted sales and GRNs are never edited or deleted.
4. Sale lines keep copies of name, pack size, price and cost.
5. Every stock change has a stock ledger row, in the same transaction.
6. A sale or GRN is saved completely or not at all.
7. Receipt and GRN numbers come from a locked counter row per branch, so there are no gaps.
8. A request key stops double submission.
9. The server never trusts totals from the client.
10. Sensitive actions record who and when.
11. No secrets or real shop data in Git. The repository is public.

## 12. Build phases

| Phase | Content | Done when |
|---|---|---|
| 0. Foundation | Laravel with the React starter kit, PostgreSQL, Pest, Pint, GitHub Actions with PostgreSQL | A fresh clone runs on both machines and CI is green |
| 1. Shops and access | Tenants, branches, users, roles, policies | Tests prove tenant isolation and role limits |
| 2. Catalog and vendors | Categories, units, products, variants, pack sizes, vendors | The pencil example can be registered |
| 3. Stock and GRN | GRN, branch stock, ledger, adjustment, average cost | Posting a GRN updates stock and cost correctly |
| 4. Shifts and checkout | Shifts, cash movements, sales, payments, receipt numbers | Tests prove totals, stock deduction, blocking and double-submit protection |
| 5. Reports | Day, week, month | Report totals match the seeded sales |
| 6. Screens | React screens for phases 1 to 5 | A pretend shop day works from the browser, with database backup and restore tested |

Dates are left out until we know the weekly hours of each person.

## 13. Differences from the current documents

- Multi-tenancy and branches replace the single-store design.
- Inertia + React replaces the Vue proposal.
- Backend is built before the screens.
- Pack sizes are added. Price and barcode move from the variant to the pack size. Base unit moves
  from the product to the variant. The variant `sku` is replaced by the product code.
- Vendors and GRN move into the first release.
- Shifts move into the first release, because every sale belongs to a shift.
- Tax moves out of the first release.
- `_cents` column names become `_paisa`.
- `stock_qty` moves to a per-branch stock table.
- Added: cash movements, cost snapshot on sale lines, gap-free number counters, shop time zone.
- Refunds, when built, record the shift and payment method, and cannot exceed the quantity sold.
- Order discounts, when built, are split across lines in proportion, with the remainder on the last
  line.
- Use the current Laravel starter kit instead of Breeze, and turn off public registration.
- Suggested: protect `develop` as well as `main`.

## 14. Open questions

1. The first release has no void and no vendor return. A wrong sale or GRN can only be handled with
   a stock adjustment, which does not correct sales totals or average cost. Is that acceptable, or
   do these two move into the first release?
2. Is udhaar fine in the next release, or is it needed in the first one?
3. Do we buy from vendors on credit? If yes, we also need a vendor ledger.
4. Do any products need expiry dates or batches?
5. Does the owner portal only record what shops owe us, with no online payment?
6. Do we create new shops ourselves, or can a shop register itself?
7. What kind of shop is the first target?
