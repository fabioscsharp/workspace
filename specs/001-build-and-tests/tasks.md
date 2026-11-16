---

description: "Task list for 001: reliable build and characterization tests for DealerDesk"
---

# Tasks: Reliable build and characterization tests for DealerDesk

**Input**: Design documents from `specs/001-build-and-tests/` ([plan.md](plan.md),
[spec.md](spec.md), [research.md](research.md), [data-model.md](data-model.md),
[contracts/](contracts/), [quickstart.md](quickstart.md))

**Prerequisites**: work in `src/dealerdesk` on branch `feature/page-tests-ticket-142`, from
the submodule root, inside WSL (`make`, gcc). Run `git checkout -- data` after any manual
run. Do not commit unless asked (CLAUDE.md rule 7).

**Tests**: this feature *is* the test suite. The "tests" are the deliverable, so they appear
as implementation tasks under User Story 2, not as a separate test-first section.

**Organization**: grouped by user story. US1 (build) is the MVP. US2 (characterization
tests) needs the binary from US1.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: can run in parallel (different files, no dependency on an unfinished task)
- **[Story]**: US1 or US2 (spec.md user stories)
- Paths are relative to the workspace root. Application paths start with `src/dealerdesk/`.

## Hard rules for every task

- Do **not** modify `main.c`, `calc.c`, `dbfile.c`, `screen.c`, `*.h` or `data/` (FR-002,
  F-13, constitution I/III).
- Existing docs in `src/dealerdesk` (README.md, BRIEF.md, RELEASE.txt, TICKET-142.md) are
  append-only: add a dated section at the bottom, never edit existing text (constitution IV).
- Never hand-edit a golden file. Generate it with `make golden SCENARIO=<dir>`, then review it.

---

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: confirm the baseline and create the empty layout from plan.md.

- [X] T001 Verify the application sources and sample data equal commit `a2ba708`: run `git -C src/dealerdesk diff --quiet a2ba708 -- '*.c' '*.h' data` and stop if it reports differences (goldens must come from the original code, F-04)
- [X] T002 [P] Create `src/dealerdesk/.gitignore` with `dealerdesk`, `*.o`, `tests/unit/test_calc` (no `.gitignore` exists today)
- [X] T003 [P] Create the directory layout `src/dealerdesk/tests/unit/`, `src/dealerdesk/tests/e2e/fixtures/empty/` (add an empty `.gitkeep`), `src/dealerdesk/tests/e2e/scenarios/`

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: none beyond Setup. The build entry point is itself User Story 1, and every US2
task depends on it (see Dependencies).

**Checkpoint**: Setup done, so US1 can start.

---

## Phase 3: User Story 1 - One command builds the app (Priority: P1) 🎯 MVP

**Goal**: `make` builds `./dealerdesk` on Linux/macOS/WSL with gcc or clang, and
`make clean` removes it (FR-001, [contracts/make-targets.md](contracts/make-targets.md)).

**Independent Test**: on a clean clone, `make` produces `dealerdesk`.
`printf '0\n' | ./dealerdesk` prints the main menu then `Goodbye.` and exits 0.

- [X] T004 [US1] Create `src/dealerdesk/Makefile` with `CC ?= cc`, `CFLAGS ?= -Wall`, `LDLIBS = -lm`, and `SRCS = main.c calc.c dbfile.c screen.c` listed explicitly (not `*.c`). Default target `all` builds `dealerdesk`. `clean` removes `dealerdesk`, `*.o` and `tests/unit/test_calc`. Declare `.PHONY`. No `-Werror` (research R-8)
- [X] T005 [US1] Validate the build in WSL: `make clean && make` succeeds (warnings allowed). `printf '0\n' | ./dealerdesk; echo $?` shows `Goodbye.` and `0`. `CC=gcc make` also works. Record any compiler warnings (count and file) for T040
- [X] T006 [US1] Append a dated section `## Build and test (added 2026-11-15)` at the bottom of `src/dealerdesk/README.md`. It documents `make`, `make test`, `make clean`, and states that this supersedes the two older build commands (B-10). Do not touch existing lines

**Checkpoint**: US1 is complete and independently usable (MVP).

---

## Phase 4: User Story 2 - Characterization tests pin today's behaviour (Priority: P1)

**Goal**: `make test` runs unit tests for pricing and end-to-end golden tests for every
screen, on temp copies of data, and passes on the unmodified code (FR-003 to FR-008,
SC-002 to SC-005).

**Independent Test**: `make test` exits 0 on an unmodified checkout. Afterwards
`git -C src/dealerdesk status --porcelain` lists nothing under `data/`.
`make test-mutation` reports that the mutated suite failed.

### Unit tests for pricing (FR-003, SC-002)

- [X] T007 [US2] Create `src/dealerdesk/tests/unit/test_calc.c`. It includes `../../calc.h`, has a local `CHECK` helper that compares `%.2f` strings, and holds a case table matching the data-model "Unit case": name, list, cust_type, disc_pct, tradein, plus 10 expected `%.2f` strings for list, trade_disc, disc_pct, disc, net, delivery, vat, reg_fee, tradein, total. Add a separate `rnd2` table of (input, expected `%.2f`). Add a `--print` mode that prints the actual values in table syntax instead of checking them. It exits non-zero on any mismatch and prints `unit: N passed, M failed`
- [X] T008 [US2] Fill the cases from research R-3 in `src/dealerdesk/tests/unit/test_calc.c`: types R and T and one other char (behaves as R); pct −5, 0, 5, 8, 10, 15, 15.01, 40; trade-in 0 and >0; list 0 and a very large list; `rnd2` half-penny inputs 0.125, 2.675, 1.005, 10.045; and the three baseline invoices (9295.00 R 5% trade-in 1500 → total 9331.30; 13495.00 T 0% → 14809.60; 10495.00 R 8% → 11821.48). Capture the expected strings by running `--print` against the unchanged `calc.c`, paste them in, and confirm that the three invoice cases match `specs/000-baseline/spec.md` worked examples
- [X] T009 [US2] Add a `unit` target to `src/dealerdesk/Makefile`: build `tests/unit/test_calc` from `tests/unit/test_calc.c calc.c` with `$(LDLIBS)`, then run it

### E2E harness (FR-004, FR-005, FR-005a, FR-008)

- [X] T010 [P] [US2] Create `src/dealerdesk/tests/e2e/normalise.sh` (POSIX sh, filter stdin to stdout, run dates passed as arguments). Apply the rules in this order: (1) delete `\033[2J\033[H`; (2) `DEALERDESK v[0-9]+\.[0-9]+` → `DEALERDESK vX.Y`; (3) each given run date `YYYY-MM-DD` → `<TODAY>`. Make no other changes (data-model "Normalisation rule", research R-5)
- [X] T011 [P] [US2] Create `src/dealerdesk/tests/e2e/fixtures/make-full.sh`. It takes a target directory and writes `stock.dat` with 500 vehicles, `customers.dat` with 500 customers, `invoices.dat` with 2000 invoices, and a matching `next_ids.dat`, all in the exact formats from `docs/architecture.md` (money `%.2f`, `|` separators)
- [X] T012 [US2] Create `src/dealerdesk/tests/e2e/run.sh` following [contracts/scenario-format.md](contracts/scenario-format.md). It takes the binary path and handles each `scenarios/NN-*` dir in name order:
  1. `mktemp -d` with a `trap` cleanup.
  2. Build the fixture from the `fixture` file (`sample` = copy of repo `data/`, the default; `empty` = `fixtures/empty`; `full` = `make-full.sh`).
  3. Record `d1`, run `(cd "$tmp" && BIN < input)`, record `d2`.
  4. Normalise the output and every data file.
  5. Compare the output with `diff -u expected.out`.
  6. Compare the data file-name set and each file with `cmp`, against `expected-data/` or, if that is absent, the normalised fixture.

  It prints PASS/FAIL per scenario and a summary, and exits non-zero on any failure. It honours `SCENARIO` (glob). `--update` writes the goldens instead, and it refuses unless `git diff --quiet a2ba708 -- '*.c' '*.h' data` succeeds or `FORCE=1` (research R-7). It never writes to the repo `data/`
- [X] T013 [US2] Add the `e2e`, `test` (= `all unit e2e`) and `golden` targets to `src/dealerdesk/Makefile`, passing `SCENARIO` through. Check that `make test` never calls `--update` ([contracts/make-targets.md](contracts/make-targets.md))

### Scenarios (one per screen/path; plan.md "Coverage map")

For each scenario, do the following, then read `expected.out` against `main.c` to confirm it shows the intended path:

1. Write `input`, deriving the prompts from `src/dealerdesk/main.c`: main menu ~l.517, stock menu ~l.186, customer menu ~l.284, sell ~l.370, invoice menu ~l.478. Prompts like "Press ENTER" need an empty line.
2. Exit with `0`s.
3. Add `fixture` if it is not `sample`.
4. Run `make golden SCENARIO=<dir>`.

Data-writing scenarios must end up with `expected-data/`.

- [X] T014 [P] [US2] `src/dealerdesk/tests/e2e/scenarios/01-main-menu-exit/`: main menu then `0` → "Goodbye." (BR-001, BR-002)
- [X] T015 [P] [US2] `src/dealerdesk/tests/e2e/scenarios/02-invalid-option/`: an invalid option in the main menu and in each submenu → "Invalid option" (BR-001)
- [X] T016 [P] [US2] `src/dealerdesk/tests/e2e/scenarios/03-eof-exit/`: input ends inside a submenu prompt with no exit keys; the app exits cleanly (screen.c `read_line`)
- [X] T017 [P] [US2] `src/dealerdesk/tests/e2e/scenarios/04-help/`: the Help screen (option 9), with its stale text pinned as-is (BR-060)
- [X] T018 [P] [US2] `src/dealerdesk/tests/e2e/scenarios/05-stock-list/`: list stock and list all vehicles (BR-010)
- [X] T019 [P] [US2] `src/dealerdesk/tests/e2e/scenarios/06-stock-add/`: add a vehicle; `expected-data/` shows the new `stock.dat` line with id 1013 and `next_ids.dat` with vehicle=1014 (BR-011, BR-070)
- [X] T020 [P] [US2] `src/dealerdesk/tests/e2e/scenarios/07-stock-find-plate/`: `LK19 XRT` found, `YF20 HBN` not found (case-sensitive, B-08), `yf20 hbn` found, and an unknown plate (BR-012)
- [X] T021 [P] [US2] `src/dealerdesk/tests/e2e/scenarios/08-stock-change-price/`: a valid new price on an in-stock vehicle (writes), price 0 and a negative price rejected ("Price not…"), and an unknown id (BR-013)
- [X] T022 [P] [US2] `src/dealerdesk/tests/e2e/scenarios/09-stock-reserve-release/`: reserve 1002, release 1006, try a SOLD vehicle (1001) and get it refused; writes `stock.dat` (BR-014)
- [X] T023 [P] [US2] `src/dealerdesk/tests/e2e/scenarios/10-customers-list/`: list customers with the "Trade" label for type T, and the customer menu offers no edit/delete (BR-020, BR-023)
- [X] T024 [P] [US2] `src/dealerdesk/tests/e2e/scenarios/11-customer-add/`: add a retail customer, a trade customer, and an empty name (rejected); writes `customers.dat` and `next_ids.dat` (BR-021)
- [X] T025 [P] [US2] `src/dealerdesk/tests/e2e/scenarios/12-customer-find/`: a substring hit (`Motors`) and a case-sensitive miss (`motors`) (BR-022)
- [X] T026 [P] [US2] `src/dealerdesk/tests/e2e/scenarios/13-quote-retail/`: retail quotes on an in-stock vehicle at 0/5/10/15 % and with a trade-in; no data change (BR-030, BR-031, BR-032, BR-040 to BR-047)
- [X] T027 [P] [US2] `src/dealerdesk/tests/e2e/scenarios/14-quote-trade/`: trade customer 2002 quotes with pct 20 (clamped to 15) and −5 (clamped to 0) (BR-040, BR-041)
- [X] T028 [P] [US2] `src/dealerdesk/tests/e2e/scenarios/15-quote-errors/`: unknown vehicle id, unknown customer id, and a SOLD vehicle; record the actual messages (BR-030)
- [X] T029 [P] [US2] `src/dealerdesk/tests/e2e/scenarios/16-sell-retail/`: sell an in-stock vehicle to retail customer 2001, then confirm `Y`; `expected-data/` has a new invoice 50233 dated `<TODAY>`, the vehicle status is 2, and `invoice=50234` (BR-033, BR-034, BR-050, FR-005a)
- [X] T030 [P] [US2] `src/dealerdesk/tests/e2e/scenarios/17-sell-trade/`: sell to trade customer 2005 with a trade-in, confirm, then view the new invoice in the same session (US2 #3)
- [X] T031 [P] [US2] `src/dealerdesk/tests/e2e/scenarios/18-sell-cancel/`: preview, then `N` → "Sale cancelled."; no `expected-data/` (data must be unchanged) (BR-033)
- [X] T032 [P] [US2] `src/dealerdesk/tests/e2e/scenarios/19-invoices-list/`: list invoices with the count (BR-051); also cover "(unknown)" if the sample data allows it, otherwise note in tests/README.md that it is covered by 21/22
- [X] T033 [P] [US2] `src/dealerdesk/tests/e2e/scenarios/20-invoice-view/`: view 50230, 50231 and 50232 (totals 9331.30, 14809.60, 11821.48) and a missing number → "Invoice not found." (BR-052, FR-006, US2 #2)
- [X] T034 [P] [US2] `src/dealerdesk/tests/e2e/scenarios/21-missing-data/`: `fixture` = `empty`; list stock, list customers, list invoices, add one vehicle → id 1000; `expected-data/` holds only the files the app wrote (BR-070)
- [X] T035 [P] [US2] `src/dealerdesk/tests/e2e/scenarios/22-capacity-full/`: `fixture` = `full`; try adding a vehicle, adding a customer and selling, and get "Stock file is full!", "Customer file is full!" and "Invoice file is full!" (BR-081)

### Mutation check and suite docs (SC-003, SC-004)

- [X] T036 [US2] Create `src/dealerdesk/tests/mutation.sh` and a `test-mutation` target in `src/dealerdesk/Makefile`. The script copies the sources and tests to `mktemp -d`, uses `sed` to change the VAT constant `0.20` to `0.21` in the copy of `calc.c` (failing if `sed` changed nothing), and runs `make test` in the copy. It exits 0 only if that run fails, and leaves the working tree untouched (research R-8)
- [X] T037 [P] [US2] Create `src/dealerdesk/tests/README.md`. It covers:
  - how to run the suite (`make test`, `make unit`, `make e2e`, `SCENARIO=`, `make test-mutation`);
  - how to add a scenario (the scenario-format rules, never hand-edit goldens);
  - the normalisation rules;
  - a coverage table mapping every `BR-` item from `specs/000-baseline/spec.md` (BR-001 … BR-082) to a scenario or unit case, with BR-080 marked "not testable: no locking, nothing to assert" and BR-082 marked "superseded by this feature".
- [X] T038 [US2] Validate US2 in WSL:
  1. `time make test` exits 0 in well under 2 min (SC-001).
  2. Run it twice; the second run is also green, so nothing leaks between runs.
  3. `git -C src/dealerdesk status --porcelain` shows no `data/` changes.
  4. `make test-mutation` reports that the mutated suite failed.
  5. Temporarily change one byte in one `expected.out` and confirm `make e2e` fails, then revert it.

**Checkpoint**: US1 and US2 are complete. The suite is the safety net for 002 and 003.

---

## Phase 5: Polish & Cross-Cutting Concerns

**Purpose**: keep the knowledge files current (CLAUDE.md rules 2 and 4) and validate.

- [X] T039 [P] Update `docs/architecture.md`:
  - "Build and run": `make`, `make test`, the `tests/` layout, explicit SRCS, no `-Werror`.
  - "Planned structural changes": mark 001 as done.
- [X] T040 [P] Update `specs/000-baseline/spec.md` BR-082: a Makefile build and characterization tests now exist; CI and release are still manual until 003. Mention any compiler warnings found in T005
- [X] T041 [P] Append entries F-14 onwards to section F of `src/dealerdesk/assumptions.md`, one per research decision R-1 to R-8 that is not already covered by F-10 to F-13. Status `confirmed`. Append only; existing rows may change status only
- [X] T042 Run the [quickstart.md](quickstart.md) steps 1 to 6 in WSL from `src/dealerdesk` and confirm each expected result, then `git checkout -- data` if anything touched it

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: none. T001 must pass before any golden is generated.
- **Foundational (Phase 2)**: empty.
- **US1 (Phase 3)**: depends on Setup.
- **US2 (Phase 4)**: depends on US1 (T004, which provides the binary and the Makefile it extends).
- **Polish (Phase 5)**: depends on US2 (T040 needs the T005 warnings; T039 and T041 describe the final layout).

### User Story Dependencies

- **US1 (P1)**: independent; it is the MVP.
- **US2 (P1)**: needs the US1 binary. It is testable on its own with `make test`.

### Within User Story 2

- T007 → T008 → T009 (unit chain).
- T010, T011 → T012 → T013 (harness chain).
- T013 → T014 to T035 (scenarios need `make golden`).
- T014 to T035 → T036, T037 → T038.
- The `Makefile` is edited in T004, T009, T013 and T036, so those run in sequence, never in parallel.

### Parallel Opportunities

- T002 ∥ T003.
- T007 to T009 (unit) ∥ T010 to T012 (harness): different files.
- T014 to T035: each scenario is its own directory, so all are [P] once T013 is done.
- T037 ∥ T036.
- T039 ∥ T040 ∥ T041.

---

## Parallel Example: User Story 2

```bash
# Unit and harness tracks together:
Task: "T007 Create tests/unit/test_calc.c"
Task: "T010 Create tests/e2e/normalise.sh"
Task: "T011 Create tests/e2e/fixtures/make-full.sh"

# After T013, all scenarios together:
Task: "T018 scenarios/05-stock-list"
Task: "T026 scenarios/13-quote-retail"
Task: "T029 scenarios/16-sell-retail"
Task: "T033 scenarios/20-invoice-view"
```

---

## Implementation Strategy

### MVP First (User Story 1 only)

1. Phase 1 Setup (T001 to T003).
2. Phase 3 US1 (T004 to T006).
3. **Stop and validate**: `make` plus the `Goodbye.` smoke test. This is a usable reliable build.

### Incremental Delivery

1. Setup plus US1, which gives the build.
2. US2 unit track (T007 to T009): pricing is pinned to the penny.
3. US2 harness plus scenarios (T010 to T035): every screen is pinned.
4. Mutation, README and validation (T036 to T038), then Polish (T039 to T042).
5. Commit in the submodule first, then the workspace pointer, but only when the user asks.

---

## Notes

- About 22 scenarios cover all screens (the "pages"). Exact keystrokes and messages come
  from the running code, not from README or help text (constitution II).
- If a scenario reveals behaviour that contradicts the baseline spec, do **not** change the
  code. Record it in `src/dealerdesk/assumptions.md` and flag it to the user.
- Ticket work (TICKET-142, spec 002) starts only after this suite is green.

## Implementation notes (2026-11-16)

All 42 tasks are done. They were validated in WSL (gcc 15.2, GNU make 4.4):

- `make test`: 30 unit cases and 22 scenarios pass in about 3 s.
- `make test-mutation`: the mutation is caught (unit 21 failed, e2e 5 failed).
- `data/` is untouched.

Deviations from the task text:

- **Added `src/dealerdesk/.gitattributes`** (`* text=auto eol=lf`). It was not in the
  plan. This Windows checkout uses `core.autocrlf=true`, which made the data and scripts
  CRLF (F-18).
- **T003/T012**: there is no committed `fixtures/empty/.gitkeep`. The runner creates an
  empty `data/` itself, so no placeholder file is ever compared (analysis finding U2, F-16).
- **T012**: the runner also appends `[exit status: N]` to the output, so a crash or
  non-zero exit fails the scenario (F-15).
- **T036**: the mutation copy includes `Makefile`, `data/` and `tests/`, and it first
  checks that the unmodified copy passes (analysis finding U1, F-17).
- **T014–T035**: the binary was built once before generating the goldens, so there is
  no parallel-rebuild race (analysis finding U3).
- **T008**: one case shows `rnd2` rounding an exact half-penny down
  (2564.325 → 2564.32). It is pinned as-is (F-19).
- **T032**: the `(unknown)` customer and `?` vehicle paths are not covered (analysis
  finding C1, F-20, `tests/README.md` "Known gap").
