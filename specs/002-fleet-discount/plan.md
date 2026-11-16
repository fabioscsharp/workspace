# Implementation Plan: Fleet discount (TICKET-142)

**Branch**: `002-fleet-discount` (work branch in `src/dealerdesk`:
`feature/page-tests-ticket-142`, on top of spec 001 commit `fcff60d`) | **Date**: 2026-11-16 |
**Spec**: [spec.md](spec.md)

**Input**: Feature specification from `specs/002-fleet-discount/spec.md` (clarified
2026-11-15 and 2026-11-16)

## Summary

Add a persistent fleet flag to customers and an automatic 4% fleet discount, shown on its own
line on quotes, previews and invoices. Release as v2.4. The change touches:

- `calc.c`/`calc.h`: `QUOTE` gains `fleet_disc`, and `calc_quote` gains an `int fleet`
  parameter. Pricing order: trade → fleet → salesman, each rounded with `rnd2`.
- `globals.h`: `CUSTOMER.fleet` (int) and `INVOICE.fleet_disc` (double).
- `dbfile.c`:
  - the customer and invoice loaders split on every `|`, keeping empty fields (E-04);
  - the savers always write field 6 (customers) and field 12 (invoices) (E-02).
- `main.c`:
  - the `, Fleet` type marker;
  - the Fleet discount line on quotes and previews;
  - fleet amounts stored on invoices;
  - the invoice view's reduced Discounts line plus the Fleet discount line;
  - the Add Customer fleet prompt;
  - new Customers menu option `4. Set fleet status`;
  - one Help line.
- `screen.c`: `v2.3` → `v2.4`.

Safety net: the spec 001 suite. Non-fleet behaviour must keep passing, and only the goldens
listed in FR-007 and FR-009 are re-baselined with `FORCE=1 make golden`, each with a
reviewed diff. New unit cases and e2e scenarios cover fleet pricing, flag persistence,
empty-field loading, the invoice view, and old-binary compatibility.

## Technical Context

**Language/Version**: C, conservative C89 for application code (B-11, FR-016). Tests as in
spec 001.

**Primary Dependencies**: none new. Same Makefile, plain-C unit test, POSIX `sh` e2e runner.

**Storage**: `data/customers.dat` gains a 6th field and `data/invoices.dat` a 12th field
(trailing, always written). `stock.dat` and `next_ids.dat` are unchanged (E-03). The sample
`data/` is **not** edited (D-09).

**Testing**:
- `make test`: unit and e2e, with new cases and scenarios.
- `make test-mutation`: still applies.
- New `make test-compat`: the `a2ba708` binary reads files written by v2.4.

**Target Platform**: Linux/macOS/WSL with gcc or clang (as 001). Verified on WSL gcc 15.2.

**Project Type**: interactive terminal application (CLI).

**Performance Goals**: n/a (single user; the suite stays well under the 2-minute SC-001
budget of spec 001).

**Constraints**:
- Non-fleet output is character-identical, apart from the 4 listed exceptions (FR-007).
- Money is identical to the penny.
- Old files load without conversion.
- Old binaries can read new files (with the documented data loss on save, E-05).
- No refactor beyond what the feature needs (constitution VII).

**Scale/Scope**: about 6 source files touched, 1 new menu screen, about 7 new e2e
scenarios, about 10 new unit cases, 7 + 3 re-baselined goldens.

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Principle | Check | Result |
|-----------|-------|--------|
| I. Behaviour preservation | This is a requested feature. The only non-fleet visible changes are the 4 approved in spec FR-007 (version, menu line, add prompt, help line). Every other 001 golden must pass unchanged. Re-baselines are listed and diff-reviewed. | PASS |
| II. Code is the source of truth | Worked examples were re-computed with the app's `rnd2` and all 4 match the spec ([research.md](research.md) R-1). Final expected values are captured from the implementation, as in 001. | PASS |
| III. Backward-compatible data (NON-NEGOTIABLE) | The new fields are trailing. Missing fields default to non-fleet/0. The loaders keep empty fields; the only change is that rows containing `\|\|` now load unshifted (E-04, an approved fix covered by a test). Old binaries ignore the trailing fields (`test-compat`). Rollback data loss is documented (E-05). Sample `data/` is untouched. | PASS (E-04 is an approved, tested behaviour change) |
| IV. Docs append-only | Dated sections are appended to `README.md` and `RELEASE.txt`. `NOTES.md` is a new file. `assumptions.md` gets new rows and status changes only. | PASS |
| V. Assumptions written down | D-18 to D-20 and E-07 were added in clarify. Planning decisions R-1 to R-7 become D-21 onwards during implementation. | PASS |
| VI. Tested, reproducible | `make test` covers it, and pricing stays in `calc.c` with no I/O. CI comes later (spec 003). | PASS |
| VII. Simplicity, minimal diffs | Only the listed functions change. There is one new screen function and one small field-splitting helper in `dbfile.c`. No restructuring. | PASS |

**Knowledge-file duties (CLAUDE.md rule 4)**, in the same change:
- `specs/000-baseline/spec.md`: update BR-002, BR-020, BR-021, BR-023, BR-030 to BR-034,
  BR-040 to BR-047, BR-050, BR-052, BR-060 and BR-070.
- `docs/architecture.md`: structs, file formats, loader behaviour, the new menu action,
  and the version.

## Project Structure

### Documentation (this feature)

```text
specs/002-fleet-discount/
├── spec.md
├── plan.md              # this file
├── research.md          # Phase 0: R-1..R-7
├── data-model.md        # Phase 1: Customer, Quote, Invoice deltas, file formats
├── quickstart.md        # Phase 1: validation guide
├── contracts/
│   ├── screens.md       # exact screen text: quote/preview, invoice view, customer
│   │                    # list, add customer, set fleet status, menus, help
│   └── data-files.md    # customers.dat (6 fields) / invoices.dat (12 fields),
│                        # loader and writer rules, compatibility matrix
└── tasks.md             # Phase 2 (/speckit-tasks)
```

### Source Code (`src/dealerdesk`)

```text
calc.h / calc.c        QUOTE.fleet_disc; calc_quote(q, list, cust_type, fleet, disc_pct, tradein)
globals.h              CUSTOMER.fleet (int 0/1); INVOICE.fleet_disc (double)
dbfile.c               next_field() splitter for customers/invoices; 6th/12th field read+write
main.c                 type_txt(); print_quote fleet line; sell stores fleet_disc;
                       view_invoice fleet lines; add_customer fleet prompt;
                       set_fleet_status() + menu option 4; help line
screen.c               header version v2.4
tests/unit/test_calc.c existing cases call with fleet=0 (unchanged expectations) + fleet cases
tests/e2e/run.sh       fixture lookup also accepts a committed dir tests/e2e/fixtures/<name>/
tests/e2e/fixtures/
  fleet/               sample data copy: 1001 and 1012 IN STOCK, 2008 fleet (R),
                       one trade+fleet customer, new-format rows (6/12 fields)
  legacy-gaps/         rows with empty phone/email (5- and 6-field), an 11-field invoice,
                       an unexpected flag value, a fleet invoice (12 fields)
tests/e2e/scenarios/   23..29 new (see coverage map); 02,04,10,11,12,16,17,21,22 re-baselined
tests/e2e/check-rebaseline.sh  one-off: re-baselined goldens differ from fcff60d only by
                       the approved lines / trailing fields (research R-6)
tests/compat.sh        builds the a2ba708 binary from git and runs it on v2.4-written data
Makefile               + test-compat target
README.md, RELEASE.txt APPEND dated v2.4 sections (RELEASE.txt: back up data/ first, E-05)
NOTES.md               NEW: rollback data-loss risk (FR-014); completed at the end (F-08)
```

**Structure Decision**: keep the existing modules (constitution VII). The fleet logic goes
where the trade logic already lives. Fixtures needed by the new scenarios are committed
directories, because sample `data/` must not change and the vehicles in the worked
examples (1001, 1012) are SOLD in the sample.

## Coverage map (new and changed tests)

| Test | What it proves | Spec |
|------|----------------|------|
| unit: all existing 30 cases with `fleet=0` | non-fleet pricing is unchanged to the penny | FR-007, SC-001 |
| unit: 4 worked examples + fleet with pct 15/20/−5, trade-in, list 0, 0.05 | fleet formula, order, clamp outside cap | FR-002 to FR-004, FR-015, SC-002 |
| 23-fleet-set-status | set Y, Enter (not changed), N, unknown id; list shows `Retail, Fleet`; customers.dat field 6 | US2 #2, #3, D-19, FR-001, FR-010 |
| 24-fleet-add-customer | Add Customer with fleet Y and N; list markers; field 6 written | US2 #1 |
| 25-fleet-quote (fixture `fleet`) | the 4 worked examples on screen; line order; fees undiscounted | US1 #1 to #6 |
| 26-fleet-sell-invoice (fixture `fleet`) | preview, sale, invoice view with `Discounts 0.00` + `Fleet discount`; invoices.dat 12th field | US3 #1, #2, D-18, FR-006 |
| 27-fleet-sell-trade-pct (fixture `fleet`) | trade+fleet+salesman sale; reduced Discounts line | US3 #2, FR-006 |
| 28-legacy-gaps (fixture `legacy-gaps`) | empty-field rows load unshifted; 5/11-field rows non-fleet; flag `2` = not fleet; old invoices print as before | US2 #4, US3 #3, FR-008, FR-009, E-04 |
| 29-existing-invoices | 50230 to 50232 view identical under v2.4 (sample data) | US3 #3, SC-004 |
| re-baselined 02,04,10,11,12,21,22 (output) | only the menu line / add prompt / help line differ | FR-007 |
| re-baselined 11,16,17 (data) | only the trailing `\|0` / `\|0.00` differ | FR-009, E-07 |
| `make test-compat` | the a2ba708 binary loads v2.4 files and shows the same totals | US3 #4, E-05 |
| `make test-mutation` | still kills the VAT mutant | constitution VI |

## Constitution Re-check (after Phase 1)

The design ([data-model.md](data-model.md), [contracts/](contracts/),
[quickstart.md](quickstart.md)) keeps every gate at PASS.

- **Screens**: the contracts list every changed line, and everything else is v2.3 byte for
  byte.
- **Data files**: only trailing fields are added, and the compatibility matrix covers both
  directions.
- **Docs**: append-only, or new files.
- **Dependencies**: none new.

Spec correction made during planning: FR-007's re-baseline list now also names
21-missing-data and 22-capacity-full, which open the Customers menu too.

For `/speckit-tasks`, include as explicit tasks:
- the baseline and architecture updates;
- the README and RELEASE.txt appends;
- `NOTES.md`;
- the D-21+ assumption rows;
- the re-baseline review.

## Complexity Tracking

No constitution violations. E-04 (loading `||` rows unshifted) is an approved behaviour fix,
listed in the spec Edge Cases and pinned by 28-legacy-gaps.
