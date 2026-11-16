# Data Model: Fleet discount (002)

These are deltas against `docs/architecture.md`. The on-disk formats are in
[contracts/data-files.md](contracts/data-files.md).

## Customer (`CUSTOMER`, `customers.dat`)

| Field | Type | Rule |
|-------|------|------|
| id, name, phone, email | unchanged | the loader now keeps empty phone/email (E-04) |
| type | char `R`/`T` | unchanged. A missing **or empty** field loads as `R` |
| **fleet** (new) | int 0/1 | field 6. `1` only if the field is exactly `1`. Missing, empty, or any other value loads as 0 |

Lifecycle: `fleet` is set by Add Customer (Y = 1, anything else = 0) and changed only by
Set fleet status with an explicit Y (→1) or N (→0) (D-19). It is saved immediately with the
whole file. There is no history: changing it never alters past invoices (D-13).

## Quote (`QUOTE`, transient)

| Field | Rule |
|-------|------|
| list, trade_disc | unchanged (BR-040) |
| **fleet_disc** (new) | `fleet ? rnd2((list - trade_disc) * 0.04) : 0` (D-01, D-02) |
| disc_pct | clamped 0..15 (unchanged; fleet is outside the cap, D-05) |
| disc | `rnd2((list - trade_disc - fleet_disc) * disc_pct / 100)` (D-16) |
| net | `list - trade_disc - fleet_disc - disc` (not rounded, as today) |
| delivery, vat, reg_fee, tradein, total | unchanged formulas (BR-044 to BR-046) |

Invariant: with `fleet = 0`, every field equals the 001 unit-test expectation.

## Invoice (`INVOICE`, `invoices.dat`)

| Field | Rule |
|-------|------|
| no … tradein, total | unchanged |
| disc | total of **all** discounts: `trade_disc + fleet_disc + disc` (D-07) |
| **fleet_disc** (new) | field 12, `%.2f`. A missing field loads as 0 |

Invariant: `list - disc = net` holds for every invoice, old or new, so old binaries show
correct totals. The view derives "Discounts" as `disc - fleet_disc` when `fleet_disc > 0`
(D-18).

## Test fixtures (new)

| Fixture | Purpose |
|---------|---------|
| `fleet` | sample data with 1001/1012 in stock, 2008 fleet (R), 2002 fleet (T), new format |
| `legacy-gaps` | empty-field rows, old 5/11-field rows, an unexpected flag value, a 12-field fleet invoice |

Both are committed directories under `tests/e2e/fixtures/` ([research.md](research.md) R-5).
