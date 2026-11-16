# DealerDesk Architecture (current)

**Updated**: 2026-11-16 (spec 001 build and tests; spec 002 fleet discount, v2.4; spec 003
CI and release). The
application code is `a2ba708` plus the spec 002 changes. Behaviour and
pricing rules are in `specs/000-baseline/spec.md`; assumptions and doc-versus-code
conflicts are in `src/dealerdesk/assumptions.md`. This file is updated when the structure
changes.

## Overview

A single-process, single-user, menu-driven terminal application in C. State is held in
fixed-size global arrays loaded from flat files at start-up and written back to disk on
every change. There is no network, no database and no external dependency beyond the C
standard library and `libm`.

## Modules

| File | Responsibility |
|------|----------------|
| `main.c` | `main()`, all menus and screens (incl. `set_fleet_status`, Customers option 4), quote/invoice printing, `type_txt()` (the `, Fleet` marker), sale flow, lookups (`find_veh`, `find_cust`). Defines the global tables and ID counters. |
| `calc.c` / `calc.h` | Pricing. `calc_quote(QUOTE*, list, cust_type, fleet, disc_pct, tradein)` fills the `QUOTE` struct (incl. `fleet_disc`, spec 002); `rnd2()` rounds to pennies. Pure functions, no I/O: unit-tested directly. |
| `dbfile.c` / `dbfile.h` | `load_all()` and `save_stock/customers/invoices/ids()`. `load_stock` parses with `strtok` on `|`. `load_customers` and `load_invoices` use `next_field()`, which keeps empty fields (spec 002, E-04). Writes with `fprintf`. `DATA_DIR` is `"data/"`. |
| `screen.c` / `screen.h` | Terminal helpers: `cls` (ANSI `\033[2J\033[H`), `header` (prints version), `read_line` (fgets, strips newline, replaces `|` with `/`, exits on EOF), `read_int`, `read_money`, `confirm`, `pause_enter`. |
| `globals.h` | `VEHICLE`, `CUSTOMER` (incl. `fleet`), `INVOICE` (incl. `fleet_disc`) structs, table sizes, status constants, `extern` declarations. |

Dependencies: `main.c` uses all modules; `dbfile.c` uses `globals.h`; `calc.c` is standalone
(needs `-lm` for `floor`); `screen.c` is standalone.

## Data model and files (`./data`)

Plain text, one record per line, fields separated by `|`. Lines that are empty or start
with `#` are skipped. A missing file means an empty table. A line with fewer fields than
expected loads with defaults (0 or empty); extra trailing fields are ignored. Every save
rewrites the whole file.

| File | Record | Notes |
|------|--------|-------|
| `stock.dat` | `id\|plate\|make\|model\|year\|mileage\|cost\|price\|status` | status 0 IN STOCK, 1 RESERVED, 2 SOLD; money `%.2f` |
| `customers.dat` | `id\|name\|phone\|email\|type\|fleet` | type `R` retail, `T` trade, missing or empty loads as `R`. `fleet` (v2.4, always written) is fleet only if exactly `1`; 5-field rows load as non-fleet |
| `invoices.dat` | `no\|date\|veh_id\|cust_id\|list\|disc\|net\|vat\|fees\|tradein\|total\|fleet_disc` | `disc` = all discounts combined (incl. fleet); `fees` = delivery + registration; date `YYYY-MM-DD`; `fleet_disc` (v2.4, always written), missing = 0 |
| `next_ids.dat` | `vehicle=N`, `customer=N`, `invoice=N` | read with `sscanf`; defaults 1000, 2000, 50000 |

Table limits: 500 vehicles, 500 customers, 2000 invoices. Struct field sizes: plate 12,
make 24, model 40, name 48, phone 20, email 64, date 11.

## Control flow

1. `main()` calls `load_all()` then loops over the main menu.
2. Submenus (`stock_menu`, `customer_menu`, `invoice_menu`) loop until option 0.
3. A sale builds a quote with `calc_quote`, prints it, asks for confirmation, copies the
   `QUOTE` into an `INVOICE` (`disc` = trade + fleet + salesman, `fleet_disc` separately),
   marks the vehicle SOLD, then saves invoices, stock and ids.

## Build and run

- Build (Linux/macOS/WSL): `make` from the repo root (`Makefile`, spec 001). It compiles
  the explicit list `main.c calc.c dbfile.c screen.c` with `-lm`, so it is equivalent to
  `gcc -o dealerdesk *.c -lm`, which still works. `CC` defaults to `cc` and `CFLAGS` to
  `-Wall` (no `-Werror`). The README's second build command (with `report.c`) is wrong (B-10).
- Tests: `make test` (= `make unit` + `make e2e`). `make test-mutation` checks that a
  wrong VAT constant fails the suite, and `make golden` regenerates goldens (guarded to
  `a2ba708`). Layout:
  - `tests/unit/test_calc.c` links `calc.c` unchanged (`%.2f` case table).
  - `tests/e2e/run.sh` feeds `tests/e2e/scenarios/NN-name/input` to the binary on a temp
    copy of a fixture (`sample` = `data/`, `empty`, `full`, or a committed directory
    under `tests/e2e/fixtures/`, such as `fleet` and `legacy-gaps`), normalises the output and
    data (ANSI clear-screen, version, run date), then compares the output with
    `expected.out` and the data byte for byte with `expected-data/`.

  - `make test-compat` builds the `a2ba708` binary from git and checks that it still reads
    files written by v2.4 (spec 002).
  - `make test-release` tests the release scripts (spec 003); its zip case needs `zip`
    and is skipped locally without it, but never in CI.

  Details: `tests/README.md`.
- Checks and CI (spec 003): `sh scripts/check.sh` is the one list of checks: `make test`,
  `make test-mutation`, `make test-compat`, `make test-release`, then it verifies that
  `git status --porcelain` is the same after as before. `.github/workflows/ci.yml` runs it on
  every pull request and every push to `master`, once with gcc and once with clang, on
  Ubuntu with the full git history. The aggregate job `build-and-test` is the required
  check on `master` (a GitHub setting, see `RELEASING.md` and F-25, F-36).
- `.gitattributes` forces LF line endings (`* text=auto eol=lf`), so Windows checkouts
  keep the shell scripts runnable and the data byte-identical to Linux.
- Run from the repo root so `./data` resolves. Restore sample data with
  `git checkout -- data` (inside the submodule).
- Historical Windows releases used the Borland compiler (not available here).

## Release (today)

Automated (spec 003): `.github/workflows/release.yml` starts from a new plain version tag
(for example `2.5`) or by hand with a version, and checks it against the version in the
`screen.c` header (`scripts/version.sh`, the single source of the version). It then runs
`scripts/check.sh`, builds `dealerdesk-<version>.zip` (the Linux binary plus the committed
sample `data/`) and a SHA256 file with `scripts/package.sh`, takes the notes from the marked
block of `RELEASING.md` (`scripts/release-notes.sh`), and publishes a GitHub Release. It
refuses to touch an existing Release, and an existing tag is used as it is. The binary is
built for Linux only (F-24); a Windows build is deferred (F-31).

Still manual (`RELEASE.txt`, `RELEASING.md`): back up `data/`, rename the old binary on
`\\fileserver01\apps\dealerdesk\` to `dealerdesk.old`, copy the new one, e-mail the
showrooms, copy `.old` back to roll back. Rolling back loses fleet data on the old
binary's first save (E-05).

## Constraints and design decisions in force

- Pricing lives in `calc.c` (moved out of `main.c` in 2015) and must stay free of I/O.
- File-format compatibility is a hard requirement: append new trailing fields only.
- Plain C, fixed-size tables, no dynamic allocation; keep the footprint.
- The app trusts its input: little validation (see assumptions C-12).
- Decisions are reconstructed from git history; there are no ADRs.

## Planned structural changes (see specs)

- `001-build-and-tests`: **done 2026-11-16**. `Makefile` and `tests/`, see "Build and run".
- `003-ci-and-release`: **done 2026-11-16** (merged to `master` of the dealerdesk repo;
  release 2.4 is published, F-41). `.github/workflows/ci.yml` and `release.yml`,
  `scripts/`, `tests/release/`, `RELEASING.md`; see "Build and run" and "Release".
- `002-fleet-discount`: **done 2026-11-16** (v2.4). The work:
  - a fleet flag on `CUSTOMER`, and a fleet term in `calc_quote`;
  - a new trailing field in `customers.dat` and `invoices.dat`, always written;
  - loaders that keep empty fields;
  - the menu action `4. Set fleet status`, and the header version v2.4.

  Decisions are in `src/dealerdesk/assumptions.md` sections B, D, E (D-18 to D-27, E-07).
