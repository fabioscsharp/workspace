# DealerDesk Architecture (current)

**Updated**: 2026-11-15. Describes `src/dealerdesk` at commit `a2ba708`. Behaviour and
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
| `main.c` | `main()`, all menus and screens, quote/invoice printing, sale flow, lookups (`find_veh`, `find_cust`). Defines the global tables and ID counters. |
| `calc.c` / `calc.h` | Pricing. `calc_quote(QUOTE*, list, cust_type, disc_pct, tradein)` fills the `QUOTE` struct; `rnd2()` rounds to pennies. Pure functions, no I/O: the only unit-testable seam today. |
| `dbfile.c` / `dbfile.h` | `load_all()` and `save_stock/customers/invoices/ids()`. Parses with `strtok` on `|`, writes with `fprintf`. `DATA_DIR` is `"data/"`. |
| `screen.c` / `screen.h` | Terminal helpers: `cls` (ANSI `\033[2J\033[H`), `header` (prints version), `read_line` (fgets, strips newline, replaces `|` with `/`, exits on EOF), `read_int`, `read_money`, `confirm`, `pause_enter`. |
| `globals.h` | `VEHICLE`, `CUSTOMER`, `INVOICE` structs, table sizes, status constants, `extern` declarations. |

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
| `customers.dat` | `id\|name\|phone\|email\|type` | type `R` retail, `T` trade; missing type loads as `R` |
| `invoices.dat` | `no\|date\|veh_id\|cust_id\|list\|disc\|net\|vat\|fees\|tradein\|total` | `disc` = all discounts combined; `fees` = delivery + registration; date `YYYY-MM-DD` |
| `next_ids.dat` | `vehicle=N`, `customer=N`, `invoice=N` | read with `sscanf`; defaults 1000, 2000, 50000 |

Table limits: 500 vehicles, 500 customers, 2000 invoices. Struct field sizes: plate 12,
make 24, model 40, name 48, phone 20, email 64, date 11.

## Control flow

1. `main()` calls `load_all()` then loops over the main menu.
2. Submenus (`stock_menu`, `customer_menu`, `invoice_menu`) loop until option 0.
3. A sale builds a quote with `calc_quote`, prints it, asks for confirmation, copies the
   `QUOTE` into an `INVOICE`, marks the vehicle SOLD, then saves invoices, stock and ids.

## Build and run

- Build (Linux/macOS/WSL): `gcc -o dealerdesk *.c -lm` from the repo root. The README's
  second build command (with `report.c`) is wrong (B-10).
- Run from the repo root so `./data` resolves. Restore sample data with
  `git checkout -- data` (inside the submodule).
- Historical Windows releases used the Borland compiler (not available here).

## Release (today)

Manual: build, smoke test with sample data, rename the old binary on
`\\fileserver01\apps\dealerdesk\` to `dealerdesk.old`, copy the new one, e-mail the
showrooms, copy `.old` back to roll back (`RELEASE.txt`). No versioning source of truth:
the version string is hard-coded in `screen.c`.

## Constraints and design decisions in force

- Pricing lives in `calc.c` (moved out of `main.c` in 2015) and must stay free of I/O.
- File-format compatibility is a hard requirement: append new trailing fields only.
- Plain C, fixed-size tables, no dynamic allocation; keep the footprint.
- The app trusts its input: little validation (see assumptions C-12).
- Decisions are reconstructed from git history; there are no ADRs.

## Planned structural changes (see specs)

- `001-delivery-pipeline`: build entry point, test harness, CI, release script.
- `002-fleet-discount`: a fleet flag on `CUSTOMER`, a fleet term in `calc_quote`, a new
  trailing field in `customers.dat` and `invoices.dat` (always written), loaders that keep
  empty fields (split on every `|`), a menu action to set fleet status, and the header
  version change to v2.4. Decisions: `src/dealerdesk/assumptions.md` sections B, D, E.
