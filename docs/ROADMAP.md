# POS System — Roadmap

> **This is a living document.** The phases and items below are a first plan. We will change
> them as we learn, and we will re-order, add and remove things. Update this file in a pull
> request whenever the plan changes, and keep [ARCHITECTURE.md](ARCHITECTURE.md) in sync.

- **Last updated:** 2026-11-17
- **Budget:** zero. Everything runs locally; tools are free (see Architecture §2).
- **How to track:** one GitHub **Milestone** per phase, one **Issue** per checklist item.
- **Dates:** intentionally left out until we know how many hours each of us can give per week.

---

## Phase 0 — Foundation

*Goal: both of us can run the same empty app locally and ship a change through a pull request.*

- [ ] Agree on the open questions in Architecture §10 (at least: business type, tax, frontend)
- [ ] Install PHP, Composer, PostgreSQL, Node on both machines
- [ ] Create the Laravel project in this repo (Inertia + Vue + Tailwind starter kit)
- [ ] Configure PostgreSQL connection; `.env.example` committed, `.env` ignored
- [ ] Add `.gitignore`, README setup instructions ("clone → run" in under 10 minutes)
- [ ] Add Pest and Laravel Pint
- [ ] Add GitHub Actions workflow: run tests + Pint on every PR
- [ ] Create GitHub Milestones and Issues for Phase 1
- [ ] Agree on the branch/PR workflow (Architecture §11)

**Done when:** a fresh clone runs on both machines, CI is green, and one trivial PR has been
reviewed and merged by each of us.

---

## Phase 1 — MVP: sell something

*Goal: a cashier can log in, sell products for cash, and the stock goes down.*

- [ ] Login/logout, roles (`admin`, `cashier`), seeded demo users
- [ ] Units (pcs, kg, g, litre, dozen) with decimal rules; quantities stored as decimals
- [ ] Categories, products and a default variant per product: create, edit, deactivate, search
- [ ] Barcode and SKU on variants (scanner works as keyboard input)
- [ ] Block the sale when stock is zero or too low (with row locking)
- [ ] Tax rates (basic, per product)
- [ ] **Checkout screen:** search/scan, cart, quantities, remove item, line and order totals
- [ ] **Cash payment** with tendered amount and change calculation
- [ ] `CompleteSale` action: one transaction, price snapshots, stock deduction, receipt number
- [ ] Double-submit protection (idempotency key)
- [ ] Printable receipt page (80 mm layout)
- [ ] Stock ledger (`stock_movements`) and manual stock adjustment (admin)
- [ ] Sales list with receipt lookup
- [ ] Feature tests for checkout, stock deduction and rounding (money correctness!)

**Done when:** we can run a pretend shop day with seeded products: sell items, print receipts,
see correct stock afterwards, and the automated tests prove the totals are right.

---

## Phase 2 — Run a shop day

*Goal: the things a manager needs to trust the system at closing time.*

- [ ] Manager role and permission policies (Architecture §7)
- [ ] Variants editor: several variants per product, attributes, per-variant price and stock
- [ ] Shifts: open with opening cash, close with counted cash and difference
- [ ] **Void** and **refund** flows (reasons, who, stock reversal, partial refunds)
- [ ] Discounts (line and order level) with a limit above which a manager must approve
- [ ] Reports: daily sales, sales by cashier, best sellers, low stock, shift summary
- [ ] CSV export of reports (free, and useful for accountants)
- [ ] Audit log for sensitive actions
- [ ] Low-stock alerts on the dashboard
- [ ] Settings page (shop name, currency, receipt header/footer)

**Done when:** a manager can open a shift, let a cashier trade, handle a refund, close the shift
and see a report whose numbers match the cash in the drawer.

---

## Phase 3 — Real-world polish

*Goal: make it comfortable and safe enough for an actual small shop.*

- [ ] Split payments (e.g. part cash, part card) and "card/other" payment recording
- [ ] Customers (optional link to a sale), purchase history
- [ ] Suppliers and receiving stock (simple purchase entries)
- [ ] Customer credit (udhaar) ledger, if the chosen shop needs it
- [ ] Urdu names and bilingual receipts, if needed
- [ ] FBR POS integration, if the chosen business requires it (research requirements first)
- [ ] Keyboard shortcuts for fast checkout
- [ ] Hold/resume a cart (serve another customer while one is searching for their wallet)
- [ ] **Automatic database backups** (scheduled `pg_dump`)
- [ ] Documentation for a shop owner: install, daily use, backup/restore
- [ ] Pilot with one real shop (or a friendly test user), collect feedback

**Done when:** a real shop can use it for a full week without us touching the code, and
backups have been restored successfully at least once.

---

## Phase 4 — Later / ideas

*Not committed. Revisit once Phase 3 is done.*

- Loyalty points and promotions
- Multiple branches
- Direct thermal printer support (ESC/POS), cash drawer kick
- Tablet/touch-friendly layout
- Offline-capable mode
- Hosting for remote access (only if a free, trustworthy option exists)
- Online card payment integration (usually has fees, so conflicts with zero-budget)
- Multi-language and RTL

---

## Suggested split of work (to confirm with Hashim)

To avoid both of us editing the same files:

| Area | Owner (proposal) |
|---|---|
| Database design, migrations, `CompleteSale`/stock/refund Actions, tests | Person A |
| Checkout and back-office screens (Vue), receipt layout | Person B |
| Reports, settings, CI, docs | Shared, taken per issue |

Swap per phase if useful, and always agree on a table's columns **before** the screen that uses it.

---

## Risks to watch

| Risk | Mitigation |
|---|---|
| Money rounding bugs | Integer cents, one `Money` helper, tests with awkward numbers (e.g. 3 × 33.33, 15% tax) |
| Overselling or lost sales under double clicks | Transactions, row locking, idempotency key |
| Scope creep (loyalty, branches, cards…) | Anything not in the current phase goes to Phase 4 first |
| Two people, different local setups | Seeders + migrations + documented setup; no shared database |
| Public repo leaking secrets | `.env` ignored, no real data in seeders, review PRs for keys |
| Losing real data later | Backups are a Phase 3 must-have before any real use |

---

## Change log

| Date | Change |
|---|---|
| 2026-11-17 | First draft created (proposal, pending review by both of us) |
