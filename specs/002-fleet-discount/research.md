# Research: Fleet discount (002)

Phase 0 of [plan.md](plan.md). The spec has no `NEEDS CLARIFICATION`, and the business
rules are decided in `src/dealerdesk/assumptions.md` (D-01 to D-20, E-02 to E-07). These
are the technical decisions. Each one becomes an assumption row (D-21 onwards) when
implemented.

## R-1 Worked examples checked against the real rounding

- **Decision**: the spec's four worked examples are correct as written. They were computed
  with the app's own `rnd2` (`floor(x*100+0.5)/100`) and the new formula:

  | Case | Fleet disc | Salesman | Net | VAT | Total | `disc` stored |
  |------|-----------|----------|-----|-----|-------|---------------|
  | 10495.00 R fleet 0% | 419.80 | 0.00 | 10075.20 | 2045.04 | 12325.24 | 419.80 |
  | 10495.00 R fleet 5% | 419.80 | 503.76 | 9571.44 | 1944.29 | 11720.73 | 923.56 |
  | 13495.00 T fleet 0% | 485.82 | 0.00 | 11659.68 | 2361.94 | 14226.62 | 1835.32 |
  | 10495.00 R non-fleet 8% | 0.00 | 839.60 | 9655.40 | 1961.08 | 11821.48 | 839.60 |

- **Note**: vehicles 1001 (10495.00) and 1012 (13495.00) are **SOLD** in the sample data, so
  the e2e scenarios use a committed fixture where they are in stock (R-5). Unit tests use
  the prices directly.
- **Rationale**: spec 001 showed that `rnd2` can disagree with paper arithmetic
  (F-19, 2564.325 → 2564.32). Final expected values are still captured from the
  implementation with `test_calc --print`, then compared against this table.

## R-2 `calc_quote` interface

- **Decision**: `void calc_quote(QUOTE *q, double list, char cust_type, int fleet, double
  disc_pct, double tradein)`, and `QUOTE` gains `double fleet_disc` (placed after
  `trade_disc`). There is one call site (`build_quote`), which passes `c->fleet`. The unit
  test passes `0` for every existing case.
- **Rationale**: pricing stays pure and in `calc.c` (constitution VI). An explicit parameter
  keeps the unit tests independent of `CUSTOMER`.
- **Alternatives considered**: passing `CUSTOMER*` (couples calc.c to globals.h), or a
  separate `apply_fleet()` (splits the ordering rule across two functions).

## R-3 Loader that keeps empty fields (E-04)

- **Decision**: add a small static helper in `dbfile.c`,
  `static char *next_field(char **p)`. It returns the text up to the next `|` (possibly
  empty), terminates it, and advances `*p`; it returns NULL once the line is used up. Use
  it for `load_customers` and `load_invoices` only. `load_stock` keeps `strtok` (E-03, no
  change). Missing trailing fields default to `type 'R'`, `fleet 0` and `fleet_disc 0`.
  The flag is fleet only when the 6th field is exactly `1`. An empty type field loads as
  `'R'`, as a missing one does today.
- **Rationale**: `strtok` merges `||`, which shifts every later field (C-16). The split
  must keep empty fields for the appended 6th field to be read reliably.
- **Behaviour change**: rows that contain `||` now load unshifted. The sample data has no
  such row, so every 001 golden is unaffected. Scenario 28 pins the new behaviour (approved
  in E-04 and the spec Edge Cases).
- **Alternatives considered**: `strsep` (not C89, and missing on some platforms), `sscanf`
  with `%[^|]` (fails on empty fields).

## R-4 Writing and showing the new fields

- **Customers**: `fprintf("%d|%s|%s|%s|%c|%d\n", ..., type, fleet)`, always (E-02).
- **Invoices**: append `|%.2f` with `fleet_disc` to every row (E-02). On a sale,
  `disc = trade_disc + fleet_disc + salesman` and `fleet_disc = q.fleet_disc` (D-07).
- **Type text**: one helper `type_txt(const CUSTOMER *c)` returns `"Retail"`, `"Trade"`,
  `"Retail, Fleet"` or `"Trade, Fleet"`. It is used by `print_cust_row` and `print_quote`
  (D-10). Non-fleet strings are identical to today's.
- **Quote/preview line**: `"  Fleet discount           %12.2f\n"` with `-fleet_disc`,
  printed after the trade line and before the salesman line, only when `fleet_disc > 0`.
  It has the same width as `Trade discount` (D-06).
- **Invoice view**: if `fleet_disc > 0`, set `rem = rnd2(disc - fleet_disc)` and print
  `Discounts` as `-rem`, or as `0.00` when `rem < 0.005` (avoids `-0.00`, D-18). Then print
  `"  Fleet discount           %12.2f\n"` with `-fleet_disc`. Otherwise the code path is
  exactly today's, `-0.00` included.
- **Version**: `screen.c` header `v2.4` (B-06). Goldens normalise it.

## R-5 Test fixtures for fleet scenarios

- **Decision**: `run.sh` gains one rule. A fixture name that is a directory
  `tests/e2e/fixtures/<name>/` is copied as is. `sample`, `empty` and `full` keep their
  meaning. Two committed fixtures:
  - `fleet`: a copy of the sample with 1001 and 1012 in stock (status 0), Greenway (2008)
    `|R|1`, Marlow Motors (2002) `|T|1` for trade+fleet, all rows in the new format, and
    invoices with `|0.00`.
  - `legacy-gaps`: a customer with an empty phone (5 fields), one with an empty email and
    the flag (`|R|1`), one with an unexpected flag `|T|2`, an 11-field invoice, and a
    12-field fleet invoice.
- **Rationale**: the sample `data/` must not be edited (constitution III, D-09), and the
  worked-example vehicles are SOLD there. The fixtures are text, so they are reviewable.
- **Alternatives considered**: flagging customers inside each scenario through the menu
  (more keystrokes, and it couples unrelated scenarios), or generating fixtures with a
  script (less readable than committed files).

## R-6 Re-baselining spec 001 goldens (E-07)

- **Decision**:
  1. Implement.
  2. Run `make test`. The expected failures are exactly: output of 02, 04, 10, 11, 12, 21,
     22 (customer menu line, add prompt, help line) and data of 11, 16, 17 (trailing
     fields).
  3. Add the extra answer to `11-customer-add/input` (`N` for each added customer).
  4. Run `FORCE=1 make golden` for exactly those scenarios.
  5. Review `git diff`. Every changed line must be one of the approved lines or an
     appended `|0` / `|0.00`.
  6. A small check, `tests/e2e/check-rebaseline.sh`, run once and not part of `make test`,
     confirms that by stripping the approved additions from the new goldens and comparing
     against the 001 goldens from git.
- **Rationale**: this keeps the strict byte comparison (clarify Q1) and makes the
  re-baseline itself verifiable rather than eyeballed.

## R-7 Old-binary compatibility check (US3 #4, E-05)

- **Decision**: `tests/compat.sh` (target `make test-compat`, not part of `make test`):
  1. Export the `a2ba708` sources with `git archive` to a temp dir and build them there.
  2. Run the old binary on the data written by scenario 26 (a fleet sale), listing and
     viewing the invoices.
  3. Assert that it loads, shows the fleet invoice's `TOTAL` correctly (the same as v2.4),
     and does not crash.

  The data-loss-on-save behaviour is documented, not asserted.
- **Rationale**: "old binaries SHOULD still read new files" (constitution III) becomes
  checkable. It is kept out of `make test` because it needs git history.
