# Quickstart: validate 002 (fleet discount)

**Prerequisites**: spec 001 is in place (`make test` green on `fcff60d`). Use WSL, Linux or
macOS with `make` and gcc or clang. Run everything from `src/dealerdesk`, and run
`git checkout -- data` after any manual run.

## 1. The whole suite

```bash
make clean && make test
```

Expected: every unit case and every scenario passes, including the new 23 to 29.

## 2. Only the approved goldens changed (FR-007, FR-009, E-07)

```bash
git diff --stat fcff60d -- tests/e2e/scenarios
sh tests/e2e/check-rebaseline.sh
```

Expected:
- **Changed** existing scenarios:
  - 02, 04, 10, 11, 12, 21 and 22 (`expected.out`);
  - 11's `input`;
  - 11, 16 and 17 (`expected-data/`).
- **Added** scenarios: 23 to 29.
- `check-rebaseline.sh` reports only approved lines: the menu line, the add prompt, the
  help line, and the trailing `|0` / `|0.00`.

## 3. Ticket acceptance by hand (optional)

```bash
make && ./dealerdesk
```

1. Customers → 4 → `2008` → `Y`. You see "Customer 2008 is now a fleet customer."
2. Stock → 2. Note an in-stock vehicle and its price (1001 is SOLD in the sample).
3. Quote with 2008, 0%, no trade-in: a `Fleet discount` line appears after the list price.
   Fees are unchanged.
4. Quote with a non-fleet customer: no fleet line, and the figures are as before.
5. Invoices → 2 → `50232`: identical to v2.3 (Greenway's old manual 8% is not recalculated).

Then run `git checkout -- data`.

## 4. Worked examples (SC-002)

The four rows in [research.md](research.md) R-1 appear in `tests/unit/test_calc.c` and on
screen in `25-fleet-quote/expected.out`.

## 5. Old binary still reads new files (US3 #4)

```bash
make test-compat
```

Expected: the `a2ba708` binary lists and views v2.4 invoices with the same totals, and
exits 0.

## 6. Still guarded

```bash
make test-mutation
```

Expected: the mutant is killed.
