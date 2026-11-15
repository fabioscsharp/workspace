# Contract: e2e scenario format (`src/dealerdesk/tests/e2e/scenarios/`)

```text
tests/e2e/scenarios/NN-<name>/
├── input            # required: stdin for the binary
├── expected.out     # required: normalised golden stdout
├── fixture          # optional: one word, sample | empty | full (default sample)
└── expected-data/   # optional: normalised golden data/*.dat after the run
```

## Runner behaviour (`tests/e2e/run.sh`)

For each scenario directory, in name order:

1. `tmp=$(mktemp -d)`; set a `trap` to remove it; build the fixture into `$tmp/data/`.
2. `d1=$(date +%Y-%m-%d)`; `(cd "$tmp" && "$BIN" < input > actual.raw)`;
   `d2=$(date +%Y-%m-%d)`.
3. Normalise `actual.raw` and every file in `$tmp/data/` (rules in
   [data-model.md](../data-model.md#normalisation-rule)), replacing `d1` and `d2`.
4. Output: `diff -u expected.out actual.out`.
5. Data: the set of file names must match exactly. Each file is checked with
   `cmp expected-data/<f> <normalised f>`, or against the normalised fixture when
   `expected-data/` is absent.
6. Record PASS/FAIL. Continue with the next scenario. Exit non-zero if any failed.

`--update` (used by `make golden`) runs steps 1 to 3 and writes the results as the new
goldens instead of comparing them.

## Rules for authors

- One concern per scenario. Name it after the screen or path (for example
  `08-stock-change-price`).
- `input` exits cleanly (`0` back to the main menu, then `0`), except in `03-eof-exit`.
- Never hand-edit `expected.out`. Regenerate it with `make golden SCENARIO=NN-*` and
  review the diff.
- Add the scenario to the coverage table in `tests/README.md` with its `BR-` items.
