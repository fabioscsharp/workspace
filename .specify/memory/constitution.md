# DealerDesk Workspace Constitution

This workspace holds the spec-kit artifacts (specs, plans, tasks) for **DealerDesk**,
a menu-driven terminal application written in C for car dealerships. The code lives in
the `src/dealerdesk` git submodule (remote: `fabioscsharp/dealerdesk`). Specs and
tooling live here; application code changes are made inside the submodule.

Only principles that are already true of the codebase or already agreed with the
requester are listed. See `src/dealerdesk/BRIEF.md` for the source of the agreements.

## Core Principles

### I. Behaviour Preservation (NON-NEGOTIABLE)
Dealership staff rely on the app as it is. Apart from the requested feature, nobody
should notice a change. Output for existing flows (quotes, invoices, listings, menus)
MUST stay identical, "to the penny" for money. Any change that alters existing
behaviour MUST be called out in the spec and approved before implementation.

### II. The Code Is the Source of Truth
The documentation is known to be stale (see `src/dealerdesk/assumptions.md`). When
README, help text, comments and code disagree, **the running code and the data files
win**. Behaviour is pinned by tests before it is changed (characterization tests),
not inferred from prose.

### III. Backward-Compatible Data Files (NON-NEGOTIABLE)
`data/*.dat` are pipe-delimited flat files, written on every change and read by
production binaries. Existing files MUST load with no manual conversion. New fields
are appended at the end of a record, and a missing field MUST default to the
pre-change behaviour. Old binaries (`dealerdesk.old` rollback) SHOULD still be able
to read files written by the new binary without crashing. Sample data in `data/` is
only modified when the spec says so.

### IV. Documentation Is Append-Only (NON-NEGOTIABLE)
Existing documentation in `src/dealerdesk` (`README.md`, `BRIEF.md`, `RELEASE.txt`,
`TICKET-142.md`, and any other existing doc) MUST NOT be edited or rewritten. Only
new text may be appended, including revisions, which go in a clearly dated section at
the bottom of the file. New documentation goes in new files, for example
`assumptions.md`. Source-code comments are code, not documentation, but existing
comments are not rewritten without need.

### V. Every Assumption Is Written Down
Anything not explicit in the ticket, docs or code, and anything that raised a question,
MUST be recorded in `src/dealerdesk/assumptions.md` with an ID, the reasoning and its
status (open, confirmed, rejected). Specs reference assumption IDs rather than
silently deciding.

### VI. Tested, Reproducible Changes
Changes ship with automated tests that run from one documented command and in CI.
Pricing logic stays in `calc.c` and is unit-testable without the terminal. A change is
not done until the characterization tests pass unchanged for non-affected behaviour.

### VII. Simplicity and Minimal Diffs
Keep the existing structure (`main.c`, `calc.c`, `dbfile.c`, `screen.c`, `globals.h`),
plain C, no new runtime dependencies, and no refactors unrelated to the feature.
Fixed-size tables and flat files are accepted constraints, not defects to fix here.

## Technical Constraints

Current behaviour is specified in `specs/000-baseline/spec.md` and the structure in
`docs/architecture.md`; every feature spec is a delta against them.

- Language: C (gcc/clang). Build: `make` from the repository root once spec 001 adds the
  `Makefile`. It builds from the same sources and libraries as
  `gcc -o dealerdesk *.c -lm`, which stays valid.
  Historical Windows releases used the Borland compiler, so avoid compiler-specific
  or C99+-only constructs in application code unless the spec accepts the risk.
- Runtime: run from the repository root; data is read from and written to `./data`.
- Money is stored with two decimals (`%.2f`) and rounded with `rnd2()` (half up).
- `|` is the field separator; input sanitisation replaces `|` with `/`.
- Release is currently manual (`RELEASE.txt`): copy the binary to the file server,
  keep `dealerdesk.old` for rollback.

## Development Workflow

- Spec-kit flow per change: `/speckit-specify` → `/speckit-clarify` → `/speckit-plan`
  → `/speckit-tasks` → `/speckit-analyze` → `/speckit-implement` → `/speckit-converge`.
- Work in the submodule on a feature branch; commit as you go (history is reviewed).
  Commit the submodule first, then the workspace pointer bump.
- Never leave modified `data/*.dat` behind after manual testing
  (`git checkout -- data`).
- The deliverable includes the agent setup (`CLAUDE.md`, specs, skills) and a
  `NOTES.md` of at most two pages in the dealerdesk repo, as required by the brief.

## Governance

This constitution supersedes other practices in this workspace. Amendments require a
note in the commit message and a version bump. Specs and plans MUST be checked against
principles I, III and IV before implementation starts.

**Version**: 1.0.1 | **Ratified**: 2026-11-15 | **Last Amended**: 2026-11-16
