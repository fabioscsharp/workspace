# Implementation Plan: Reliable build and characterization tests for DealerDesk

**Branch**: `001-build-and-tests` (work branch in `src/dealerdesk`:
`feature/page-tests-ticket-142`) | **Date**: 2026-11-15 |
**Spec**: [spec.md](spec.md)

**Input**: Feature specification from `specs/001-build-and-tests/spec.md` (clarified
2026-11-15)

## Summary

Add a `Makefile` (`make` builds `dealerdesk`, `make test` runs everything) and a
characterization test suite that pins today's behaviour without changing any application
source file:

- **Unit tests** (`tests/unit/test_calc.c`, plain C, no framework) link `calc.c` directly
  through the existing `calc.h` and check every pricing path of `calc_quote` and `rnd2`
  against values captured from the unmodified code.
- **End-to-end tests** (`tests/e2e/`, POSIX `sh` + `diff`/`cmp`/`sed`) feed a scripted
  keystroke file to the real binary, running in a temporary copy of a data fixture. They
  compare normalised screen output with a committed golden file. Afterwards they compare
  every `.dat` file byte for byte with a committed golden data set.
- **Mutation check** (`make test-mutation`) copies the sources to a temp dir, changes one
  pricing constant, and asserts that the suite fails (SC-004).

All goldens are generated from the code at `a2ba708`. The application sources and
`data/` are identical at the current submodule HEAD (`1b98799`; verified with
`git diff a2ba708 HEAD -- '*.c' '*.h' data`, which is empty).

## Technical Context

**Language/Version**: C (application: conservative C89, B-11; unit test code: C, any
standard gcc/clang accepts). Harness: POSIX `sh`.

**Primary Dependencies**: none new. Build needs `make` and gcc or clang plus `libm`. Tests
need only POSIX tools (`sh`, `diff`, `cmp`, `sed`, `awk`, `mktemp`, `date`), which
Linux, macOS and WSL already ship. No Python and no test framework (constitution VII).

**Storage**: flat files in `./data` (`stock.dat`, `customers.dat`, `invoices.dat`,
`next_ids.dat`). Tests only ever touch a temporary copy.

**Testing**: plain-C unit test binary plus golden-file e2e runner, both run by `make test`.

**Target Platform**: Linux, macOS and WSL with gcc or clang (F-02). Developed and
verified on WSL Ubuntu (gcc 15.2, GNU make 4.4). clang is not installed here, so the
clang build is checked in CI (spec 003).

**Project Type**: interactive terminal (CLI) application driven through stdin.

**Performance Goals**: build plus full suite in under 2 minutes from a clean clone
(SC-001). The expected time is a few seconds.

**Constraints**: zero application source changes (FR-002, F-13). `data/` stays
untouched (FR-005, C-13). No `-Werror` (warnings reported only). Tests must not hang on
EOF; stdin is always a file, so the app exits through `read_line`.

**Scale/Scope**: 4 application C files (884 lines). 1 main menu plus 3 submenus, about
20 screens (BR-001 to BR-070). About 22 e2e scenarios and about 30 unit cases.

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Principle | Check | Result |
|-----------|-------|--------|
| I. Behaviour preservation | No application source change; the binary is built with the same compiler command (warning flags only). Goldens come from the original code. SC-005 is asserted by the suite itself. | PASS |
| II. Code is the source of truth | Expected values are captured from running code, not from README or help text. Stale help text is pinned as-is (BR-060). | PASS |
| III. Backward-compatible data | Data files are only read and written in temp copies. Byte-for-byte data goldens (FR-005a) make any format drift fail. | PASS |
| IV. Docs append-only | README gets a new dated section at the bottom only (build and test commands). New docs are new files (`tests/README.md`). | PASS |
| V. Assumptions written down | F-10 to F-13 were added during clarify. New planning decisions R-1 to R-8 in [research.md](research.md) are recorded as F-14 onwards during implementation. | PASS |
| VI. Tested, reproducible | One command each: `make`, `make test`. Pricing is unit-tested without the terminal. | PASS |
| VII. Simplicity, minimal diffs | No new runtime or test dependency. No refactor. The only new files are `Makefile`, `.gitignore` and `tests/`, plus an appended README section. | PASS |

**Knowledge-file duties (CLAUDE.md rule 4)**: in the same change, update
`specs/000-baseline/spec.md` BR-082 (tests and a build now exist) and
`docs/architecture.md` "Build and run" (Makefile, test layout).

## Project Structure

### Documentation (this feature)

```text
specs/001-build-and-tests/
├── spec.md
├── plan.md              # this file
├── research.md          # Phase 0: decisions R-1..R-8
├── data-model.md        # Phase 1: test artefacts (scenario, golden, fixture, unit case)
├── quickstart.md        # Phase 1: how to validate the feature end to end
├── contracts/
│   ├── make-targets.md      # the Makefile interface (what CI in 003 calls)
│   └── scenario-format.md   # e2e scenario directory layout and normalisation rules
└── tasks.md             # Phase 2 (/speckit-tasks, not created here)
```

### Source Code (`src/dealerdesk`, repository root of the submodule)

```text
Makefile                      # NEW: all, test, unit, e2e, test-mutation, golden, clean
.gitignore                    # NEW: dealerdesk, *.o, tests/unit/test_calc (none exists today)
main.c calc.c dbfile.c screen.c *.h   # UNCHANGED
data/                         # UNCHANGED (sample data; also the default fixture)
README.md                     # APPEND ONLY: dated "Build and test" section at the bottom
tests/
├── README.md                 # NEW: how the suite works, how to add a scenario
├── unit/
│   └── test_calc.c           # calc_quote + rnd2 case table, exits non-zero on failure
├── e2e/
│   ├── run.sh                # runner: build check, temp dir, run, normalise, compare
│   ├── normalise.sh          # ANSI, version, run-date normalisation (sed)
│   ├── fixtures/
│   │   ├── empty/            # no .dat files (BR-070 missing file = empty table)
│   │   └── make-full.sh      # generates full tables (BR-081 capacity limits)
│   └── scenarios/
│       └── NN-<name>/        # input, expected.out, [fixture], [expected-data/]
└── mutation.sh               # SC-004: altered calc.c copy must make the suite fail
```

**Structure Decision**: a single project. The tests live in `tests/` inside the
submodule, next to the code they pin, so `make test` works from a clean clone of
`fabioscsharp/dealerdesk` alone. The `Makefile` lists the four application sources
explicitly rather than using `*.c`, so test C files can never end up in the app binary.
The resulting command matches `gcc -o dealerdesk *.c -lm` today.

## Coverage map (screens and baseline items to scenarios)

"Pages" means the app's screens. Every screen reachable from the main menu gets at least
one scenario (SC-003). Every `BR-` item maps to a test.

| Scenario (`tests/e2e/scenarios/`) | Screens / paths | BR items |
|---|---|---|
| 01-main-menu-exit | main menu, exit "Goodbye." | BR-001, BR-002 |
| 02-invalid-option | invalid option in main and each submenu | BR-001 |
| 03-eof-exit | input ends mid-menu and mid-prompt | screen.c `read_line` |
| 04-help | Help screen (stale text pinned) | BR-060 |
| 05-stock-list | List stock, list all vehicles | BR-010 |
| 06-stock-add | Add vehicle (writes) | BR-011, BR-070 |
| 07-stock-find-plate | exact hit, case-sensitive miss (`YF20 HBN`), not found | BR-012, B-08 |
| 08-stock-change-price | valid price (writes), price ≤ 0 rejected, unknown id | BR-013 |
| 09-stock-reserve-release | reserve, release (writes), SOLD refused | BR-014 |
| 10-customers-list | List customers, Trade label | BR-020, BR-023 |
| 11-customer-add | retail, trade, empty name rejected (writes) | BR-021 |
| 12-customer-find | substring hit, case-sensitive miss | BR-022 |
| 13-quote-retail | retail with 0/5/10/15 % and trade-in | BR-030..032, BR-040..047 |
| 14-quote-trade | trade customer, clamp > 15 and < 0 | BR-040, BR-041 |
| 15-quote-errors | unknown vehicle, unknown customer, SOLD vehicle | BR-030 |
| 16-sell-retail | preview, confirm Y (writes invoice, stock, ids) | BR-033, BR-034, BR-050 |
| 17-sell-trade | trade sale with trade-in (writes) | BR-033, BR-034 |
| 18-sell-cancel | confirm N: "Sale cancelled.", no data change | BR-033 |
| 19-invoices-list | list with count, "(unknown)" customer | BR-051 |
| 20-invoice-view | 50230, 50231, 50232 golden (FR-006), not found | BR-052, US2 #2 |
| 21-missing-data | `empty` fixture: empty tables, first ids default | BR-070 |
| 22-capacity-full | `full` fixture: "Stock/Customer/Invoice file is full!" | BR-081 |
| unit `test_calc` | all `calc_quote` / `rnd2` paths | BR-040..047, SC-002 |
| `test-mutation` | wrong VAT constant → suite fails | SC-004 |

Items that are not behaviour: BR-080 (no locking) is a non-feature with nothing to
assert, and it is documented in the coverage table of `tests/README.md`. BR-082 is
superseded by this feature, and the baseline is updated in the same change. Exact
keystroke scripts and screen texts are set at implementation time from the running code.

## Constitution Re-check (after Phase 1)

The design ([data-model.md](data-model.md), [contracts/](contracts/),
[quickstart.md](quickstart.md)) keeps every gate at PASS. No application file is
changed. Data is only written in temp copies and compared byte for byte. Docs are
append-only (README) or new files. There are no new dependencies. Open item for
`/speckit-tasks`: include the knowledge-file updates (baseline BR-082,
`docs/architecture.md`), the README append, and the F-14+ assumption entries as
explicit tasks.

## Complexity Tracking

No constitution violations; nothing to justify.
