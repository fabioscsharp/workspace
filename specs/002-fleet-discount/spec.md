# Feature Specification: Fleet discount (TICKET-142)

**Feature Branch**: `002-fleet-discount` (in `src/dealerdesk`)

**Created**: 2026-11-15

**Status**: Draft (run `/speckit-clarify` before planning; depends on `001-delivery-pipeline`)

**Baseline**: delta against `specs/000-baseline/spec.md`. Changes BR-020, BR-021, BR-023,
BR-030 to BR-034, BR-040 to BR-047, BR-050 and BR-052; all other `BR-` items stay as is.
Structure: `docs/architecture.md`.

**Input**: `src/dealerdesk/TICKET-142.md`, requested by the Sales Director, priority High:
fleet customers (Greenway Taxis first) get an automatic 4% off the vehicle price,
shown as its own line on quotes and invoices; everything else unchanged.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Fleet customers get an automatic 4% discount on quotes (Priority: P1)

A salesperson prices a vehicle for a fleet customer. The quote automatically includes a
"Fleet discount" line equal to 4% of the vehicle price, fees are not discounted, and the
salesperson's own discount allowance is untouched.

**Why this priority**: it is the whole point of the ticket and the fleet contracts start
next month.

**Independent Test**: flag customer 2008 as fleet, quote vehicle 1001 (list 10495.00)
with 0% discount and no trade-in, and check the quote lines and total.

**Acceptance Scenarios**:

1. **Given** a fleet customer and vehicle 1001 (10495.00), **When** a quote is made with
   0% discount and no trade-in, **Then** the quote shows "Fleet discount -419.80", net
   price 10075.20, VAT 2045.04, delivery 150.00, registration 55.00 and **TOTAL 12325.24**
   (provisional figures per D-01/D-03/D-11 in `assumptions.md`).
2. **Given** a fleet customer, **When** the salesperson also enters a discount, **Then**
   both lines show and the salesman discount is applied per assumption D-03 (open).
3. **Given** a non-fleet (retail or trade) customer, **When** the same quotes are made,
   **Then** every line and the total are identical to today's output, to the penny.
4. **Given** a fleet customer, **Then** delivery (150.00) and registration (55.00) are not
   discounted.

---

### User Story 2 - Staff can mark customers as fleet (Priority: P1)

Sales staff can set the fleet flag when adding a new customer and on existing customers
such as Greenway Taxis, and see the status in customer listings.

**Why this priority**: without it nobody can receive the discount.

**Independent Test**: add a customer as fleet, restart the app, and confirm the flag is
still set; flag Greenway Taxis (2008) via the customer menu and confirm it persists.

**Acceptance Scenarios**:

1. **Given** the Add Customer screen, **When** staff answer that the customer is a fleet
   customer, **Then** the customer is saved as fleet and shows as fleet in the list.
2. **Given** an existing customer, **When** staff use the new "Set fleet status" action
   with their ID, **Then** the flag is saved immediately like every other change.
3. **Given** a fleet customer, **When** staff set it back to not-fleet, **Then** future
   quotes have no fleet discount; past invoices are untouched.
4. **Given** the existing data files, **When** the new version starts, **Then** every
   existing customer loads as non-fleet with no manual conversion.

---

### User Story 3 - Sales on fleet customers produce invoices with a separate fleet line (Priority: P1)

Confirming a sale to a fleet customer creates an invoice that shows the fleet discount as
its own line, and the invoice totals add up.

**Why this priority**: the ticket requires the line on invoices as well as quotes.

**Independent Test**: sell vehicle 1001 to a fleet customer, view the invoice, check the
fleet line and total equal the quote's.

**Acceptance Scenarios**:

1. **Given** a fleet quote, **When** the sale is confirmed, **Then** the invoice total
   equals the quote total and the invoice view shows a "Fleet discount" line.
2. **Given** existing invoices (50230, 50231, 50232), **When** viewed with the new
   version, **Then** they print exactly as before (including 50232 for Greenway Taxis).
3. **Given** a non-fleet sale, **Then** the invoice view output is identical to the old
   version's.
4. **Given** the new binary has written invoices, **When** the old binary is restored
   (rollback), **Then** it still loads all files and shows correct totals (E-01).

---

### Edge Cases

- A customer who is both trade and fleet (D-02, open): both discounts apply on the list
  price unless the business says otherwise.
- Fleet discount rounding: `rnd2(list * 0.04)`; the vehicle price has two decimals.
- Vehicle list price changes between quote and sale: sale uses the current price (as
  today).
- Trade-in larger than the net price (not validated today): unchanged behaviour.
- Fleet flag toggled after a quote was shown on screen: quotes are not stored, so
  nothing to reconcile.
- Customer ID not found when setting fleet status: "Customer not found." like other
  screens.
- Old data row with 5 fields, or a new row with an unexpected 6th value: treated as not
  fleet unless it is exactly the fleet value (C-07, C-08).
- Greenway's earlier manual 8% discount on invoice 50232 is not recalculated (C-14).

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: A customer MUST have a persistent fleet flag, settable when adding a
  customer and for existing customers (ticket points 1, D-08, D-09).
- **FR-002**: `calc_quote` MUST apply a fleet discount of 4% of the vehicle list price,
  rounded with `rnd2`, for fleet customers only (ticket point 2, D-01).
- **FR-003**: Fees (delivery 150.00, registration 55.00) MUST NOT be discounted (D-12).
- **FR-004**: The fleet discount MUST reduce the net price so VAT follows the existing
  rule `(net + delivery) * 20%` (D-11).
- **FR-005**: Quotes and invoice previews MUST show a "Fleet discount" line for fleet
  customers, following the trade-discount line format; it is omitted when zero (D-06).
- **FR-006**: Invoices MUST store the fleet discount separately so the invoice view can
  show its own line; `disc` remains the total of all discounts (D-07).
- **FR-007**: For non-fleet customers, quotes, invoices, listings and menus MUST be
  byte-identical to the current version, and money identical to the penny (ticket
  point 4, constitution I).
- **FR-008**: Existing `customers.dat` and `invoices.dat` MUST load without conversion;
  missing new fields default to non-fleet / zero (E-01, ticket acceptance).
- **FR-009**: The old binary MUST still be able to read files written by the new binary
  (rollback safety, E-01).
- **FR-010**: The fleet flag MUST be visible in customer listings and the quote header
  for fleet customers only (D-10).
- **FR-011**: The feature MUST ship through the pipeline built in
  `001-delivery-pipeline` (build, tests, CI) with new tests for fleet pricing, the flag
  persistence, and the invoice view.
- **FR-012**: Existing documentation is not edited; changes to documented behaviour are
  appended, dated, to the bottom of the relevant file; new rules are listed in
  `assumptions.md` (constitution IV, V).
- **FR-013**: Salesman discount rules (0-15% clamp, no PIN) MUST NOT change (B-04).

### Key Entities

- **Customer**: id, name, phone, email, type (R/T), **fleet flag (new)**.
- **Quote** (transient): list, trade discount, **fleet discount (new)**, salesman
  discount, net, delivery, VAT, registration, trade-in, total.
- **Invoice**: number, date, vehicle, customer, list, total discounts, **fleet discount
  (new, subset of discounts)**, net, VAT, fees, trade-in, total.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: Every non-fleet quote and invoice scenario in the characterization suite
  (spec 001) passes unchanged: 0 differing cents or characters.
- **SC-002**: A fleet quote for 10495.00 with no other discount and no trade-in totals
  12325.24 (provisional until D-01/D-03 are confirmed).
- **SC-003**: Staff can flag a customer as fleet in under 30 seconds from the main menu
  (at most three menu selections).
- **SC-004**: The sample data files from commit `a2ba708` load with zero manual edits and
  every existing invoice displays as before.
- **SC-005**: Fleet discounts are no longer entered manually through the salesman field
  for fleet contracts (removes the inconsistent manual process described in the ticket).

## Assumptions

Full list with reasoning and status in `src/dealerdesk/assumptions.md` (section D, E).
Key defaults, all still open for confirmation by the Sales Director:

- 4% applies to the list price, additive with trade discount for trade+fleet customers
  (D-01, D-02).
- Salesman percentage applies after trade and fleet discounts (D-03).
- Fleet amount on the invoice is stored in a new trailing field; `disc` stays the
  total (D-07).
- Fleet customers are marked through the app (new Customers menu action), not by
  editing `data/customers.dat` (D-09).
- [NEEDS CLARIFICATION: Can a customer be both trade and fleet, and if so do the 10%
  trade discount and the 4% fleet discount stack? (D-02)]
- [NEEDS CLARIFICATION: Is the 4% taken from the list price or from the price after
  the trade discount? Does the salesman % apply before or after it? (D-01, D-03)]
- [NEEDS CLARIFICATION: Exact wording and placement of the "Fleet discount" line on
  invoices, where the trade discount has no own line today (D-06).]
