# Feature Specification: Reliable build and characterization tests for DealerDesk

**Feature Branch**: `001-build-and-tests` (in `src/dealerdesk`)

**Created**: 2026-11-15 (split 2026-11-15: CI and release moved to `003-ci-and-release`)

**Status**: Draft; the version question (B-06) and compiler style (B-11) are decided
2026-11-15. Clarified 2026-11-15 (see Clarifications).

**Baseline**: characterizes `specs/000-baseline/spec.md` (every `BR-` item gets a test);
changes no `BR-` behaviour. Structure: `docs/architecture.md`.

**Related**: `003-ci-and-release` (runs these tests in CI and automates releases) depends on
this spec. `002-fleet-discount` depends on this spec for its safety net.

**Input**: BRIEF.md task 1: "Improve how changes get delivered on this codebase. You decide
what matters most and how far to take it." Today there are no tests and no reliable build.
This spec covers the part everything else needs: a repeatable build and tests that pin
today's behaviour. CI and release automation are out of scope here.

## Clarifications

### Session 2026-11-15

- Q: Is a Makefile the single build and test entry point? → A: Yes. `make` builds
  `dealerdesk` (same `gcc ... *.c -lm` command), `make test` runs the full suite.
- Q: How do tests handle the sale date stamped from the system clock (`main.c:395`)? →
  A: The harness normalises the run date (`YYYY-MM-DD`) to a placeholder in screen output
  and written data files before comparing; no app change, no fake-clock tool.
- Q: How do tests check `data/*.dat` after scenarios that change data? → A: Every data
  file is compared byte for byte with a committed golden copy (only the run date
  normalised).

## User Scenarios & Testing *(mandatory)*

### User Story 1 - One command builds the app (Priority: P1)

A developer clones the repo on Linux, macOS or WSL and builds the same binary the README
promises with one documented command, with no guesswork between the two conflicting build
commands in the README.

**Why this priority**: nothing else (tests, CI, release) is possible without a repeatable
build.

**Independent Test**: on a clean clone, the documented build command produces a
`dealerdesk` binary that starts and shows the main menu; exit with `0`.

**Acceptance Scenarios**:

1. **Given** a clean clone and gcc or clang, **When** the build command is run, **Then** a
   `dealerdesk` executable is produced with no errors.
2. **Given** the built binary and sample data, **When** it is started from the repository
   root and `0` is entered, **Then** it prints "Goodbye." and exits with status 0.

---

### User Story 2 - Characterization tests pin today's behaviour (Priority: P1)

Before anything changes, tests record what the app does now, "to the penny", so any
unintended change is caught. Pricing is covered at unit level; menus, quotes, sales and
invoices are covered end to end through the terminal.

**Why this priority**: the requirement "everything that exists today must keep working
exactly as it does now" can only be proved with tests that were generated from the old code.

**Independent Test**: run the test command on an unmodified checkout: all tests pass and
the committed sample data is untouched afterwards.

**Acceptance Scenarios**:

1. **Given** the pricing function, **When** quotes are computed for retail and trade
   customers with discounts 0, 5, 10, 15, >15 and <0, with and without trade-in, **Then**
   every field equals the value produced by the original code.
2. **Given** a scripted session that views the sample invoices, **Then** the output equals
   the committed golden output for invoices 50230, 50231 and 50232.
3. **Given** a scripted sale of an in-stock vehicle to retail and to trade customers,
   **When** the invoice is viewed, **Then** the totals match the golden values and the
   vehicle is marked SOLD, the invoice counter advances, and the data files are
   written in the same format as before.
4. **Given** a tests run, **Then** tests work on a temporary copy of `data/` and
   `git status` in the repo is clean afterwards.
5. **Given** a deliberately wrong pricing constant, **When** the tests run locally,
   **Then** they fail.

---

### Edge Cases

- Tests run from a directory other than the repo root (the app reads `./data`): the test
  harness always sets the working directory.
- ANSI clear-screen codes in the output (`\033[2J\033[H`) must not break comparisons.
- The version string in the header (`v2.3`, becomes `v2.4` in spec 002) is normalised by
  tests so the bump does not break them.
- A sale stamps the invoice with today's date (`main.c:395`): tests replace the run date
  with a placeholder in output and in written data files before comparing; existing
  invoice dates in sample data are compared as-is.
- Input ends unexpectedly (EOF): the app exits with status 0 (`read_line`), tests must
  not hang.
- Floating-point rounding at half-cent boundaries (`rnd2`): expected values come from
  the original implementation, not recomputation.
- No `-Werror` on the legacy code: warnings are reported, not fatal, until cleaned up.
- Tests are developed and run on Linux/WSL; WSL Ubuntu with gcc and make is available here.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: The repo MUST provide a `Makefile` whose default target (`make`) builds
  `dealerdesk` on Linux/macOS/WSL with gcc or clang, from the same sources and libraries
  as `gcc -o dealerdesk *.c -lm`. Only warning flags (for example `-Wall`) may be added,
  and `CC`/`CFLAGS` can be overridden (see assumptions F-01, B-10).
- **FR-002**: The build and tests MUST NOT change application behaviour or require any
  change to existing application files. `calc_quote` and `rnd2` are already public in
  `calc.h`, so unit tests link `calc.c` unchanged (F-13).
- **FR-003**: There MUST be unit tests for `calc_quote` and `rnd2` covering customer
  types R and T, discount clamping, trade-in, and rounding, with expected values captured
  from the unmodified code (F-04).
- **FR-004**: There MUST be end-to-end tests driving the binary via stdin that cover:
  stock list, add vehicle, find by plate, change price, reserve/release, customers list/
  add/find, quote, sell, invoice list and view, help, invalid option, EOF.
- **FR-005**: Tests MUST run against a temporary copy of the data and leave the working
  tree clean (C-13).
- **FR-005a**: After every end-to-end scenario that writes data (add vehicle, change
  price, reserve/release, add customer, sell), each `data/*.dat` file in the temporary
  copy MUST equal its committed golden copy byte for byte, with only the run date of new
  sales normalised.
- **FR-006**: There MUST be a golden-file check that sample data loads and prints
  unchanged, proving "existing data files load without manual conversion".
- **FR-007**: The whole suite MUST run with `make test` (builds first if needed; suitable
  to be called from CI later, see `003-ci-and-release`).
- **FR-008**: Tests MUST normalise the version string; this feature keeps the on-screen
  header at `v2.3` (it changes no behaviour). The bump to `v2.4` belongs to spec 002 (B-06).
- **FR-009**: Any application code touched (for example to expose pricing to tests) MUST use
  conservative C89 style (B-11); the test code is not bound by this.
- **FR-010**: All assumptions made MUST be recorded in `assumptions.md`.

### Key Entities

- **Characterization test**: a recorded input script plus the expected output, generated
  from the code at commit `a2ba708` before any feature work.
- **Golden output**: the committed text output for a scenario, normalised for ANSI codes,
  the version string and the run date of new sales.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: From a clean clone, build plus the full test suite completes with one
  documented command each, in under 2 minutes.
- **SC-002**: 100% of the pricing paths in `calc_quote` (R/T, clamp low/high, trade-in)
  are exercised by a test.
- **SC-003**: Every menu screen reachable from the main menu is exercised by at least
  one end-to-end scenario; each `BR-` item in the baseline maps to at least one test.
- **SC-004**: Deliberately breaking a pricing constant makes the suite fail locally.
- **SC-005**: Zero changes in the output of any scenario between the original code and
  the code after this feature.

## Assumptions

See `src/dealerdesk/assumptions.md`, section F (F-01 to F-04, F-07, F-10 to F-13) and sections B/C:

- Linux/macOS/WSL with gcc or clang is the supported dev platform; Windows/Borland
  release builds stay out of scope (F-02).
- Tests drive the real binary through stdin on a data copy, plus unit tests on `calc.c`
  (F-03); golden outputs come from the current code before any change (F-04).
- Which test framework to use is a planning decision, not made here; prefer no new
  runtime dependency (constitution VII).
- Version: current is 2.3; the fleet release is 2.4 (B-06). Application code uses
  conservative C89 style (B-11).
- Build and test entry point: a `Makefile` (`make`, `make test`); decided 2026-11-15.
