# Research: Reliable build and characterization tests (001)

Phase 0 of [plan.md](plan.md). Technical Context has no `NEEDS CLARIFICATION`. The three
clarified decisions (Makefile, date normalisation, byte-for-byte data) are in
[spec.md](spec.md#clarifications). Each decision below is recorded in
`src/dealerdesk/assumptions.md` as F-14 onwards when it is implemented.

## R-1 Test framework and harness language

- **Decision**: unit tests in plain C with a small local `CHECK` macro (no framework). The
  e2e runner is a POSIX `sh` script using `diff`, `cmp`, `sed`, `awk`, `mktemp`.
- **Rationale**: constitution VII forbids new dependencies, and the spec prefers none
  (Assumptions). These tools ship with every Linux, macOS and WSL install. CI
  (spec 003) needs nothing beyond `build-essential`.
- **Alternatives considered**: Unity/cmocka (a vendored or installed dependency for about
  30 assertions), Python + pytest (a runtime dependency, and slower startup), BATS (an
  extra install), `expect` (not needed because the app reads plain stdin).

## R-2 Exposing pricing to unit tests

- **Decision**: link `tests/unit/test_calc.c` with the unchanged `calc.c` through the
  existing `calc.h`. No application change.
- **Rationale**: `calc_quote` and `rnd2` are already public and `calc.c` has no I/O or
  global dependencies (it needs only `-lm`). The FR-002 allowance is not needed (F-13).
- **Alternatives considered**: `#include "../../calc.c"` (fragile), moving code (a
  refactor, which is forbidden).

## R-3 How unit expectations are captured "from the original code"

- **Decision**: the case table holds inputs plus expected outputs as `%.2f` strings for
  every `QUOTE` field. The expected strings are produced once by running the test binary
  in capture mode (`test_calc --print`) against `calc.c` at `a2ba708`, pasted into the
  table, and cross-checked against the three worked examples in the baseline (invoices
  50230 to 50232).
- **Rationale**: `%.2f` is exactly what the app shows and stores, which is what
  "to the penny" means. Comparing formatted strings also avoids false failures from
  unrounded `net` (C-01) while still catching any visible penny change.
- **Cases**: R and T. pct −5, 0, 5, 8, 10, 15, 15.01, 40. Trade-in 0 and >0. List prices
  that land on half-penny boundaries for `rnd2` (for example 0.125, 2.675, 1.005, 10.045).
  The three invoice examples. One very large and one zero list price.
- **Alternatives considered**: exact `double` equality (brittle across compilers, and
  shows nothing a user would see), recomputing expectations in the test (it would copy
  the bugs, and spec Edge Cases forbid it).

## R-4 Driving the binary and avoiding hangs

- **Decision**: each scenario runs `cd "$tmp" && "$BIN" < input > actual.raw`, with
  stdin always redirected from a file. Every input file ends by exiting the menus with
  `0`, so EOF is a safety net and is also tested on purpose (03-eof-exit).
- **Rationale**: `read_line` calls `exit(0)` on EOF, so a file-backed stdin cannot hang.
  No `timeout` (macOS lacks it by default) and no pty are needed.
- **Alternatives considered**: `expect` or pty (a dependency, and timing flakiness).

## R-5 Normalisation

- **Decision**: one `sed` pipeline (`tests/e2e/normalise.sh`) applied to both output and
  data, in this order:
  1. delete `ESC[2J ESC[H` sequences (F-07)
  2. `DEALERDESK v[0-9]+\.[0-9]+` → `DEALERDESK vX.Y`. The column width is preserved
     because v2.3 and v2.4 are the same length (FR-008)
  3. the run date → `<TODAY>`. The runner records `date +%Y-%m-%d` before and after the
     binary runs and replaces both values, which covers a run across midnight. Only those
     exact strings are replaced, so the 2021 sample dates are untouched (F-11)
- **Rationale**: this is the minimum needed for stable goldens. Everything else is
  compared literally.
- **Alternatives considered**: libfaketime, or an app date override (both rejected in
  clarify Q2).

## R-6 Data comparison and isolation

- **Decision**: each scenario gets `mktemp -d`. The fixture (default `data/`) is copied to
  `$tmp/data/`, the binary runs with `$tmp` as its working directory, and a `trap`
  removes the directory. After the run, every file in `$tmp/data/` is normalised and
  compared with `cmp` against `expected-data/`. If the scenario has no `expected-data/`,
  the files are compared against the original fixture (meaning "nothing changed"). Extra
  or missing files fail the scenario.
- **Rationale**: FR-005, FR-005a and C-13. The repo's `data/` is never opened for writing.
  `git status` is checked to be clean in the quickstart and by CI later.

## R-7 Golden generation and drift protection

- **Decision**: `make golden` (`run.sh --update`) writes `expected.out` and
  `expected-data/` from the current binary. It is used only to create new scenarios, and
  every regenerated golden is reviewed in `git diff` before committing. Before the first
  generation, the runner checks that the app sources match `a2ba708`
  (`git diff --quiet a2ba708 -- '*.c' '*.h' data`) and refuses otherwise unless
  `FORCE=1`.
- **Rationale**: F-04. Goldens must come from the original code, and later features (002)
  must not regenerate them silently. `make test` never writes goldens.

## R-8 Build flags and the mutation check

- **Decision**: `CC ?= cc`, `CFLAGS ?= -Wall`, and `LDLIBS = -lm`. Sources are listed
  explicitly (`main.c calc.c dbfile.c screen.c`). `-Wall` only adds warnings and does not
  change generated code, so the binary behaves the same as one built with
  `gcc -o dealerdesk *.c -lm`. `make test-mutation` copies the sources to a temp dir,
  uses `sed` to change the VAT constant `0.20` to `0.21` in `calc.c`, builds, runs
  `make test` against that binary, and passes only if the suite fails.
- **Rationale**: FR-001, SC-004, and "no `-Werror`" (spec Edge Cases). Mutating a copy
  never touches the working tree.
- **Alternatives considered**: a manual "edit and revert" instruction (not repeatable),
  `-Werror` (rejected by the spec until the legacy warnings are cleaned up).
