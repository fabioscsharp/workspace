# Implementation Plan: CI and repeatable release for DealerDesk

**Branch**: `003-ci-and-release` (work branch in `src/dealerdesk`) | **Date**: 2026-11-16 |
**Spec**: [spec.md](spec.md)

**Input**: Feature specification from `specs/003-ci-and-release/spec.md` (clarified
2026-11-16)

## Summary

Run the existing `make` targets automatically and turn the hand-made release into a
tag-driven (or manually started) GitHub Actions workflow, without changing any
application source file.

- **CI** (`.github/workflows/ci.yml`): on push to `master` and on every pull request,
  build and run `make test`, `make test-mutation`, `make test-compat` and the new release
  script tests, on Ubuntu with gcc and with clang. The workflow is a required check on
  `master` (a manual GitHub setting, documented).
- **Release** (`.github/workflows/release.yml`): starts from a plain version tag (`2.4`) or
  a manual run. It checks that the requested version equals the version in the on-screen
  header (`screen.c`), reruns the same checks, packages `dealerdesk-<ver>.zip` (binary plus
  sample data) with a SHA256 file, and publishes a GitHub Release whose notes repeat the
  backup step and rollback warning.
- **One definition of "the checks"**: `scripts/check.sh` runs the whole check list; CI and
  release both call it, so they cannot drift apart.
- **Scripts** (`scripts/version.sh`, `package.sh`, `release-notes.sh`) hold the logic, so
  everything but the GitHub parts can be run and tested locally.
- **Docs**: a new `RELEASING.md` holds the manual steps (file server copy, showroom
  e-mail, rollback, the required-check setting); `RELEASE.txt` only gets an appended,
  dated pointer.

## Technical Context

**Language/Version**: POSIX `sh` for scripts; GitHub Actions workflow YAML. No change to
the C application (conservative C89, B-11).

**Primary Dependencies**: none new at runtime. CI uses GitHub-hosted `ubuntu-latest` with
the preinstalled gcc, clang, make, zip, `sha256sum` and `gh`. Two actions:
`actions/checkout` and `actions/upload-artifact` (pinned to a major version; see R-5).

**Storage**: none. Release input is the committed `data/` sample set, copied unchanged.

**Testing**: the 001 suite (unit, e2e, mutation, compat) plus a small shell test for the
new release scripts (`tests/release/test_release.sh`, target `make test-release`).

**Target Platform**: CI on Ubuntu; the produced binary is Linux x86-64 (F-02). Showroom
platform is not confirmed (see R-9 and open assumption F-24).

**Project Type**: interactive terminal application plus delivery tooling.

**Performance Goals**: CI result in under 5 minutes per run (the 001 suite takes a few
seconds; the rest is runner start and checkout). The 10 and 15 minute job timeouts are
ceilings, not goals.

**Constraints**: no `-Werror`; full git history in the checkout (`test-compat`); tests use
temporary data copies and leave the tree clean; `RELEASE.txt` byte-identical apart from an
appended section (SC-004); no application source change (constitution I, VII).

**Scale/Scope**: 2 workflows, 4 scripts, 1 new doc, 1 appended section, 1 Makefile
target.

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Principle | Check | Result |
|-----------|-------|--------|
| I. Behaviour preservation | No application source change. The version stays in `screen.c` and is only read. | PASS |
| II. Code is the source of truth | The release version is read from the running code's header, not from docs or the tag. | PASS |
| III. Backward-compatible data | `make test-compat` is a required CI check, so the rollback promise is enforced. The package carries `data/` unchanged. | PASS |
| IV. Docs append-only | `RELEASE.txt` gets an appended dated section. New text goes in `RELEASING.md`. README untouched. | PASS |
| V. Assumptions written down | Decisions R-1 to R-10 in [research.md](research.md) become F-21 onwards in `assumptions.md` during implementation. | PASS |
| VI. Tested, reproducible | One command (`scripts/check.sh`) is the full check list, local and in CI. The release scripts have their own test. | PASS |
| VII. Simplicity, minimal diffs | No new runtime dependency, no app refactor. One extra Makefile target and shell scripts. | PASS |

**Knowledge-file duties (CLAUDE.md rule 4)**: in the same change, update
`docs/architecture.md` "Build and run" and "Release" (CI, release workflow, scripts) and
`specs/000-baseline/spec.md` BR-082 (CI now exists). No pricing or data format behaviour
changes.

## Project Structure

### Documentation (this feature)

```text
specs/003-ci-and-release/
├── spec.md
├── plan.md              # this file
├── research.md          # Phase 0: decisions R-1..R-10
├── data-model.md        # Phase 1: version, workflows, release package, notes, gate
├── quickstart.md        # Phase 1: how to validate the feature end to end
├── contracts/
│   ├── workflows.md     # triggers, inputs, jobs, permissions, failure rules
│   └── scripts.md       # command-line contract of scripts/*.sh and make test-release
└── tasks.md             # created by /speckit-tasks
```

### Source code (submodule `src/dealerdesk`)

```text
.github/workflows/
├── ci.yml                    # new
└── release.yml               # new
scripts/
├── check.sh                  # new: the full check list (CI and release call this)
├── version.sh                # new: print or compare the header version
├── package.sh                # new: build dealerdesk-<ver>.zip + .sha256
└── release-notes.sh          # new: print the release notes
tests/release/
└── test_release.sh           # new: tests version.sh, package.sh, release-notes.sh
RELEASING.md                  # new: manual steps, required-check setting, notes source
Makefile                      # appended: test-release target (no change to existing targets)
RELEASE.txt                   # appended: dated pointer to RELEASING.md and CI
assumptions.md                # appended: F-21 onwards
```

**Structure Decision**: workflow YAML stays thin (checkout, call a script, upload/publish).
All logic lives in `scripts/` so it can be run locally and covered by `test_release.sh`.
Workspace side: `docs/architecture.md`, `specs/000-baseline/spec.md`, then the pointer
bump, committed after the submodule.

## Complexity Tracking

No constitution violations.
