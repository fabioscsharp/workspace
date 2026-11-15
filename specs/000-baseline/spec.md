# Baseline Specification: DealerDesk v2.3 as it behaves today

**Created**: 2026-11-15

**Status**: Reference (describes existing behaviour; changes only when the application changes)

**Source**: `src/dealerdesk` at commit `a2ba708`. Derived from the code, data files and git
history, **not** from the README (which is stale, see `src/dealerdesk/assumptions.md`
section B). Where this file and the README disagree, this file reflects the code.

Every feature spec (001, 002, ...) is a delta against this baseline. Requirements here
use the `BR-` prefix so specs can cite them. When a feature changes behaviour listed
here, that feature's spec MUST say which `BR-` it changes, and this file is updated in
the same change. Related: `docs/architecture.md` (structure, data formats).

## Users and context

Dealership staff (sales, stock, admin) use a single-user, menu-driven terminal app,
started from the repository root, with no login. Every change is saved immediately.

## Functional Requirements

### Main menu

- **BR-001**: Main menu options: 1 Stock, 2 Customers, 3 Quote, 4 Sell vehicle,
  5 Invoices, 9 Help, 0 Exit. Any other value shows "Invalid option". Non-numeric input
  reads as 0 (`atoi`), which exits from the main menu and goes back from submenus.
  End of input exits the program. (There is no Reports option; B-05.)
- **BR-002**: Every screen starts with a clear-screen and a header showing
  `DEALERDESK v2.3` and the screen title. Screens that show output wait for ENTER.

### Stock

- **BR-010**: List stock (status IN STOCK or RESERVED) and list all vehicles (incl. SOLD):
  id, plate, make, model (truncated to 22), year, mileage, price, status, with a count.
- **BR-011**: Add vehicle: plate, make, model, year, mileage, cost, list price. Plate and
  price > 0 are required. The id comes from the vehicle counter (starts at 1000), status
  IN STOCK. Plate is stored as typed (not uppercased; B-08). Rejects when 500 vehicles
  exist.
- **BR-012**: Find by plate: exact, case-sensitive match; shows id, vehicle, mileage,
  price, status, or "No vehicle with plate ...".
- **BR-013**: Change price: by vehicle id; a new price > 0 is saved, otherwise "Price not
  changed".
- **BR-014**: Reserve / release toggles IN STOCK and RESERVED; a SOLD vehicle cannot be
  reserved ("Vehicle is already sold.").

### Customers

- **BR-020**: List customers: id, name, phone, email, type shown as "Trade" for type `T`
  and "Retail" for anything else, with a count.
- **BR-021**: Add customer: name (required), phone, email, "Trade customer (Y/N)?"
  (`Y`/`y` gives type `T`, anything else `R`). The id comes from the customer counter
  (starts at 2000). Rejects when 500 customers exist.
- **BR-022**: Find customer: name contains the text (case-sensitive substring).
- **BR-023**: There is no way to edit or delete a customer (D-09).

### Quote and sale

- **BR-030**: Quote asks for vehicle id, customer id, discount %, trade-in amount. Errors:
  "Vehicle not found.", "Vehicle is already sold.", "Customer not found." A reserved
  vehicle can be quoted and sold. Quotes are never stored. The header reads
  "QUOTATION (valid 7 days)" but validity is not enforced (B-09).
- **BR-031**: Pricing rules (see next section) are computed from the vehicle's current
  list price, the customer's type, the discount % and the trade-in.
- **BR-032**: Quote layout (money right-aligned, two decimals): customer name and type,
  vehicle plate, make and model; then `List price`, `Trade discount` (only if > 0),
  `Discount (x.x%)` (only if > 0), `Net price`, `Delivery`, `VAT 20%`,
  `Registration fee`, `Trade-in` (only if not 0), `TOTAL`. Discount and trade-in are
  shown as negative amounts.
- **BR-033**: Sell vehicle shows the same figures as "INVOICE PREVIEW" and asks
  "Confirm sale (Y/N)?". Only `Y`/`y` confirms; otherwise "Sale cancelled." and nothing is
  saved. Rejects when 2000 invoices exist.
- **BR-034**: On confirmation, an invoice is created (number from the invoice counter,
  starts at 50000; today's date `YYYY-MM-DD`), the vehicle becomes SOLD, and invoices,
  stock and counters are saved.

### Pricing rules (as implemented in `calc.c`)

All amounts are rounded to two decimals with `rnd2` (half up on doubles; C-02) unless
stated.

- **BR-040**: Trade discount `T = rnd2(list * 10%)` when the customer type is `T`, else 0
  (B-01: not 5%).
- **BR-041**: The salesman discount percentage typed by the user is clamped to 0..15
  (below 0 becomes 0, above 15 becomes 15) with no PIN (B-04). The stored/displayed
  percentage is the clamped one.
- **BR-042**: Salesman discount `D = rnd2((list - T) * pct / 100)`.
- **BR-043**: `net = list - T - D` (not rounded; C-01).
- **BR-044**: Delivery is a fixed 150.00 and is VATable (B-03). Registration fee is a fixed
  55.00 and is VAT exempt.
- **BR-045**: `VAT = rnd2((net + delivery) * 20%)` (B-02).
- **BR-046**: `total = net + delivery + VAT + registration - trade-in`. The trade-in is
  not validated and may exceed the total.
- **BR-047**: Pricing depends only on list price, customer type, discount % and
  trade-in; there is no stacking logic beyond the order above.

Worked examples (from the sample data):

| Case | List | Type | Pct | Trade-in | Trade disc | Disc | Net | VAT | Total |
|------|------|------|-----|----------|-----------|------|-----|-----|-------|
| Invoice 50230 | 9295.00 | R | 5 | 1500.00 | 0.00 | 464.75 | 8830.25 | 1796.05 | 9331.30 |
| Invoice 50231 | 13495.00 | T | 0 | 0.00 | 1349.50 | 0.00 | 12145.50 | 2459.10 | 14809.60 |
| Invoice 50232 | 10495.00 | R | 8 | 0.00 | 0.00 | 839.60 | 9655.40 | 1961.08 | 11821.48 |

### Invoices

- **BR-050**: Invoice record: number, date, vehicle id, customer id, list, `disc` (trade +
  salesman discount combined), net, VAT, `fees` (delivery + registration combined),
  trade-in, total (C-04).
- **BR-051**: List invoices: number, date, vehicle id, customer name ("(unknown)" if
  missing), total, with a count.
- **BR-052**: View invoice by number: header with date, customer, vehicle (`?` if
  missing), then `List price`, `Discounts` (negative), `Net price`, `Fees`, `VAT`,
  `Trade-in` (only if not 0), `TOTAL`. "Invoice not found." otherwise. Invoices are never
  recalculated (C-03).

### Help and persistence

- **BR-060**: The Help screen prints static text (known stale in parts: B-01, B-04).
- **BR-070**: Data is read from `./data` at start-up and every change writes the
  affected whole file at once. A missing file means an empty table. Details:
  `docs/architecture.md`.

## Non-functional characteristics (current)

- **BR-080**: Single process, single user, no locking: hand-editing `.dat` files while the
  app runs is unsupported.
- **BR-081**: Capacity limits: 500 vehicles, 500 customers, 2000 invoices (C-10).
- **BR-082**: No automated tests, no CI, build and release are manual.

## Known defects and oddities (preserved, not fixed)

Plate search is case-sensitive while stored plates are mixed case (B-08); stale help text
(B-01, B-04); no input validation on numbers (C-12); save errors are printed but ignored
(C-11); quote validity is cosmetic (B-09).

## Success criteria for the baseline

Used by spec 001 to define "unchanged": for each `BR-` item, a characterization test
exists whose expected output was generated from the code at commit `a2ba708`.
