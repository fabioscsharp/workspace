# Feature Specification: Fleet discount (TICKET-142)

**Feature Branch**: `002-fleet-discount` (in `src/dealerdesk`)

**Created**: 2026-11-15

**Status**: Clarified (all questions in `src/dealerdesk/assumptions.md` section D and E are
decided, 2026-11-15, and clarified again 2026-11-16 against the spec 001 test suite: see
Clarifications); ready for `/speckit-plan`. Depends on `001-build-and-tests`; does not wait for
`003-ci-and-release` (tests run locally until CI exists).

**Baseline**: delta against `specs/000-baseline/spec.md`. Changes BR-002, BR-020, BR-021,
BR-023, BR-030 to BR-034, BR-040 to BR-047, BR-050, BR-052, BR-060 and BR-070; all other `BR-` items
stay as is. Structure: `docs/architecture.md`.

**Input**: `src/dealerdesk/TICKET-142.md`, requested by the Sales Director, priority High:
fleet customers (Greenway Taxis first) get an automatic 4% off the vehicle price,
shown as its own line on quotes and invoices; everything else unchanged.

## Clarifications

### Session 2026-11-16

- Q: Spec 001 compares data files byte for byte, but 002 appends a field to every
  customer/invoice row. How are the affected 001 goldens handled? → A: Keep the byte check.
  Re-baseline only the 001 golden data that changes (`FORCE=1 make golden`). The reviewed
  diff must show nothing but the appended `|0` / `|0.00` fields. There is no parsed-value
  comparison.
- Q: Which visible changes are allowed for non-fleet use? → A: The Customers menu gains
  `4. Set fleet status` between `3. Find customer` and `0. Back`; Add Customer asks
  `Fleet customer (Y/N)? ` right after the trade question; the header shows v2.4. The Help
  line from Q4 below is also allowed. Nothing else changes.
- Q: On "Set fleet status", what does an answer other than Y or N do? → A: The screen first
  shows the customer's current status. Y/y sets fleet and N/n clears it. Anything else,
  including just Enter, leaves the flag unchanged, prints "Fleet status not changed." and
  saves nothing.
- Q: Should the Help screen mention the fleet discount? → A: Yes. Add one line,
  `  - Fleet customers receive an automatic 4% fleet discount.`, directly after the
  trade-discount line. The existing stale lines stay untouched, and 04-help is re-baselined.
- Q: On a fleet invoice whose remaining discounts (`disc - fleet_disc`) are 0, what does the
  "Discounts" line show? → A: `0.00`, unsigned. On fleet invoices a zero remainder prints
  `0.00`, not `-0.00`. Non-fleet invoices keep today's output exactly (including `-0.00`
  when `disc` is 0).

## Pricing rule (new order)

For every customer, in this order, each step rounded with `rnd2`:

```
trade_disc = (type == 'T') ? rnd2(list * 10%) : 0                    # unchanged (BR-040)
fleet_disc = fleet ? rnd2((list - trade_disc) * 4%) : 0              # new (D-01, D-02)
salesman   = rnd2((list - trade_disc - fleet_disc) * pct / 100)      # pct clamped 0..15 (D-16, D-05)
net        = list - trade_disc - fleet_disc - salesman
vat        = rnd2((net + 150.00) * 20%)                              # unchanged (BR-045)
total      = net + 150.00 + vat + 55.00 - trade-in                   # unchanged (BR-046)
```

For a non-fleet customer `fleet_disc` is 0 and the formulas reduce exactly to today's, so
output is unchanged (constitution I).

Worked examples (provisional until tests pin them from the implementation, computed with
the same `rnd2` as the code):

| Case | List | Type | Fleet | Pct | Trade disc | Fleet disc | Salesman | Net | VAT | Total |
|------|------|------|-------|-----|-----------|-----------|----------|-----|-----|-------|
| Retail fleet, no discount | 10495.00 | R | yes | 0 | 0.00 | 419.80 | 0.00 | 10075.20 | 2045.04 | 12325.24 |
| Retail fleet, 5% | 10495.00 | R | yes | 5 | 0.00 | 419.80 | 503.76 | 9571.44 | 1944.29 | 11720.73 |
| Trade + fleet, no discount | 13495.00 | T | yes | 0 | 1349.50 | 485.82 | 0.00 | 11659.68 | 2361.94 | 14226.62 |
| Non-fleet (unchanged) | 10495.00 | R | no | 8 | 0.00 | 0.00 | 839.60 | 9655.40 | 1961.08 | 11821.48 |

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Fleet customers get an automatic 4% discount on quotes (Priority: P1)

A salesperson prices a vehicle for a fleet customer. The quote automatically includes a
"Fleet discount" line, fees are not discounted, and the salesperson's own discount
allowance (0 to 15%, separate from fleet) is untouched.

**Why this priority**: it is the whole point of the ticket and the fleet contracts start
next month.

**Independent Test**: flag customer 2008 as fleet, quote vehicle 1001 (list 10495.00) with
0% discount and no trade-in, and check the quote lines and total. Vehicles 1001 and 1012
are SOLD in the sample data, so the automated tests use the test fixture `fleet`, where
they are in stock and 2008/2002 are already fleet (plan research R-5).

**Acceptance Scenarios**:

1. **Given** a fleet customer and vehicle 1001 (10495.00), **When** a quote is made with 0%
   discount and no trade-in, **Then** the quote shows "Fleet discount -419.80", net price
   10075.20, delivery 150.00, VAT 2045.04, registration 55.00 and **TOTAL 12325.24**.
2. **Given** the same customer and vehicle with a 5% discount, **Then** the quote shows
   fleet -419.80, discount (5.0%) -503.76 and TOTAL 11720.73 (salesman % applies after the
   fleet discount, D-16).
3. **Given** a customer who is both trade and fleet (list 13495.00, 0%), **Then** the quote
   shows trade discount -1349.50, then fleet discount -485.82, and TOTAL 14226.62 (D-01, D-02).
4. **Given** a non-fleet (retail or trade) customer, **When** the same quotes are made,
   **Then** every line and the total are identical to today's output, to the penny.
5. **Given** a fleet customer, **Then** delivery (150.00) and registration (55.00) are not
   discounted, and the 15% limit applies only to the typed percentage (D-05, D-12).
6. **Given** a quote, **Then** the "Fleet discount" line appears directly after the
   "Trade discount" line and before the "Discount (x.x%)" line, in the same format
   (D-06), and is omitted when the fleet discount is 0.

---

### User Story 2 - Staff can mark customers as fleet (Priority: P1)

Sales staff can set the fleet flag when adding a customer and on existing customers such as
Greenway Taxis, and see the status in customer listings.

**Why this priority**: without it nobody can receive the discount.

**Independent Test**: add a customer as fleet, restart the app, and confirm the flag is
still set; flag Greenway Taxis (2008) via the Customers menu and confirm it persists.

**Acceptance Scenarios**:

1. **Given** the Add Customer screen, **When** staff answer Y to the new "Fleet customer
   (Y/N)?" prompt (asked right after the trade question), **Then** the customer is saved as fleet and shows `Retail, Fleet` or
   `Trade, Fleet` in the list and in the quote/invoice-preview customer line (D-10).
2. **Given** an existing customer, **When** staff use the new Customers menu action
   `4. Set fleet status` with the customer's ID, see the current status and answer Y/N,
   **Then** the flag is saved immediately like every other change (D-09). Any other answer,
   including just Enter, prints "Fleet status not changed." and writes nothing. An unknown
   ID prints "Customer not found."
3. **Given** a fleet customer set back to not-fleet, **Then** future quotes have no fleet
   discount; past invoices are untouched (D-13).
4. **Given** the existing data files, **When** the new version starts, **Then** every
   existing customer loads as non-fleet with no manual conversion, and customers with an
   empty phone or email read their flag reliably (E-04).
5. **Given** non-fleet customers, **Then** the customer list and all other customer
   screens print exactly as today.

---

### User Story 3 - Sales on fleet customers produce invoices with a separate fleet line (Priority: P1)

Confirming a sale to a fleet customer creates an invoice that shows the fleet discount as
its own line, and the invoice totals add up.

**Why this priority**: the ticket requires the line on invoices as well as quotes.

**Independent Test**: sell vehicle 1001 to a fleet customer, view the invoice, check the
fleet line and total equal the quote's. As in US1, this runs on test fixture `fleet`,
because 1001 is SOLD in the sample data.

**Acceptance Scenarios**:

1. **Given** a fleet quote, **When** the sale is confirmed, **Then** the invoice total equals
   the quote total, `invoices.dat` stores the fleet amount in a new 12th field, and `disc`
   is the total of all discounts (trade + fleet + salesman) (D-07).
2. **Given** an invoice with a fleet amount, **When** viewed, **Then** the "Discounts" line
   shows the remaining discounts (`disc - fleet_disc`), followed directly by a "Fleet
   discount" line (negative, same format) before "Net price". A zero remainder prints as
   `0.00`, not `-0.00` (D-18). **Given** an invoice without one, **Then** the output is
   identical to the old version's (D-06).
3. **Given** existing invoices (50230, 50231, 50232), **When** viewed with the new version,
   **Then** they print exactly as before, including 50232 for Greenway Taxis.
4. **Given** the old binary restored over data written by the new binary, **Then** it still
   loads every file and shows correct totals, but its next save drops fleet flags and fleet
   amounts (E-05); this risk is documented (see FR-014).

---

### Edge Cases

- Customer who is both trade and fleet: both apply, trade first (D-01, D-02).
- Fleet discount rounding: `rnd2((list - trade_disc) * 0.04)`.
- Vehicle list price changes between quote and sale: the sale uses the current price, as today.
- Trade-in larger than the net price (not validated today): unchanged behaviour.
- Fleet flag toggled after a quote was shown: quotes are not stored, nothing to reconcile.
- Customer ID not found when setting fleet status: "Customer not found." like other screens.
- Set fleet status answered with Enter or anything other than Y/y/N/n: the flag is kept,
  "Fleet status not changed." is printed, and no file is written. This differs on purpose
  from Add Customer, where anything but Y means "not fleet" like the trade question.
- Old row with 5 customer fields, or an invoice row with 11 fields: loads as non-fleet /
  fleet 0. A new-format row with an unexpected flag value is treated as not fleet unless it is
  exactly `1`.
- Customer row with an empty phone or email (`2009|Acme||a@x|R|1`): the new loader keeps
  empty fields so the flag and type are read correctly; rows already containing `||` now
  load unshifted, which is a deliberate fix to be covered by a test (E-04).
- Greenway's earlier manual 8% discount on invoice 50232 is not recalculated (C-14).
- `data/customers.dat` is not edited by us: Greenway is flagged through the app (D-09).

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: A customer MUST have a persistent fleet flag, settable when adding a customer
  and for existing customers through a new Customers menu action that shows the current
  status and changes it only on an explicit Y or N (ticket point 1, D-08, D-09). Stored as a 6th trailing field in `customers.dat`: `1` fleet, anything else not.
- **FR-002**: `calc_quote` MUST compute `fleet_disc = rnd2((list - trade_disc) * 0.04)` for
  fleet customers only, then the salesman discount on `list - trade_disc - fleet_disc`
  (D-01, D-02, D-16). Its interface gains a fleet parameter; pricing stays free of I/O.
- **FR-003**: Fees (delivery 150.00, registration 55.00) MUST NOT be discounted (D-12).
- **FR-004**: The fleet discount MUST reduce the net price so VAT follows the existing rule
  `(net + delivery) * 20%` (D-11).
- **FR-005**: Quotes and invoice previews MUST show a "Fleet discount" line (negative
  amount) after the trade-discount line when the fleet discount is greater than 0 (D-04, D-06).
- **FR-006**: Invoices MUST store the fleet discount in a new 12th field of `invoices.dat`;
  `disc` stays the total of all discounts. The invoice view shows a "Fleet discount" line
  only when the fleet discount is greater than 0, placed directly after a reduced
  "Discounts" line (`disc - fleet_disc`; `0.00` unsigned when that is zero) and before
  "Net price" (D-06, D-07, D-18).
- **FR-007**: For non-fleet customers, quotes, invoices, listings and menus MUST be
  character-identical to the current version, and money identical to the penny (ticket
  point 4, constitution I). There are exactly four exceptions:
  1. the version header (FR-013);
  2. the Customers menu line `4. Set fleet status`, between `3. Find customer` and `0. Back`;
  3. the Add Customer prompt `Fleet customer (Y/N)? `, asked right after
     `Trade customer (Y/N)? `;
  4. the Help screen line `  - Fleet customers receive an automatic 4% fleet discount.`,
     directly after the trade-discount line. The other Help lines stay as they are,
     including the stale ones (BR-060).

  The spec 001 output goldens that show these screens (02-invalid-option, 04-help,
  10-customers-list, 11-customer-add, 12-customer-find, 21-missing-data,
  22-capacity-full) are re-baselined. 11-customer-add's `input` gains one
  answer per added customer. The reviewed diff MUST show only these lines.
- **FR-008**: Existing `customers.dat` and `invoices.dat` MUST load without conversion;
  missing new fields default to non-fleet / 0 (ticket acceptance criteria).
- **FR-009**: The customers and invoices loaders MUST keep empty fields (split on every
  `|`) so the appended field is read reliably (E-04). New code writes the new field on every
  record (E-02). Tests keep spec 001's byte-for-byte data comparison. The 001 golden
  data that changes is re-baselined with `FORCE=1 make golden`: today that is
  `expected-data/customers.dat` of 11-customer-add and `expected-data/invoices.dat` of
  16-sell-retail and 17-sell-trade. The reviewed diff of each re-baselined file MUST show
  only the appended `|0` (customers) or `|0.00` (invoices) field on each line.
- **FR-010**: The fleet marker `, Fleet` MUST be appended to the type text for fleet
  customers only, in the customer list and the quote/invoice-preview customer line (D-10).
- **FR-011**: The feature MUST ship through the build and test suite from
  `001-build-and-tests` (and CI from `003-ci-and-release` once it exists) with new tests for fleet pricing, flag persistence, empty-field
  loading and the invoice view.
- **FR-012**: Existing documentation is not edited; changes are appended, dated, to the
  bottom of the relevant file; new rules are listed in `assumptions.md` (constitution IV, V).
- **FR-013**: The on-screen header version MUST change from `v2.3` to `v2.4` for this release
  (B-06). The only other visible changes for non-fleet flows are the three listed in FR-007.
- **FR-014**: The rollback data-loss risk (old binary drops fleet data on its next save) MUST
  be documented in the release notes and `NOTES.md`, and a "back up `data/` before
  deploying" step MUST be appended as a dated section at the bottom of `RELEASE.txt` (E-05).
- **FR-015**: Salesman discount rules (0-15% clamp, no PIN) MUST NOT change (B-04, D-05).
- **FR-016**: Application code MUST be written in conservative C89 style (B-11).

### Key Entities

- **Customer**: id, name, phone, email, type (R/T), **fleet flag (new)**.
- **Quote** (transient): list, trade discount, **fleet discount (new)**, salesman discount,
  net, delivery, VAT, registration, trade-in, total.
- **Invoice**: number, date, vehicle, customer, list, total discounts, **fleet discount
  (new, a part of the total discounts)**, net, VAT, fees, trade-in, total.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: Every non-fleet quote and invoice scenario in the characterization suite
  (spec 001) passes with 0 differing cents or characters in its output (the version header
  is normalised). The only re-baselined golden data is the appended-field change listed
  in FR-009.
- **SC-002**: The four worked examples above produce exactly the stated figures.
- **SC-003**: Staff can flag a customer as fleet in at most three menu selections from the
  main menu.
- **SC-004**: The sample data files from commit `a2ba708` load with zero manual edits and
  every existing invoice displays as before.
- **SC-005**: Fleet discounts are no longer entered manually through the salesman field for
  fleet contracts (removes the inconsistent manual process described in the ticket).

## Assumptions

All decisions, with reasoning and the git evidence, are in
`src/dealerdesk/assumptions.md` (sections B, D and E); none is open. Summary:

- D-01/D-02/D-16: 4% of list minus trade; trade and fleet stack; salesman % after both.
- D-05: fleet is outside the 15% cap. D-06/D-07: invoice line only for fleet invoices,
  stored in a new trailing field. D-09/D-10: menu action, `, Fleet` marker.
- E-02/E-04/E-05: always write the field, loader keeps empty fields, rollback documented.
- Added 2026-11-16:
  - D-18: on a fleet invoice, "Fleet discount" follows "Discounts", and a zero remainder
    prints `0.00`.
  - D-19: Set fleet status changes the flag only on an explicit Y or N.
  - D-20: one new Help line.
  - E-07: keep byte-for-byte data goldens and re-baseline only the appended fields.
    This supersedes E-06.
- B-06/B-11: version v2.4, conservative C89.
- Not decided and out of scope: fleet pricing for other fee types, fleet reporting,
  per-customer fleet percentages (the 4% is fixed).
