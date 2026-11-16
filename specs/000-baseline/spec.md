# Baseline Specification: DealerDesk as it behaves today (v2.4)

**Created**: 2026-11-15. **Updated**: 2026-11-16 for v2.4, the fleet discount from spec 002
(TICKET-142).

**Status**: Reference (describes existing behaviour; changes only when the application changes)

**Source**: `src/dealerdesk` at commit `a2ba708`, plus the spec 001 build and tests and the
spec 002 fleet changes noted per item. Derived from the code, data files and git
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
  `DEALERDESK v2.4` (v2.3 before spec 002) and the screen title. Screens that show output wait for ENTER.

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
  and "Retail" for anything else, with `, Fleet` appended for fleet customers
  (`Retail, Fleet`, `Trade, Fleet`; spec 002), and a count. The Customers menu is 1 List,
  2 Add, 3 Find, 4 Set fleet status, 0 Back.
- **BR-021**: Add customer: name (required), phone, email, "Trade customer (Y/N)?"
  (`Y`/`y` gives type `T`, anything else `R`), then "Fleet customer (Y/N)?" (`Y`/`y` is
  fleet, anything else not; spec 002). Both prompts are asked before the name check. The id
  comes from the customer counter (starts at 2000). Rejects when 500 customers exist.
- **BR-022**: Find customer: name contains the text (case-sensitive substring).
- **BR-023**: There is no way to edit or delete a customer (D-09). The only change
  possible is the fleet flag, through "Set fleet status" (spec 002, D-19):
  - it shows the customer and `Fleet    : Yes|No`;
  - `Y`/`y` sets the flag and `N`/`n` clears it, each saved at once;
  - any other answer prints "Fleet status not changed." and saves nothing;
  - an unknown id prints "Customer not found."

### Quote and sale

- **BR-030**: Quote asks for vehicle id, customer id, discount %, trade-in amount. Errors:
  "Vehicle not found.", "Vehicle is already sold.", "Customer not found." A reserved
  vehicle can be quoted and sold. Quotes are never stored. The header reads
  "QUOTATION (valid 7 days)" but validity is not enforced (B-09).
- **BR-031**: Pricing rules (see next section) are computed from the vehicle's current
  list price, the customer's type and fleet flag, the discount % and the trade-in.
- **BR-032**: Quote layout (money right-aligned, two decimals): customer name and type text
  (BR-020), vehicle plate, make and model; then `List price`, `Trade discount` (only if
  > 0), `Fleet discount` (only if > 0; spec 002), `Discount (x.x%)` (only if > 0), `Net price`, `Delivery`, `VAT 20%`,
  `Registration fee`, `Trade-in` (only if not 0), `TOTAL`. Discount and trade-in are
  shown as negative amounts.
- **BR-033**: Sell vehicle shows the same figures as "INVOICE PREVIEW" and asks
  "Confirm sale (Y/N)?". Only `Y`/`y` confirms; otherwise "Sale cancelled." and nothing is
  saved. Rejects when 2000 invoices exist.
- **BR-034**: On confirmation, an invoice is created (number from the invoice counter,
  starts at 50000; today's date `YYYY-MM-DD`), storing `disc` as all discounts and
  `fleet_disc` separately (BR-050). The vehicle becomes SOLD, and invoices, stock and
  counters are saved.

### Pricing rules (as implemented in `calc.c`)

All amounts are rounded to two decimals with `rnd2` (half up on doubles; C-02) unless
stated.

- **BR-040**: Trade discount `T = rnd2(list * 10%)` when the customer type is `T`, else 0
  (B-01: not 5%).
- **BR-041**: The salesman discount percentage typed by the user is clamped to 0..15
  (below 0 becomes 0, above 15 becomes 15) with no PIN (B-04). The stored/displayed
  percentage is the clamped one.
- **BR-042**: Fleet discount `F = rnd2((list - T) * 4%)` for fleet customers, else 0
  (spec 002, D-01, D-02). It is outside the 15% cap (D-05). Salesman discount
  `D = rnd2((list - T - F) * pct / 100)` (D-16; with `F = 0` this is the v2.3 rule).
- **BR-043**: `net = list - T - F - D` (not rounded; C-01).
- **BR-044**: Delivery is a fixed 150.00 and is VATable (B-03). Registration fee is a fixed
  55.00 and is VAT exempt.
- **BR-045**: `VAT = rnd2((net + delivery) * 20%)` (B-02).
- **BR-046**: `total = net + delivery + VAT + registration - trade-in`. The trade-in is
  not validated and may exceed the total.
- **BR-047**: Pricing depends only on list price, customer type, fleet flag, discount % and
  trade-in. The order is trade, then fleet, then salesman, and there is no other stacking
  logic. Fees are never discounted (D-12).

Worked examples (from the sample data):

| Case | List | Type | Pct | Trade-in | Trade disc | Disc | Net | VAT | Total |
|------|------|------|-----|----------|-----------|------|-----|-----|-------|
| Invoice 50230 | 9295.00 | R | 5 | 1500.00 | 0.00 | 464.75 | 8830.25 | 1796.05 | 9331.30 |
| Invoice 50231 | 13495.00 | T | 0 | 0.00 | 1349.50 | 0.00 | 12145.50 | 2459.10 | 14809.60 |
| Invoice 50232 | 10495.00 | R | 8 | 0.00 | 0.00 | 839.60 | 9655.40 | 1961.08 | 11821.48 |

Fleet examples (v2.4, `tests/unit/test_calc.c`):

| Case | List | Type | Fleet | Pct | Trade disc | Fleet disc | Disc | Net | VAT | Total |
|------|------|------|-------|-----|-----------|-----------|------|-----|-----|-------|
| Retail fleet | 10495.00 | R | yes | 0 | 0.00 | 419.80 | 0.00 | 10075.20 | 2045.04 | 12325.24 |
| Retail fleet 5% | 10495.00 | R | yes | 5 | 0.00 | 419.80 | 503.76 | 9571.44 | 1944.29 | 11720.73 |
| Trade + fleet | 13495.00 | T | yes | 0 | 1349.50 | 485.82 | 0.00 | 11659.68 | 2361.94 | 14226.62 |

### Invoices

- **BR-050**: Invoice record: number, date, vehicle id, customer id, list, `disc` (trade +
  fleet + salesman discount combined), net, VAT, `fees` (delivery + registration combined),
  trade-in, total (C-04), `fleet_disc` (12th field, spec 002; 0 for older rows).
- **BR-051**: List invoices: number, date, vehicle id, customer name ("(unknown)" if
  missing), total, with a count.
- **BR-052**: View invoice by number: header with date, customer, vehicle (`?` if
  missing), then `List price`, `Discounts` (negative), `Net price`, `Fees`, `VAT`,
  `Trade-in` (only if not 0), `TOTAL`. "Invoice not found." otherwise. Invoices are never
  recalculated (C-03).

  When `fleet_disc > 0` (spec 002, D-18), `Discounts` shows `disc - fleet_disc` (`0.00`
  unsigned when zero) and is followed by `Fleet discount`. Other invoices print exactly as
  in v2.3, including `-0.00` when `disc` is 0.

### Help and persistence

- **BR-060**: The Help screen prints static text (known stale in parts: B-01, B-04). In v2.4
  it includes "Fleet customers receive an automatic 4% fleet discount." (D-20).
- **BR-070**: Data is read from `./data` at start-up and every change writes the
  affected whole file at once. A missing file means an empty table. Since v2.4:
  - `customers.dat` has 6 fields and `invoices.dat` has 12, always written;
  - their loaders keep empty fields, so `||` no longer shifts later fields (E-04);
  - older 5- and 11-field rows load as non-fleet / fleet 0.

  Details: `docs/architecture.md`.

## Non-functional characteristics (current)

- **BR-080**: Single process, single user, no locking: hand-editing `.dat` files while the
  app runs is unsupported.
- **BR-081**: Capacity limits: 500 vehicles, 500 customers, 2000 invoices (C-10).
- **BR-082**: Build is `make` (Makefile) and `make test` runs the characterization
  suite in `tests/` (unit and end-to-end, with goldens taken from `a2ba708`); both were
  added by spec 001 on 2026-11-16 and change no behaviour. The build compiles with no
  warnings under gcc `-Wall`. There is no CI, and release is manual until spec 003.

## Known defects and oddities (preserved, not fixed)

Plate search is case-sensitive while stored plates are mixed case (B-08); stale help text
(B-01, B-04); no input validation on numbers (C-12); save errors are printed but ignored
(C-11); quote validity is cosmetic (B-09).

## Success criteria for the baseline

Used by spec 001 to define "unchanged": for each `BR-` item, a characterization test
exists whose expected output was generated from the code at commit `a2ba708`.
