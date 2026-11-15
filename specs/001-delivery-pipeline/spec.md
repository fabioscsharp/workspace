# Feature Specification: Reliable build, tests, CI and release for DealerDesk

**Feature Branch**: `001-delivery-pipeline` (in `src/dealerdesk`)

**Created**: 2026-11-15

**Status**: Draft; the version question (B-06) and compiler style (B-11) are decided
2026-11-15. One clarification remains (Makefile) before `/speckit-plan`.

**Baseline**: characterizes `specs/000-baseline/spec.md` (every `BR-` item gets a test);
changes no `BR-` behaviour. Structure: `docs/architecture.md`.

**Input**: BRIEF.md task 1: "Improve how changes get delivered on this codebase. You decide
what matters most and how far to take it." Today there are no tests, no CI, no reliable
build, and releases are done by hand (`RELEASE.txt`).

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

---

### User Story 3 - CI runs build and tests on every change (Priority: P2)

Every push and pull request builds the app and runs the tests on a clean machine (GitHub
Actions), with the result visible on the PR.

**Why this priority**: the team does not trust changes; automatic evidence replaces
nervous manual testing.

**Independent Test**: push a branch with a deliberately wrong pricing constant and see CI
fail; revert and see it pass.

**Acceptance Scenarios**:

1. **Given** a pull request, **When** CI runs, **Then** it builds with warnings enabled
   and runs all tests, and fails if either step fails.
2. **Given** a green run on the default branch, **Then** the built binary is available as
   a CI artifact.

---

### User Story 4 - Repeatable release steps (Priority: P3)

The manual release checklist is turned into a script plus a CI-built artifact, so the
steps that can be automated are, and the manual ones (copy to the file server, e-mail
showrooms, rollback copy) are written down in a new file. The existing `RELEASE.txt` is
only appended to, never rewritten.

**Why this priority**: valuable but the team cannot be reached from here (no file server
access).

**Independent Test**: running the packaging step on a clean checkout yields a versioned
archive containing the binary and the sample data, with the version taken from one place.

**Acceptance Scenarios**:

1. **Given** a tagged commit, **When** the release workflow runs, **Then** it uploads a
   versioned binary artifact.
2. **Given** `RELEASE.txt`, **When** this feature is done, **Then** its original content
   is intact and any change is an appended, dated section at the bottom.

---

### Edge Cases

- Tests run from a directory other than the repo root (the app reads `./data`): the test
  harness always sets the working directory.
- ANSI clear-screen codes in the output (`\033[2J\033[H`) must not break comparisons.
- Input ends unexpectedly (EOF): the app exits with status 0 (`read_line`), tests must
  not hang.
- Floating-point rounding at half-cent boundaries (`rnd2`): expected values come from
  the original implementation, not recomputation.
- No `-Werror` on the legacy code: warnings are reported, not fatal, until cleaned up.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: The repo MUST provide one documented command that builds `dealerdesk` on
  Linux/macOS/WSL with gcc or clang (see assumption F-01, B-10).
- **FR-002**: The build MUST NOT change application behaviour or require source changes
  to existing application files beyond what is needed to expose `calc_quote` to tests.
- **FR-003**: There MUST be unit tests for `calc_quote` and `rnd2` covering customer
  types R and T, discount clamping, trade-in, and rounding, with expected values captured
  from the unmodified code (F-04).
- **FR-004**: There MUST be end-to-end tests driving the binary via stdin that cover:
  stock list, add vehicle, find by plate, change price, reserve/release, customers list/
  add/find, quote, sell, invoice list and view, help, invalid option, EOF.
- **FR-005**: Tests MUST run against a temporary copy of the data and leave the working
  tree clean (C-13).
- **FR-006**: There MUST be a golden-file check that sample data loads and prints
  unchanged, proving "existing data files load without manual conversion".
- **FR-007**: A CI workflow MUST build and run all tests on push and pull request and
  fail on any error.
- **FR-008**: A single place MUST define the version string used by the release step. This
  feature keeps the on-screen header at `v2.3` (it changes no behaviour); the header becomes
  `v2.4` only in the fleet release (spec 002, B-06). Tests normalise the version string so
  the bump does not break them.
- **FR-009**: The release procedure MUST be scripted/automated where possible; manual
  steps are documented in a new file and `RELEASE.txt` is append-only. The checklist
  includes backing up `data/` before deploying (E-05; the step itself is appended to
  `RELEASE.txt` by spec 002).
- **FR-011**: Any application code touched (for example to expose pricing to tests) MUST
  use conservative C89 style (B-11); the test code is not bound by this.
- **FR-010**: All assumptions made MUST be recorded in `assumptions.md`.

### Key Entities

- **Characterization test**: a recorded input script plus the expected output, generated
  from the code at commit `a2ba708` before any feature work.
- **Golden output**: the committed text output for a scenario, normalised for ANSI codes.
- **Release artifact**: the built `dealerdesk` binary plus sample data, versioned.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: From a clean clone, build plus the full test suite completes with one
  documented command each, in under 2 minutes.
- **SC-002**: 100% of the pricing paths in `calc_quote` (R/T, clamp low/high, trade-in)
  are exercised by a test.
- **SC-003**: Every menu screen reachable from the main menu is exercised by at least
  one end-to-end scenario.
- **SC-004**: Deliberately breaking a pricing constant makes the suite fail locally and
  in CI.
- **SC-005**: Zero changes in the output of any scenario between the original code and
  the code after this feature.

## Assumptions

See `src/dealerdesk/assumptions.md`, section F and sections B/C for evidence:

- Linux/macOS/WSL with gcc or clang is the supported dev/CI platform; Windows/Borland
  release builds stay out of scope (F-02).
- Tests drive the real binary through stdin on a data copy, plus unit tests on `calc.c`
  (F-03).
- GitHub Actions is the CI provider (F-05); copying to the file server and e-mailing
  showrooms remain manual (F-06).
- Which test framework to use is a planning decision, not made here; prefer no new
  runtime dependency (constitution VII).
- Version: current is 2.3; the fleet release is 2.4 (B-06). Application code uses
  conservative C89 style (B-11).
- [NEEDS CLARIFICATION: Is a `Makefile` acceptable as the single build entry point, or is
  another build tool preferred?]
