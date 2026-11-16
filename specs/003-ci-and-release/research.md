# Research: CI and repeatable release (003)

Phase 0 of [plan.md](plan.md). Technical Context has no `NEEDS CLARIFICATION`. The five
clarified decisions (trigger, tag format, CI checks, release output, required check) are in
[spec.md](spec.md#clarifications). Each decision below is recorded in
`src/dealerdesk/assumptions.md` as F-21 onwards when it is implemented.

## R-1 Single source of the version

- **Decision**: the version string in the header (`screen.c`, `DEALERDESK v2.4`) is the
  single place. `scripts/version.sh` reads it with `sed`; nothing else stores a version.
  The tag or typed version is only compared with it.
- **Rationale**: FR-002 asks for one place; moving it to `version.h` needs an application
  change, which constitution I and VII avoid. The e2e runner already normalises
  `DEALERDESK vN.N`, so bumping the header never breaks the goldens.
- **Alternatives considered**: `version.h` (app change), a `VERSION` file (a second place
  that can disagree with the header), taking the version from the tag (the binary would
  then say something different).

## R-2 One list of checks for CI and release

- **Decision**: `scripts/check.sh` runs, in order, `make clean`, `make test`,
  `make test-mutation`, `make test-compat`, `make test-release`, then fails if
  `git status --porcelain` differs from its output before the first step (so it also works
  in a development tree with untracked files). Both workflows call it.
- **Rationale**: FR-001 and FR-005 require the same tests before packaging; one script
  makes drift impossible and lets a developer run the CI checks locally. The clean-tree
  check proves tests leave the repository untouched (FR-005 of spec 001).
- **Alternatives considered**: listing the commands in each workflow (can drift), a
  reusable `workflow_call` (more YAML for the same result).

## R-3 Triggers

- **Decision**: CI: `push` to `master` and `pull_request` (so a PR branch runs once, not
  twice). Release: `push` of tags matching `[0-9]*.[0-9]*` and `workflow_dispatch` with a
  required `version` input.
- **Rationale**: clarified answers 1 and 2. The existing tag `2.4` has no "v". A manual run
  uses the branch the person selects and creates the tag itself (see R-6).
- **Alternatives considered**: CI on every push to every branch (duplicate runs on PRs),
  tag pattern `v*` (does not match the existing convention).

## R-4 Compiler matrix and history

- **Decision**: CI runs the checks twice, with `CC=gcc` and `CC=clang`. Checkout uses
  `fetch-depth: 0`. The release uses gcc only.
- **Rationale**: plan 001 left the clang build to CI (not installed on the dev machine).
  `Makefile` has `CC ?= cc`, so an environment `CC` is honoured, and `compat.sh` uses
  `${CC:-cc}`. `test-compat` builds `a2ba708` from history, so a shallow clone fails it.
- **Alternatives considered**: gcc only (hides clang breakage), `macos-latest` (macOS is
  supported in the README, but out of the clarified scope; can be added as one matrix
  line later).

## R-5 Actions and permissions

- **Decision**: only `actions/checkout@v4` and `actions/upload-artifact@v4`. CI has
  `permissions: contents: read`; release has `contents: write`, used only by `gh release`.
  A CI concurrency group cancels superseded runs of the same PR.
- **Rationale**: least privilege and few third-party actions. The GitHub CLI `gh` is
  preinstalled on runners, so no release action is needed.
- **Alternatives considered**: `softprops/action-gh-release` (an extra third-party
  dependency for one command), pinning actions to commit SHAs (stronger, but heavier to
  maintain; noted as a possible later step in RELEASING.md).

## R-6 Manual run and tag creation

- **Decision**: a manual run first checks that the `version` input equals the header
  version, then runs the checks, packages, and has `gh release create <version> --target
  <sha>` create the tag with the Release. A tag run uses the pushed tag. If a Release for
  that version already exists, the run fails (no overwrite). A tag that already exists is
  used as is and never moved. The existing tag `2.4` (lightweight, on `49fb770`, which has
  no workflow files, and already on the remote) cannot fire a push event, so it is released
  by a manual run; before that the owner proves `git diff 2.4 HEAD -- '*.c' '*.h' data` is
  empty, so the build and the tagged commit have the same application. The version value is
  passed to the steps as an environment variable, never written into a `run:` script.
- **Rationale**: spec scenario US2 1a. Failing before packaging on a mismatch is the edge
  case "tag differs from header version".
- **Alternatives considered**: manual run only builds an artifact and no Release (would
  not meet "both paths produce the same result"), force-overwriting a Release (loses the
  history of what staff downloaded).

## R-7 Package layout and checksum

- **Decision**: `dealerdesk-<ver>.zip` containing one top folder `dealerdesk-<ver>/` with
  the `dealerdesk` binary and the committed `data/` sample files, plus
  `dealerdesk-<ver>.zip.sha256` (`sha256sum` format). Built with `zip -X` from a fixed file
  order and the committed data, so the contents are predictable.
- **Rationale**: FR-005 and clarified answer 4. The app must run from a folder that
  contains `./data` (README), so shipping them together matches how it is used.
- **Alternatives considered**: `tar.gz` (no extra tool, but less familiar for showroom
  staff), binary only (staff would lack the sample data and the run-from-root layout).

## R-8 Release notes source

- **Decision**: the notes are generated by `scripts/release-notes.sh` from the version and
  from the block in `RELEASING.md` between `<!-- notes:begin -->` and `<!-- notes:end -->`,
  which holds the "back up `data/`" step and the rollback warning (E-05).
- **Rationale**: FR-004 asks the checklist and the notes to stay consistent; one source
  block avoids a copy. `RELEASE.txt` keeps its own wording (append-only), and its appended
  section points to `RELEASING.md`.
- **Alternatives considered**: copying the text into the workflow (can drift), extracting
  from `RELEASE.txt` (fragile, and that file is read-only for us).

## R-9 Platform of the released binary

- **Decision**: the Release holds a Linux x86-64 binary built by gcc on Ubuntu. Notes and
  `RELEASING.md` say so. A Windows build stays out of scope (F-02, B-11).
- **Rationale**: nothing we can reach builds Windows or Borland binaries, and the spec says
  Linux. This is the main open risk: the showrooms may need a Windows binary. It is
  recorded as an open assumption (F-24) so the owner can decide on a follow-up.
- **Alternatives considered**: adding a `windows-latest` MinGW job now (out of scope and
  not verified against Borland behaviour).

## R-10 Required check and how to verify the pipeline

- **Decision**: the required check is the aggregate job `build-and-test` (it needs the
  matrix job `checks` and fails unless every compiler leg passed, so the name does not
  depend on the matrix), and `RELEASING.md` lists the
  steps to require it on `master` (Settings, Branches). Verification is manual: a draft PR
  with a wrong pricing constant must show a red check and a blocked merge (SC-001, SC-002);
  a manual release run with a wrong version must fail at the version step. The workflow
  YAML is also checked locally with `actionlint` when available.
- **Rationale**: the setting lives in GitHub and cannot be verified from files (FR-007).
- **Alternatives considered**: configuring protection with `gh api` from a script (needs
  admin rights on the repository and is a standing change, so it needs the owner's
  decision).
