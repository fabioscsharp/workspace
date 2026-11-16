---

description: "Task list for 002: fleet discount (TICKET-142)"
---

# Tasks: Fleet discount (TICKET-142)

**Input**: Design documents from `specs/002-fleet-discount/`:
- [plan.md](plan.md)
- [spec.md](spec.md)
- [research.md](research.md)
- [data-model.md](data-model.md)
- [contracts/screens.md](contracts/screens.md)
- [contracts/data-files.md](contracts/data-files.md)
- [quickstart.md](quickstart.md)

**Prerequisites**:
- Spec 001 is committed (`src/dealerdesk` at `fcff60d`, `make test` green).
- Work in `src/dealerdesk` on `feature/page-tests-ticket-142`, in WSL (`make`, gcc).
- Run `git checkout -- data` after any manual run.
- Do not commit unless asked.

**Tests**: the spec requires them (FR-011), so test tasks are included in each story.

**Organization**:
- Phases 1 and 2 add the shared data, pricing and test infrastructure.
- US1 (quotes), US2 (flagging) and US3 (invoices) each add their own screens and tests.
- Phase 6 makes the approved visible changes to old screens (the Help line) and
  re-baselines the spec 001 goldens in one reviewed step.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: can run in parallel (different files, no dependency on an unfinished task)
- **[Story]**: US1, US2 or US3 (spec.md user stories)
- Paths are relative to the workspace root, and application paths start with `src/dealerdesk/`.

## Hard rules for every task

- Application code is conservative C89 (FR-016, B-11): declarations at block top, no `//`
  comments, and no C99-only features.
- Do **not** edit `src/dealerdesk/data/` (D-09). Fleet scenarios use committed fixtures.
- Existing docs (`README.md`, `RELEASE.txt`, `BRIEF.md`, `TICKET-142.md`) are append-only:
  a dated section at the bottom (constitution IV).
- Never hand-edit a golden. Use `make golden SCENARIO=...` for new scenarios, and
  `FORCE=1 make golden SCENARIO=...` only in Phase 6 for the listed 001 scenarios.
- Screen text must match [contracts/screens.md](contracts/screens.md) exactly. File formats
  must match [contracts/data-files.md](contracts/data-files.md).

---

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: confirm the safety net and add fleet test fixtures.

- [X] T001 In `src/dealerdesk`, run `make clean && make test && make test-mutation` and confirm everything is green at `fcff60d` before touching code. Record the result in the T039 notes
- [X] T002 Extend `src/dealerdesk/tests/e2e/run.sh` `make_fixture()`: a fixture name other than `sample`, `empty` or `full` that matches a directory `tests/e2e/fixtures/<name>/` has its `*.dat` files copied into the run's data dir. An unknown name still fails. Update the header comment and `src/dealerdesk/tests/README.md` "How a scenario works" (research R-5)
- [X] T003 [P] Create the fixture `src/dealerdesk/tests/e2e/fixtures/fleet/`, based on the sample `data/`:
  - `stock.dat`: same rows, but 1001 and 1012 have status `0`.
  - `customers.dat`: every row in the 6-field format `id|name|phone|email|type|fleet`, with 2008 Greenway Taxis `|R|1`, 2002 Marlow Motors `|T|1`, and the rest `|0`.
  - `invoices.dat`: the 3 sample invoices with `|0.00` appended.
  - `next_ids.dat`: as in the sample.
- [X] T004 [P] Create the fixture `src/dealerdesk/tests/e2e/fixtures/legacy-gaps/`:
  - `stock.dat`: copy of the sample.
  - `customers.dat` with these rows:
    - `2001|No Phone Ltd||np@example.com|T` (5 fields, empty phone);
    - `2002|No Email Fleet|07700 900001||R|1` (empty email, fleet);
    - `2003|Odd Flag|07700 900002|odd@example.com|T|2` (unexpected flag value);
    - `2004|Old Row|07700 900003|old@example.com|R` (5 fields);
    - `2005|Empty Type|07700 900004|et@example.com||1` (empty type, so it loads as `R`).
  - `invoices.dat`: one 11-field invoice for 2001, and one 12-field fleet invoice for 2002 whose `disc` includes `fleet_disc`. Compute its figures with the R-1 formula: list 10995.00, R, fleet, 0% gives fleet 439.80, net 10555.20, VAT 2141.04, total 12901.24, disc 439.80, fees 205.00.
  - `next_ids.dat` consistent with these rows.

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: data fields, file I/O and the pricing interface that every story needs.

**⚠️ CRITICAL**: no story work starts until this phase is done and the 001 unit tests still pass.

- [X] T005 In `src/dealerdesk/globals.h`:
  - add `int fleet;` to `CUSTOMER`, after `type`, with the comment `/* 1 = fleet customer (TICKET-142) */`;
  - add `double fleet_disc;` to `INVOICE`, after `total`, with the comment `/* part of disc */`.
- [X] T006 In `src/dealerdesk/dbfile.c`, add `static char *next_field(char **p)`. It returns the text up to the next `|` (which may be empty), NUL-terminates it, advances `*p` past the `|`, and returns NULL when `*p` is NULL. Rewrite `load_customers` and `load_invoices` to use it instead of `strtok` (`load_stock` is unchanged). The rules, verbatim from data-model.md:
  - type: "missing **or empty** field loads as `R`";
  - fleet: "`1` only if the field is exactly `1`. Missing, empty, or any other value loads as 0";
  - fleet_disc: "missing or empty → 0";
  - keep the existing defaults for the other missing fields (research R-3).
- [X] T007 In `src/dealerdesk/dbfile.c`, make `save_customers` write `"%d|%s|%s|%s|%c|%d\n"` (adding `fleet`) and `save_invoices` append `"|%.2f"` (`fleet_disc`) to every row, always (E-02, contracts/data-files.md)
- [X] T008 In `src/dealerdesk/calc.h` and `src/dealerdesk/calc.c`:
  - add `double fleet_disc;` to `QUOTE` after `trade_disc`;
  - change the signature to `void calc_quote(QUOTE *q, double list, char cust_type, int fleet, double disc_pct, double tradein)`;
  - compute `q->fleet_disc = fleet ? rnd2((list - q->trade_disc) * 0.04) : 0`, then `q->disc = rnd2((list - q->trade_disc - q->fleet_disc) * disc_pct / 100.0)` and `q->net = list - q->trade_disc - q->fleet_disc - q->disc`. VAT, fees and total stay unchanged.
  - Append a dated note to the header comment. Do not rewrite the existing stale comment lines.
  - Update the single call site in `src/dealerdesk/main.c` `build_quote` to pass `c->fleet`.
  - Keep the VAT line text `(q->net + q->delivery) * 0.20)` unchanged, because `tests/mutation.sh` targets it.
- [X] T009 In `src/dealerdesk/tests/unit/test_calc.c`, add an `int fleet` member to `CASE` and set it to `0` in all 30 existing cases. Pass it to `calc_quote`, include `fleet_disc` in the `--print` output and the comparisons (11 fields; the existing cases expect `"0.00"`), and run `make unit`. All existing expectations must pass **unchanged** (FR-007, SC-001)
- [X] T010 [P] In `src/dealerdesk/screen.c`, change the header from `v2.3` to `v2.4` (FR-013, B-06)
- [X] T011 Build and run `make test`. The expected result is that unit tests pass, and e2e fails only on the data goldens of 11-customer-add (`|0`), 16-sell-retail and 17-sell-trade (`|0.00`), plus the version-normalised output is unchanged everywhere. Investigate any other failure before continuing; do not re-baseline yet (Phase 6)

**Checkpoint**: the data and pricing foundations are in place, and non-fleet pricing is proven unchanged.

---

## Phase 3: User Story 1 - Fleet customers get an automatic 4% discount on quotes (Priority: P1) 🎯 MVP

**Goal**: quotes and invoice previews for fleet customers show a "Fleet discount" line and the correct total. Non-fleet quotes are unchanged.

**Independent Test**: `make unit` passes the 4 worked examples. `make e2e SCENARIO='25-*'` shows
TOTAL 12325.24, 11720.73 and 14226.62 for the fleet quotes and 11821.48 for the non-fleet
one, using fixture `fleet`.

- [X] T012 [US1] In `src/dealerdesk/main.c`, add `static const char *type_txt(const CUSTOMER *c)`, which returns `"Retail"`, `"Trade"`, `"Retail, Fleet"` or `"Trade, Fleet"`. Use it in `print_cust_row` (last column) and `print_quote` (`  Customer : %s (%s)`) in place of the inline `c->type == 'T' ? "Trade" : "Retail"` (D-10, FR-010)
- [X] T013 [US1] In `src/dealerdesk/main.c` `print_quote`, print `"  Fleet discount           %12.2f\n"` with `-q->fleet_disc` when `q->fleet_disc > 0`. It goes directly after the Trade discount line and before the Discount (x.x%) line (D-06, FR-005, contracts/screens.md)
- [X] T014 [P] [US1] In `src/dealerdesk/tests/unit/test_calc.c`, add fleet cases:
  - the 4 R-1 worked examples: 10495 R fleet 0%, 10495 R fleet 5%, 13495 T fleet 0%, and 10495 R non-fleet 8%;
  - fleet with pct 15, 20 (clamped to 15) and −5 (clamped to 0);
  - fleet with a trade-in;
  - fleet with list 0, and fleet with list 0.05.

  Capture the values with `--print`, paste them, confirm the 4 examples match the research.md R-1 table exactly, and run `make unit`
- [X] T015 [US1] Create `src/dealerdesk/tests/e2e/scenarios/25-fleet-quote/` with `fixture` = `fleet`. The input runs Quote (main 3) for:
  - 1001 / 2008 / 0 / 0;
  - 1001 / 2008 / 5 / 0;
  - 1012 / 2002 / 0 / 0;
  - 1001 / 2001 / 8 / 0 (non-fleet);
  - 1003 / 2008 / 15 / 1500 (fleet with trade-in; fees must be 150.00 and 55.00).

  Each ends with an empty line for the pause, then `0`. Generate with `make golden SCENARIO='25-*'` and check `expected.out` against the US1 acceptance scenarios 1 to 6 (line order, customer line `(Retail, Fleet)` / `(Trade, Fleet)`)

**Checkpoint**: US1 is complete; fleet pricing is visible on quotes.

---

## Phase 4: User Story 2 - Staff can mark customers as fleet (Priority: P1)

**Goal**: the fleet flag can be set on Add Customer and through the new "Set fleet status" screen, it persists, and old or gappy files load correctly.

**Independent Test**: `make e2e SCENARIO='2[348]-*'` passes. `expected-data/customers.dat`
shows field 6 set, the customer list shows `Retail, Fleet`, and empty-field rows load
unshifted.

- [X] T016 [US2] In `src/dealerdesk/main.c` `add_customer`, right after the trade `confirm`, add `c.fleet = confirm("  Fleet customer (Y/N)? ");`. Y/y means fleet; anything else means 0 (contracts/screens.md "Add Customer")
- [X] T017 [US2] In `src/dealerdesk/main.c`, add `static void set_fleet_status(void)`, exactly as in contracts/screens.md "Set Fleet Status":
  1. `header("Set Fleet Status")`, then read the Customer ID.
  2. An unknown id prints "Customer not found.".
  3. Otherwise show the customer and type text, `Fleet    : Yes|No`, then prompt with `read_line` into a small buffer.
  4. `Y`/`y` sets `fleet = 1` and saves; `N`/`n` sets `fleet = 0` and saves (`save_customers()`); anything else prints "Fleet status not changed." and does not save.
  5. End with `pause_enter()`.

  Add `printf("  4. Set fleet status\n");` between options 3 and 0 in `customer_menu`, and `case 4: set_fleet_status(); break;` (D-19, FR-001)
- [X] T018 [P] [US2] Create `src/dealerdesk/tests/e2e/scenarios/23-fleet-set-status/` (sample fixture). The input:
  1. List customers.
  2. Set 2008 → `Y`.
  3. Set 2008 → empty line (not changed).
  4. Set 2008 → `x` (not changed).
  5. Set 9999 (not found).
  6. List customers (2008 shows `Retail, Fleet`).
  7. Set 2001 → `Y`, then set 2001 → `N`.
  8. Exit.

  Generate, then check that `expected-data/customers.dat` has `|R|1` only for 2008 and `|0` everywhere else (US2 #2, #3)
- [X] T019 [P] [US2] Create `src/dealerdesk/tests/e2e/scenarios/24-fleet-add-customer/` (sample fixture). The input:
  1. Add `Fleet Retail Ltd` with trade N and fleet Y.
  2. Add `Fleet Trade Ltd` with trade Y and fleet Y.
  3. Add `Plain Co` with trade N and fleet N.
  4. List customers, then exit.

  Generate, then check the type texts and that field 6 is `1`, `1`, `0` (US2 #1)
- [X] T020 [P] [US2] Create `src/dealerdesk/tests/e2e/scenarios/28-legacy-gaps/` with `fixture` = `legacy-gaps`. The input lists customers, quotes 1002 for each of 2001 to 2005 at 0%, lists invoices, views both invoices, then exits. Generate and check:
  - 2001 is `Trade`, with its email in the email column (not shifted);
  - 2002 is `Retail, Fleet`;
  - 2003 is `Trade`, not fleet;
  - 2004 is `Retail`;
  - 2005 is `Retail, Fleet`;
  - the data is unchanged (no `expected-data/`).

  (US2 #4, FR-008, FR-009, E-04)

**Checkpoint**: US2 is complete; fleet status can be set, persists, and legacy rows load correctly.

---

## Phase 5: User Story 3 - Sales on fleet customers produce invoices with a separate fleet line (Priority: P1)

**Goal**: a fleet sale stores `fleet_disc` and the total `disc`, and the invoice view shows the reduced Discounts line plus the Fleet discount line. Old invoices print as before, and the old binary can still read the new files.

**Independent Test**: `make e2e SCENARIO='2[679]-*'` passes. `make test-compat` passes.

- [X] T021 [US3] In `src/dealerdesk/main.c` `sell_vehicle`, set `i->disc = q.trade_disc + q.fleet_disc + q.disc;` and `i->fleet_disc = q.fleet_disc;` (D-07, FR-006). Nothing else in the flow changes
- [X] T022 [US3] In `src/dealerdesk/main.c` `view_invoice`: when `p->fleet_disc > 0`, compute `rem = rnd2(p->disc - p->fleet_disc)` (include `calc.h`), print `Discounts` as `-rem`, or `0.0` when `rem < 0.005`, then `"  Fleet discount           %12.2f\n"` with `-p->fleet_disc`, before "Net price". Otherwise keep today's exact code path, including `-0.00` (D-18, contracts/screens.md "Invoice view")
- [X] T023 [P] [US3] Create `src/dealerdesk/tests/e2e/scenarios/26-fleet-sell-invoice/` with `fixture` = `fleet`. The input sells 1001 to 2008 at 0% with trade-in 0, confirms `Y`, then views invoice 50233. Generate and check:
  - the preview TOTAL is 12325.24;
  - the view shows `Discounts 0.00` (unsigned), then `Fleet discount -419.80`, then TOTAL 12325.24;
  - `expected-data/invoices.dat` has a new last row ending `|419.80` (with `disc` 419.80), and the 3 old rows end with `|0.00`;
  - 1001 has status 2.

  (US3 #1, #2)
- [X] T024 [P] [US3] Create `src/dealerdesk/tests/e2e/scenarios/27-fleet-sell-trade-pct/` with `fixture` = `fleet`. The input sells 1012 to 2002 (trade and fleet) at 5% with trade-in 2000, confirms, and views the invoice. Generate and check:
  - the view's `Discounts` = trade + salesman (excluding fleet);
  - `Fleet discount` = 485.82;
  - the totals match the preview.
- [X] T025 [P] [US3] Create `src/dealerdesk/tests/e2e/scenarios/29-existing-invoices/` (sample fixture). The input views 50230, 50231 and 50232, then lists invoices. Generate it, then `diff` the invoice-view part against `20-invoice-view/expected.out` and confirm it is identical (US3 #3, SC-004)
- [X] T026 [US3] Create `src/dealerdesk/tests/compat.sh` and a `test-compat` target in `src/dealerdesk/Makefile` (research R-7). The script:
  1. `git archive a2ba708 '*.c' '*.h'` into a temp dir, then build `old` there with `cc -o old *.c -lm`.
  2. Copy `tests/e2e/scenarios/26-fleet-sell-invoice/expected-data/*.dat` into a temp `data/`, replacing `<TODAY>` with a fixed date.
  3. Run the old binary with input `5 1 "" 2 50233 "" 0 0`.
  4. Assert exit status 0, that the invoice list includes 50233, and that its view shows `TOTAL` 12325.24.
  5. Print a note that the old binary would drop field 6/12 on its next save (E-05).

  It fails clearly if git history is unavailable

**Checkpoint**: all three stories are complete.

---

## Phase 6: Approved visible changes and re-baseline of spec 001 goldens

**Purpose**: add the Help line, then re-baseline exactly the 001 goldens named in FR-007 and FR-009, and prove the diff contains only the approved changes (E-07, research R-6).

- [X] T027 In `src/dealerdesk/main.c` `help_screen`, insert `printf("  - Fleet customers receive an automatic 4%% fleet discount.\n");` directly after the trade-discount line, and leave the other lines unchanged (D-20)
- [X] T028 In `src/dealerdesk/tests/e2e/scenarios/11-customer-add/input`, insert an `N` answer right after each trade answer (`N`, `Y` and `N` lines), so each Add Customer gets a fleet answer
- [X] T029 Run `make test`. The failures must be **exactly**:
  - output: 02-invalid-option, 04-help, 10-customers-list, 11-customer-add, 12-customer-find, 21-missing-data, 22-capacity-full;
  - data: 11-customer-add (customers.dat), 16-sell-retail and 17-sell-trade (invoices.dat).

  Any other failure is a regression: fix the code, not the golden
- [X] T030 Run `FORCE=1 make golden SCENARIO=<name>` for each scenario listed in T029, one at a time
- [X] T031 Create `src/dealerdesk/tests/e2e/check-rebaseline.sh` (not part of `make test`). For each re-baselined file, it takes `git show fcff60d:<path>` as the old version and the working file as the new version. It deletes the approved additions from the new version:
  - the line `  4. Set fleet status`;
  - the substring `  Fleet customer (Y/N)? `;
  - the line `  - Fleet customers receive an automatic 4% fleet discount.`;
  - a trailing `|0` on customers.dat lines;
  - a trailing `|0.00` on invoices.dat lines.

  Then it `cmp`s the result with the old version. It prints OK or the diff per file and exits non-zero on any mismatch. Run it: everything must be OK
- [X] T032 Run `make clean && make test`, `make test-mutation` and `make test-compat` in WSL: all must be green. Then check that `git -C src/dealerdesk status --porcelain` shows no `data/` changes

**Checkpoint**: the suite is green, and the 001 goldens changed only as approved.

---

## Phase 7: Polish & Cross-Cutting Concerns

- [X] T033 [P] Update `specs/000-baseline/spec.md` for v2.4. These BR items change:
  - BR-002: v2.4 header.
  - BR-020: `, Fleet` marker.
  - BR-021: fleet prompt.
  - BR-023: the new Set fleet status action; still no edit or delete.
  - BR-030 to BR-034: fleet line on quotes and previews; the stored fleet amount.
  - BR-040 to BR-047: the new pricing order with the fleet step.
  - BR-050: 12th field; `disc` includes fleet.
  - BR-052: fleet invoice view.
  - BR-060: help line.
  - BR-070: loaders keep empty fields; 6th/12th fields.

  Also add a fleet worked-example row.
- [X] T034 [P] Update `docs/architecture.md`:
  - the structs, plus the `customers.dat` (6 fields) and `invoices.dat` (12 fields) rows;
  - loader behaviour (`next_field`, empty fields kept, stock still uses `strtok`);
  - `calc_quote` signature, the Set fleet status action, version 2.4;
  - the new fixtures and `make test-compat`;
  - mark 002 done in "Planned structural changes".
- [X] T035 [P] Append a dated `## Fleet discount (v2.4, added 2026-11-…)` section at the bottom of `src/dealerdesk/README.md`. It covers what fleet does, how to flag a customer, and the pricing order. Do not edit the existing text
- [X] T036 [P] Append a dated `v2.4` section at the bottom of `src/dealerdesk/RELEASE.txt` with:
  - a "back up `data/` before deploying" step, before step 1 of the existing procedure;
  - a rollback warning: the old binary drops fleet flags and invoice fleet amounts on its first save, so after a rollback, flags must be re-set (E-05, FR-014).
- [X] T037 [P] Create `src/dealerdesk/NOTES.md` (new file, F-08), with a "Rollback risk (v2.4)" section stating the E-05 data loss and the backup step. Leave room for the final notes of the whole effort, at most two pages
- [X] T038 [P] Append assumption rows D-21 onwards to `src/dealerdesk/assumptions.md`, one per research decision R-2 to R-7 that is not already covered, with status `confirmed`. Update `src/dealerdesk/tests/README.md`: the new fixtures (`fleet`, `legacy-gaps`), scenarios 23 to 29 in the coverage table, `make test-compat`, and `check-rebaseline.sh`
- [X] T039 Run [quickstart.md](quickstart.md) steps 1 to 6 in WSL from `src/dealerdesk`, confirm each expected result, and run `git checkout -- data` if anything touched it. Append "Implementation notes" at the bottom of this tasks.md, covering deviations, findings and the T001 baseline result

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: none. T003 and T004 can run in parallel; T002 is needed before any fixture is used.
- **Foundational (Phase 2)**: depends on Setup. T005 → T006 → T007 (same file `dbfile.c` for T006/T007); T008 → T009; T010 can run in parallel; T011 runs last.
- **US1 (Phase 3)**: depends on Phase 2.
- **US2 (Phase 4)**: depends on Phase 2. It shares `main.c` with US1, so its `main.c` tasks run after T012 (`type_txt`).
- **US3 (Phase 5)**: depends on Phase 2. Fleet sales need a fleet customer, which fixture `fleet` provides, so it does not depend on US2.
- **Phase 6**: depends on US1 to US3 (all `main.c` changes done), so the re-baseline happens once.
- **Polish (Phase 7)**: depends on Phase 6.

### User Story Dependencies

- US1, US2 and US3 are independently testable through fixtures, but they edit the same `main.c`, so their `main.c` tasks run in sequence: T012 → T013 → T016 → T017 → T021 → T022 → T027.

### Parallel Opportunities

- T003 ∥ T004.
- T010 ∥ T005 to T009.
- T014 ∥ T012 and T013 (a different file).
- Scenario tasks: T018 ∥ T019 ∥ T020, and T023 ∥ T024 ∥ T025. Build once before generating, so there is no rebuild race.
- T033 ∥ T034 ∥ T035 ∥ T036 ∥ T037 ∥ T038.

---

## Parallel Example: User Story 2

```bash
# after T017 is built (make):
Task: "T018 scenarios/23-fleet-set-status"
Task: "T019 scenarios/24-fleet-add-customer"
Task: "T020 scenarios/28-legacy-gaps"
```

---

## Implementation Strategy

### MVP first (User Story 1)

1. Phases 1 and 2: fixtures, data fields, pricing, v2.4.
2. Phase 3 (US1): fleet quotes are correct and visible.
3. **Stop and validate**: `make unit` and scenario 25. The ticket's first acceptance criterion is met.

### Incremental delivery

1. Add US2: staff can flag customers (needed in real use, since sample data has no flags).
2. Add US3: invoices and compatibility.
3. Phase 6: one reviewed re-baseline.
4. Phase 7: docs.
5. Commit the submodule first, then the workspace pointer, and only when the user asks.

---

## Notes

- Expected values come from the running code (`--print`, `make golden`) and are checked
  against research.md R-1. Never type expected values by hand.
- If any non-fleet output changes beyond the FR-007 list, it is a bug: fix the code.
- The C1 gap from spec 001 (orphan invoice `(unknown)` / `?`) is still out of scope here.

## Implementation notes (2026-11-16)

All 39 tasks are done. They were validated in WSL (gcc 15.2, GNU make 4.4).

**Results:**
- T001 baseline at `fcff60d`: 30 unit cases and 22 scenarios green, mutation killed.
- Final: `make test` gives unit 40/40 and e2e 29/29.
- `make test-mutation`: killed. `make test-compat`: OK.
- `check-rebaseline.sh`: all 10 re-baselined files differ only by approved changes.
- `make CFLAGS='-std=c89 -pedantic -Wall'`: 0 warnings (analysis G1).
- Sample `data/` is untouched.

**Findings and deviations:**
- **`make golden` needs `FORCE=1` for new scenarios too.** The sources now differ from
  `a2ba708`, so the guard refuses every regeneration (D-25). Every FORCE run was
  reviewed.
- **T002**: fixture names may contain digits and `-`. The 001 runner's `tr -cd 'a-z'`
  would have turned `legacy-gaps` into `legacygaps`.
- **T004 (analysis U1)**: the `legacy-gaps` invoices are 50300 (11 fields, vehicle 1002,
  customer 2001) and 50301 (fleet, vehicle 1002, customer 2002), with `next_ids`
  invoice=50302.
- **T011**: the failures were exactly 11, 16 and 17 (data), as planned.
- **T020**: the 28-legacy-gaps golden was regenerated after T022, so its fleet invoice
  shows the new view.
- **T029**: the failures were exactly the 9 planned scenarios (7 output, 3 data; 11 has
  both).
- **T034**: a first sed attempt mangled `docs/architecture.md` (`\|` is alternation in
  GNU sed). It was restored from the last commit and edited by hand. No content was lost.
- **D-22 side effect**: a line made only of `|` separators now loads as an id-0 record,
  where it was skipped before. No real file has one.
- **Fixture `fleet`**: invoices 50231/50232 refer to vehicles 1012/1001, which are in
  stock there. This is harmless (D-24).
- **Still open (out of scope)**: the orphan-invoice coverage gap from spec 001
  (F-20).
