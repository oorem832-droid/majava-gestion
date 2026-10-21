# MAJAVA — Rental management system (Supabase version 2, MAJAVA V1 rules)

| File | Purpose |
|---|---|
| `01_schema.sql` | Tables, six roles, row-level security, workflow rules enforced by the database |
| `02_catalogue_et_demo.sql` | The 354 articles of the client's inventory (STOCK_MAJAVA_NV): available stock, rental price and damage cost per piece; plates, cutlery and glasses are stored per piece (lots are built in the app). Clients and orders are **fictitious demo data** |
| `03_inventaire_production.sql` | Same as below but **with no fictitious data at all**: also removes the demo clients (c1–c8) and suppliers (s1–s6) and creates the starting stock. Use this one for the real platform |
| `03_maj_inventaire.sql` | Run once on an **already installed** database: replaces the catalogue by the inventory above and **erases orders, purchases and stock movements** (demo/test data). Keeps accounts, clients, suppliers and settings |
| `04_import_factures_reelles.sql` | Imports the real clients and the 88 real orders extracted from MAJAVA's invoices (invoice number = order number, real dates, discounts, transport, deposits). Run **after** `03_inventaire_production.sql`; re-running replaces the already imported orders (no duplicates). Also adds the 4 catalogue items that only appear on invoices |
| `index.html` | The application (French UI, login screen, six roles) |
| `config.js` | Your Supabase URL and public key |
| `tests/` | The test scripts used to check the schema and the front end (optional) |

## What changed from version 1
- Six roles: admin, commercial, stock manager (`stock`), driver, maintenance agent (`maint`), accountant.
- Order lifecycle: Devis → Pré-réservation (1–3 h hold, auto-expiry) → Confirmée (full payment required: articles + transport + caution, no advance; delivery and pick-up time slots chosen) → Sortie (prepared + invoice and caution paid in full; there is no exception) → Retournée (received by stock/maintenance, then settled by the commercial); Expirée / Annulée.
- Cancellation (article 10): refund is computed (full if >72 h before delivery, else 50 %, caution returned) and editable; recorded as negative payment entries (`refund: true`)
- Item states: available / reserved / rented / in maintenance / damaged (last two excluded from availability until a manual state change).
- Caution (rate set by admin), daily cash sheet, timestamped movement history, non-returned order flag.
- New `settings` table (caution rate, max hold hours), editable by the admin only.

## Set-up (about 30 minutes)
1. Create a project at supabase.com. In **SQL Editor** run `01_schema.sql`, then `02_catalogue_et_demo.sql` (for production keep only the PRODUITS block).
| `03_maj_inventaire.sql` | Run once on an **already installed** database: replaces the catalogue by the inventory above and **erases orders, purchases and stock movements** (demo/test data). Keeps accounts, clients, suppliers and settings |
2. **Authentication → Sign In / Providers → Email**: keep Email on, turn **OFF "Allow new users to sign up"**.
3. **Authentication → Users → Add user** (tick "Auto confirm user"): create the **owner first** — the first account becomes **Administrateur**. Other accounts start as **Livreur** (no access to amounts); the admin sets the right role in **Équipe & accès → Modifier**.
4. Copy the **Project URL** and **anon public** key into `config.js`. Never use the `service_role` key here.
5. Upload `index.html` + `config.js` to a static host (Netlify, Vercel, Cloudflare Pages, GitHub Pages).
6. Optional: schedule `select public.expire_holds();` every minute with pg_cron. The app also calls it at every load and refresh.

## Rules enforced by the database (not only by the screens)
Confirmation needs a recorded advance · delivery needs confirmed + prepared + caution paid + date within one day · only the right role performs each step · maintenance/damaged units never count as available · overlapping bookings are refused even under simultaneous clicks · movement history cannot be edited or deleted · closed orders cannot be altered.

## Confidentiality of amounts (security update)

Only **admin, commercial and accountant** can read prices, payments, deposits, discounts and fees.
Stock, driver and maintenance staff never read the `rentals` table: they read the view `rentals_ops`
(all amounts replaced by 0, plus a `fully_paid` yes/no flag so the driver is still blocked when an order is unpaid)
and change an order only through `ops_update_rental` (whitelisted columns: status, prepared, out/return times, losses, incidents).
- New install: `01_schema.sql` already contains it.
- Existing database: run `05_securite_montants.sql` once in SQL Editor, then publish the new `index.html`.
- Not covered: product purchase cost (visible to staff roles) and supplier purchases (admin/stock only).
- Also required on Supabase: Authentication > disable "Allow new users to sign up"; never put the service_role key in `config.js`; keep `04_import_factures_reelles.sql` out of GitHub.

## Test checklist
- [ ] Commercial: create an order, pre-reserve it, record an advance → Confirmée.
- [ ] Stock manager: prepare it. Driver: try to deliver → refused until the date (the day before the event); orders can only be confirmed once fully paid. Check the slots, the devis (conditions on page 2) and the WhatsApp / e-mail buttons.
- [ ] Stock or maintenance: receive the return (1 missing, 1 to clean). Commercial: settle the return (late days, damage, retention), then close the caution. Print the 5 documents. Check Caisse du jour as accountant.
- [ ] Pre-reserve, wait past the deadline, refresh: it shows Expirée and the stock is free again.
- [ ] Two browsers: confirm two orders needing the same chairs on the same date — the second is refused.
- [ ] Logged out: only the login screen appears.

## What was tested, and what was not
- ✅ `01_schema.sql` + seed on real PostgreSQL 16 with a stand-in for Supabase Auth: all checks pass (`tests/test.sh`) — per-role permissions, workflow refusals, double-booking, maintenance exclusion, expiry, append-only history, settings.
- ✅ Front end driven end to end (`tests/sbtest.js`) against that real Postgres through a stand-in for the Supabase JavaScript client: login, hold → advance → prepare → blocked delivery → caution → delivery → return with losses → caution closed → settings, across five roles.
- ❌ **Not tested against a real Supabase project** (PostgREST, Auth service, hosted settings). Run the checklist above first.

## Known limits (production items)
- **Amounts are hidden by the screens, not by the database**: any staff account can technically read payments through the API. Production: a restricted view or column-level access for roles without the "money" right.
- Document numbers (LOC-…, ACH-…) are generated in the browser; a collision shows a "retry" message. Production: numbering in the database.
- A return writes several tables in separate calls, not one transaction.
- Demo records show "—" for staff names (their ids are not real accounts); real records use the signed-in user.
- Lines, payments and incidents are JSON inside the order. Fine at this size.
- Refresh every 45 s, no real-time push. Password reset is done from the Supabase dashboard.
- Printing uses the browser print dialog; server-generated PDFs are a production item.


**Updating an existing database:** re-run `01_schema.sql` (safe to run again: it adds the new columns and rules, existing data is kept). Replace `index.html` and keep your own `config.js`.


## Supabase notice (30 Oct 2026)
From 30 October 2026 Supabase no longer grants Data API access automatically on new tables. `01_schema.sql` now ends with explicit GRANTs (authenticated and service_role; anon deliberately none), so a new project works as before. Existing projects are unaffected. **Do not run `00_reset.sql` on a live project after that date unless you then re-run `01_schema.sql`** (tables are recreated, so their grants must be re-applied).
