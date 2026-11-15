# Contract: Makefile targets (`src/dealerdesk/Makefile`)

This is the developer interface for building and testing. Spec 003 (CI) calls only
`make` and `make test`. Every target is run from the submodule root.

| Target | Does | Exit status | Writes |
|--------|------|-------------|--------|
| `make` / `make all` | builds `./dealerdesk` from `main.c calc.c dbfile.c screen.c` | 0 on success | `dealerdesk`, `*.o` |
| `make test` | `all`, then `unit`, then `e2e`; prints a pass/fail summary | 0 only if every check passes | build outputs, temp dirs (removed) |
| `make unit` | builds and runs `tests/unit/test_calc` | 0 if all cases pass | `tests/unit/test_calc` |
| `make e2e` | runs `tests/e2e/run.sh` against `./dealerdesk` | 0 if all scenarios pass | temp dirs only |
| `make test-mutation` | builds a copy with VAT `0.20` changed to `0.21` and runs the suite on it | 0 only if that suite **fails** | temp dir only |
| `make golden` | regenerates `expected.out` / `expected-data/` (refuses unless sources equal `a2ba708` or `FORCE=1`) | 0 on success | `tests/e2e/scenarios/*` |
| `make clean` | removes build and test binaries | 0 | — |

Variables: `CC` (default `cc`), `CFLAGS` (default `-Wall`), and `SCENARIO=<glob>` to
limit `e2e` or `golden` to matching scenarios.

Guarantees:

- No target modifies `data/` or any tracked file, except `make golden` (which only
  touches scenario goldens).
- `make test` never regenerates goldens.
- On failure, `e2e` prints the scenario name and a unified `diff` of the normalised
  expected and actual output, or a `cmp` report for data files.
- The output of `make` (the `dealerdesk` binary) behaves the same as one built with
  `gcc -o dealerdesk *.c -lm` (FR-001, FR-002).
