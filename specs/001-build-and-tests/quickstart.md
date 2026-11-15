# Quickstart: validate 001 (build and characterization tests)

**Prerequisites**: Linux, macOS or WSL with `make` and gcc or clang. On Windows, run
inside WSL (Ubuntu here: gcc 15.2, GNU make 4.4).

All commands run from the submodule root, `src/dealerdesk`.

## 1. Build (US1, FR-001)

```bash
make clean && make
```

Expected: `./dealerdesk` exists. There may be warnings, but no errors.

```bash
printf '0\n' | ./dealerdesk; echo "exit=$?"
```

Expected: the main menu, then `Goodbye.`, then `exit=0` (US1 #2).

## 2. Full suite (US2, FR-007, SC-001)

```bash
time make test
```

Expected: unit and e2e summaries with every check PASS, exit status 0, well under 2
minutes.

## 3. Working tree stays clean (FR-005, US2 #4)

```bash
git status --porcelain
```

Expected: only build outputs, and those are ignored. `data/` is unchanged.

## 4. A wrong pricing constant is caught (SC-004, US2 #5)

```bash
make test-mutation
```

Expected: the target reports that the mutated suite failed, and exits 0.

## 5. Goldens come from the original code (F-04, SC-005)

```bash
git diff --quiet a2ba708 -- '*.c' '*.h' data && echo "sources = a2ba708"
```

Expected: `sources = a2ba708` (before any feature work). After later features, `make test`
must still pass, apart from scenarios a spec explicitly re-baselines.

## 6. Spot checks

- Invoices 50230, 50231 and 50232 view as their totals 9331.30, 14809.60 and 11821.48
  (`tests/e2e/scenarios/20-invoice-view/expected.out`; baseline worked examples).
- The sell scenarios' `expected-data/invoices.dat` has a new line with `<TODAY>` as the
  date, and `next_ids.dat` shows `invoice=50234` (FR-005a).
- `tests/README.md` coverage table: every `BR-` item from `specs/000-baseline/spec.md` is
  listed (SC-003).
