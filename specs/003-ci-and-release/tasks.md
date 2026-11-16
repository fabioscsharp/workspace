---

description: "Task list for 003: CI and repeatable release"
---

# Tasks: CI and repeatable release for DealerDesk

**Input**: Design documents from `specs/003-ci-and-release/`:
- [plan.md](plan.md)
- [spec.md](spec.md)
- [research.md](research.md)
- [data-model.md](data-model.md)
- [contracts/workflows.md](contracts/workflows.md)
- [contracts/scripts.md](contracts/scripts.md)
- [quickstart.md](quickstart.md)

**Prerequisites**:
- Specs 001 and 002 are in `src/dealerdesk` (tag `2.4`, commit `e2dd49a`; `make test` green).
- Work in `src/dealerdesk` on a new branch `003-ci-and-release`, in WSL (`make`, gcc, `zip`,
  `sha256sum`).
- Run `git checkout -- data` after any manual run.
- Do not commit, push, tag or publish unless asked. Tasks marked **(needs approval)** touch
  GitHub (push, pull request, tag, release, repository settings) and wait for the owner.

**Tests**: the spec asks for proof that CI fails on a wrong pricing constant (SC-002) and
the plan adds `tests/release/test_release.sh` for the new scripts, so test tasks are
included in each story.

**Organization**:
- Phases 1 and 2 prepare the branch and the shared ground rules.
- US1 (CI, P2) is the MVP: it can ship without the release work.
- US2 (release, P3) builds on US1's `scripts/check.sh` and `RELEASING.md`.
- The Polish phase updates the knowledge files and runs the whole quickstart.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: can run in parallel (different files, no dependency on an unfinished task)
- **[Story]**: US1 or US2 (spec.md user stories)
- Paths are relative to the workspace root; application paths start with `src/dealerdesk/`.

## Hard rules for every task

- No application source file changes (`main.c`, `calc.c`, `dbfile.c`, `screen.c`, their
  headers) and no change to `data/` (constitution I, III, VII).
- Existing documentation (`README.md`, `BRIEF.md`, `RELEASE.txt`, `TICKET-142.md`) is
  append-only: new dated section at the bottom, never edits (constitution IV, SC-004).
- Scripts are POSIX `sh`, called as `sh scripts/<name>.sh`, LF line endings (`.gitattributes`
  already enforces this). No `bash`-only syntax.
- The released binary is Linux only; Windows is deferred (F-24, F-31). Do not add a Windows
  job.
- The workflows contain no logic beyond checkout, calling a script and publishing;
  logic lives in `scripts/` (plan, Structure Decision).

---

## Phase 1: Setup

- [X] T001 In `src/dealerdesk`, create branch `003-ci-and-release` from the current HEAD (`e2dd49a`), then record a green baseline in WSL: `make clean && make test`, `make test-mutation`, `make test-compat`. Note the results in the Implementation notes at the bottom of this file.
- [X] T002 [P] Append `dist/` on a new line to `src/dealerdesk/.gitignore` (package output, contracts/scripts.md "check.sh rules").

---

## Phase 2: Foundational

**Purpose**: confirm the assumptions the workflows depend on, before writing them.

- [X] T003 Confirm the build honours an environment compiler: in WSL run `CC=gcc make clean all` and `CC=gcc sh tests/compat.sh`, and confirm both pass (research R-4: `CC ?= cc` in `src/dealerdesk/Makefile`, `${CC:-cc}` in `src/dealerdesk/tests/compat.sh`). If clang is installed, repeat with `CC=clang`; if not, say so in the notes (the clang leg is first proven in GitHub).
- [X] T004 [P] Confirm tool availability for the scripts: `zip`, `unzip`, `sha256sum`, `mktemp`, `git` exist in WSL, and note the GitHub runner equivalents (`ubuntu-latest` has all of them and `gh`). Check whether `actionlint` is installed; if not, note that workflow linting is skipped locally (research R-10).

**Checkpoint**: both prerequisites hold. US1 can start.

---

## Phase 3: User Story 1 - CI runs build and tests on every change (Priority: P2) MVP

**Goal**: every push to `master` and every pull request builds and tests on a clean machine,
and a failing result blocks the merge (FR-001, FR-007, SC-001, SC-002).

**Independent Test**: a draft PR with a wrong pricing constant shows a red `build-and-test`
check and a blocked merge; reverting it makes the check green (spec US1 Independent Test).

- [X] T005 [US1] Create `src/dealerdesk/scripts/check.sh` per contracts/scripts.md: run, in order, `make clean && make test`, `make test-mutation`, `make test-compat`, then the clean-tree check: save `git status --porcelain` before the first step and compare it with the output after the last step; any difference fails (this works in a development tree that already has untracked files and in a clean CI checkout, A4). Run every step even after a failure, print `check: ALL PASSED` or `check: FAILED at <step>[, <step>]`, exit 0 only if all passed, and let `CC` pass through to `make`. The `make test-release` step is added later (T020).
- [X] T006 [US1] Run `sh scripts/check.sh` in `src/dealerdesk` in WSL. Expected: `check: ALL PASSED` and `git status --porcelain` is the same as before the run (only the new files you created). This is US1 acceptance scenario 1 locally and SC-001.
- [X] T007 [US1] Prove the check fails on a wrong pricing constant (SC-002, local): temporarily change the VAT constant in `src/dealerdesk/calc.c` (`0.20` to `0.21`), run `sh scripts/check.sh`, confirm `check: FAILED at make test`, then restore with `git checkout -- calc.c` and confirm `git status --porcelain` is the same as before the experiment.
- [X] T008 [US1] Create `src/dealerdesk/.github/workflows/ci.yml` per contracts/workflows.md: name `CI`; triggers `push` to `master` and `pull_request`; `permissions: contents: read`; a concurrency group per workflow and ref that cancels superseded pull request runs; job `build-and-test` on `ubuntu-latest` with `timeout-minutes: 10`, matrix `cc: [gcc, clang]` exported as `CC`; steps `actions/checkout@v4` with `fetch-depth: 0` and `sh scripts/check.sh`; and, only for `gcc` on a push to `master`, `actions/upload-artifact@v4` of `dealerdesk` named `dealerdesk-linux-<sha>`. The build runs with the `Makefile` default warning flag `-Wall` and no `-Werror`; the job log must show the compiler line (US1 scenario 1, A8).
- [X] T009 [P] [US1] Lint `src/dealerdesk/.github/workflows/ci.yml` with `actionlint` if available (T004); fix every finding. If it is not installed, re-read the YAML against contracts/workflows.md and note that it was not linted.
- [X] T010 [P] [US1] Create `src/dealerdesk/RELEASING.md` with the CI part only: what CI runs (point to `scripts/check.sh`), how to run the same checks locally, and a "Required check" section with the manual steps for the owner (GitHub: Settings, Branches, rule for `master`, "Require status checks to pass", add `build-and-test`, covering both compiler legs) and how to confirm it (the throwaway PR of T012). Say that actions are pinned to a major version and that pinning to commit SHAs is a possible later step (F-27). The manual release steps and the notes block are added in T016.
- [X] T011 [US1] **(needs approval)** Push the branch to `fabioscsharp/dealerdesk` and open a pull request to `master`. Confirm `build-and-test` runs for `gcc` and for `clang` and both pass. If the `clang` leg fails, fix the cause in `scripts/` or the workflow (not in the application) and record the finding in the Implementation notes.
- [X] T012 [US1] **(needs approval)** Draft PR proof (quickstart section 3, step 2): on a throwaway branch change the VAT constant in `calc.c`, open a draft PR, and confirm both legs fail. Close it without merging. Confirm that the artifact upload appears only after a green run on `master` (US1 acceptance scenario 2) once the real PR is merged.
- [X] T013 [US1] **(needs approval)** Owner applies the required check on `master` (steps in `RELEASING.md`, T010) and confirms with the throwaway PR from T012 that the merge button is blocked while the check is red (FR-007, SC-001). Record who applied it and when in the Implementation notes.

**Checkpoint**: US1 is complete and independently shippable. CI protects `master`.

---

## Phase 4: User Story 2 - Repeatable release steps (Priority: P3)

**Goal**: a tag (or a manual run) produces a verified, versioned package published as a
GitHub Release, with the manual steps written down (FR-002 to FR-005).

**Independent Test**: on a clean checkout, `sh scripts/package.sh 2.4` yields a zip with the
binary and sample data and a matching checksum, and a wrong version is refused (spec US2
Independent Test).

### Scripts and their tests

- [X] T014 [P] [US2] Create `src/dealerdesk/scripts/version.sh` per contracts/scripts.md: with no argument print the version from the `DEALERDESK vX.Y` header in `screen.c` (digits and dots only, without the "v"), exit 2 if it cannot be read; with `--check <ver>` print nothing on success, and exit 1 with a one-line reason on stderr if `<ver>` differs from the header, has a leading "v", or is not `X.Y` or `X.Y.Z`. Read `screen.c` with `sed`; do not modify it (F-26).
- [X] T015 [P] [US2] Create `src/dealerdesk/scripts/package.sh` per contracts/scripts.md and data-model.md: require `<ver>`, run `version.sh --check`, refuse if `git diff --quiet -- data` fails, run `make`, then write `dist/dealerdesk-<ver>.zip` (one top folder `dealerdesk-<ver>/` holding the executable `dealerdesk` and the four committed `data/*.dat` files, built with `zip -X` in a fixed order) and `dist/dealerdesk-<ver>.zip.sha256` (one line, `<hash>  dealerdesk-<ver>.zip`, verifiable with `sha256sum -c` from inside `dist/`). Print the two file names; exit 1 on any failure.
- [X] T016 [US2] Extend `src/dealerdesk/RELEASING.md` (after T010) with the manual steps: back up `data/` on the file server first, rename the old binary to `dealerdesk.old`, copy the new one, e-mail the showrooms, and the rollback steps. Between the markers `<!-- notes:begin -->` and `<!-- notes:end -->` put the release-notes text: the Linux x86-64 platform note (F-24), the "back up the whole data folder before deploying" step, and the rollback warning that `dealerdesk.old` drops every fleet flag and every fleet discount amount on its first save (E-05). Keep the wording consistent with the 2.4 section already in `RELEASE.txt` (FR-004). Describe how to start a release: push a new tag like `2.5`, or run the `Release` workflow by hand with the version. State that a tag that already exists on the remote does not fire a push event again, so it can only be released by hand (it is used as is and never moved), and that the build then comes from the commit the run was started on (A1).
- [X] T017 [US2] Create `src/dealerdesk/scripts/release-notes.sh` per contracts/scripts.md (after T016): print Markdown notes with the version, the block between the `notes:begin` and `notes:end` markers in `RELEASING.md`, and the checksum file name; exit 1 if either marker is missing.
- [X] T018 [US2] Create `src/dealerdesk/tests/release/test_release.sh` with the six cases of contracts/scripts.md: (1) `version.sh` prints the number in `screen.c`; (2) `--check` accepts it and rejects a different number, `v2.4` and `abc`; (3) `package.sh` creates a zip with exactly the expected entries and a verifying checksum; (4) `package.sh` fails on a version mismatch; (5) `release-notes.sh` output contains the version, the backup step and the rollback warning, and fails when the markers are missing (test on a temporary copy of `RELEASING.md`); (6) `git status --porcelain` is the same before and after the test run. Work in temporary directories and remove `dist/` output it creates. Print a pass/fail summary and exit 0 only if every case passes.
- [X] T019 [US2] Append a new `test-release` target to `src/dealerdesk/Makefile` that runs `sh tests/release/test_release.sh`, with its own `.PHONY: test-release` line and a comment line. Do not edit any existing target or line.
- [X] T020 [US2] Add the `make test-release` step to `src/dealerdesk/scripts/check.sh` (order 4, before the clean-tree check) so CI and release run it too. Re-run `sh scripts/check.sh` and confirm `check: ALL PASSED`.

### Workflow and docs

- [X] T021 [US2] Create `src/dealerdesk/.github/workflows/release.yml` per contracts/workflows.md: name `Release`; triggers `push` of tags matching `[0-9]*.[0-9]*` and `workflow_dispatch` with a required string input `version`; `permissions: contents: write`; a concurrency group per version that never cancels; job `release` on `ubuntu-latest` with `CC=gcc` and `timeout-minutes: 15`. Steps in order: checkout with `fetch-depth: 0`; resolve `VERSION` (tag name or input) into a step `env:` variable (`VERSION: ${{ github.event.inputs.version || github.ref_name }}`) and only ever use it as a quoted shell variable `"$VERSION"`, never as `${{ ... }}` text inside a `run:` script (A3); `sh scripts/version.sh --check "$VERSION"` (this validates the format before any other use); fail if `gh release view "$VERSION"` finds a Release; `sh scripts/check.sh`; `sh scripts/package.sh "$VERSION"`; `sh scripts/release-notes.sh "$VERSION" > dist/notes.md`; `gh release create "$VERSION" dist/*.zip dist/*.sha256 --title "DealerDesk $VERSION" --notes-file dist/notes.md --target "$GITHUB_SHA"` with `GH_TOKEN: ${{ github.token }}`; `actions/upload-artifact@v4` of `dist/`.
- [X] T022 [P] [US2] Lint `src/dealerdesk/.github/workflows/release.yml` with `actionlint` if available; fix every finding, or note that it was not linted.
- [X] T023 [US2] Run quickstart section 2 locally in WSL: `sh scripts/package.sh 2.4`, `(cd dist && sha256sum -c dealerdesk-2.4.zip.sha256)`, `unzip -l dist/dealerdesk-2.4.zip` (expect `dealerdesk-2.4/dealerdesk` and the four `data/*.dat`), and `sh scripts/release-notes.sh 2.4` (expect the backup step and the rollback warning). Delete `dist/` afterwards and confirm `git status --porcelain` is the same as before.
- [X] T024 [US2] Append a dated section to the bottom of `src/dealerdesk/RELEASE.txt` that points to `RELEASING.md` and to the CI and release workflows, and keeps the backup and rollback wording consistent (FR-003, FR-004). Then confirm nothing was removed or changed: `git diff -- RELEASE.txt | grep '^-' | grep -v '^---'` prints nothing (SC-004).
- [X] T025 [US2] **(needs approval)** Release the existing tag `2.4` (SC-003, FR-005; decided 2026-11-16: reuse it, do not wait for the next version; A1). Facts checked 2026-11-16: `2.4` is a lightweight tag on commit `49fb770`, which has no `.github/`, and the tag is already on the remote, so pushing it again fires no event and a tag push cannot start the workflow. Steps: (a) push the branch and merge it into `master`; (b) in `src/dealerdesk` prove the application is unchanged since the tag: `git diff 2.4 HEAD -- '*.c' '*.h' data` must print nothing, otherwise stop and tell the owner; (c) run the `Release` workflow **by hand** from `master` with version `2.4` (`gh release create` uses the existing tag as is, and never moves it); (d) confirm the GitHub Release has the zip, the `.sha256` file and the notes, then download both and run `sha256sum -c`. Do not force-move or delete the published tag. The tag-push path (US2 scenario 1) is first proved at the next version tag, which has `release.yml` in its commit (see T026).
- [ ] T026 [US2] **(needs approval)** Guards (quickstart section 6, run after T025): (a) run the `Release` workflow by hand with version `9.9` and confirm it fails at the version step with nothing published and no tag created; (b) run it again with `2.4` and confirm it fails with "Release already exists" and changes nothing; (c) also try a hostile input such as `2.4; echo pwned` and confirm it is rejected by the version check without running the extra command (A3). The paths "a manual run creates a new tag" and "a tag push starts the release" cannot be exercised with `2.4`; they are proved at the next real version (or on a throwaway repository) and recorded in the Implementation notes.

**Checkpoint**: US1 and US2 both work. The release process is a tag or one manual run.

---

## Phase 5: Polish and cross-cutting

- [X] T027 [P] (FR-006) Append the implementation findings to `src/dealerdesk/assumptions.md`: record the outcome of each planned decision (F-21 to F-30), change statuses in place only (for example F-05 `assumed` to `confirmed`, F-26 to F-30 `assumed` to `confirmed` once verified), and add new F-notes (starting at F-32) for anything learned, such as the `clang` leg result and the tag `2.4` question. Never delete or reword existing rows.
- [X] T028 [P] Update `docs/architecture.md` "Build and run" and "Release (today)" for the new CI, `scripts/`, workflows and `RELEASING.md`, and move the old "Planned structural changes" entry for `003` to the current description (CLAUDE.md rule 4).
- [X] T029 [P] Update `specs/000-baseline/spec.md` BR-082 so it says CI and a release workflow now exist (CLAUDE.md rule 4). No `BR-` behaviour changes.
- [X] T030 [P] Append a dated section to `src/dealerdesk/NOTES.md`: what spec 003 added and why, how it was verified, and what is next (Windows build deferred, F-24 and F-31; required-check owner step; pinning actions to commit SHAs). Keep the file within two pages.
- [ ] T031 Run the whole quickstart (sections 1 to 7) for the parts that need no approval, run `sh scripts/check.sh` one last time, and run `git checkout -- data`. Confirm `git status` in `src/dealerdesk` shows only the intended new and appended files, and that no application source file, `README.md`, `BRIEF.md` or `TICKET-142.md` changed.
- [ ] T032 Run `/speckit-analyze` on spec 003 and fix any inconsistency it finds between `spec.md`, `plan.md` and this file.
- [ ] T033 **(needs approval)** Commit in `src/dealerdesk` first, then commit the updated submodule pointer and the workspace changes (`specs/003-ci-and-release/`, `docs/architecture.md`, `specs/000-baseline/spec.md`). Do not push unless asked.

---

## Dependencies and execution order

- **Phase 1 to Phase 2**: T001 first; T002, T003 and T004 can then run (T002 and T004 in parallel).
- **US1 (T005 to T013)**: needs Phase 2. T005 then T006 and T007 (sequential, same working tree); T008 needs T005; T009 and T010 run in parallel after T008; T011 to T013 need approval and run in order.
- **US2 (T014 to T026)**: scripts T014 and T015 run in parallel and need only Phase 1. T016 needs T010 (same file). T017 needs T016. T018 needs T014, T015 and T017. T019 needs T018. T020 needs T005 and T019. T021 needs T014, T015, T017 and T020. T022 to T024 follow. T025 and T026 need approval and a merged branch.
- **US2 does not block US1**: US1 can be merged and the required check applied before US2 starts.
- **Polish**: T027 to T030 can run in parallel once the stories they describe are done; T031 to T033 run last in order.

### Parallel opportunities

- T002 with T004 (setup).
- T009 with T010 (after T008).
- T014 with T015 (independent scripts).
- T022 with T023 (after T021).
- T027, T028, T029 and T030 (different files).

## Implementation strategy

- **MVP**: Phases 1 and 2 plus US1 (T001 to T013). That alone makes CI run on every change
  and block bad merges, which answers "nervous manual testing".
- **Increment 2**: US2 (T014 to T026) turns the manual release into a tag or one click, and
  needs US1's `check.sh`.
- **Approval gates**: every GitHub step (T011 to T013, T025, T026, T033) waits for the owner.
  Everything else can be verified locally in WSL.

## Implementation notes

### 2026-11-16: T001 to T010 done (local only, WSL Ubuntu, gcc 15.2, GNU make 4.4.1)

**Results**
- T001: branch `003-ci-and-release` created in `src/dealerdesk` from `e2dd49a`. Baseline:
  `make test` unit 40/40 and e2e 29/29, `make test-mutation` killed, `make test-compat` OK.
- T002: `dist/` appended to `.gitignore`.
- T003: `CC=gcc make clean all` and `CC=gcc sh tests/compat.sh` pass. `clang` is not
  installed in WSL, so the clang leg is first proved on GitHub (`ubuntu-latest` has clang).
- T005, T006: `scripts/check.sh` written. `sh scripts/check.sh` prints `check: ALL PASSED`
  and `git status --porcelain` is identical before and after.
- T007: with VAT changed to 21% in `calc.c`, `check.sh` exits 1 with `FAILED at make test,
  make test-mutation` (the mutation step also fails, because it first needs the unmodified
  suite to pass). `calc.c` restored, status unchanged.
- T008: `.github/workflows/ci.yml` written (matrix job `checks`, aggregate job
  `build-and-test`). YAML parses with PyYAML and has the expected jobs.
- T010: `RELEASING.md` written with the CI part, the local command and the required-check
  steps. The release steps and the notes block are still to come (T016).

**Findings and deviations**
- **CRLF in the checkout (environment, not the repo).** The first baseline failed 8 of 29
  e2e scenarios. Seven tracked files (`data/*.dat`, `dbfile.h`, `screen.h`, `BRIEF.md`,
  `TICKET-142.md`) had CRLF endings, written when the submodule was first checked out at
  `a2ba708` with `core.autocrlf=true`, before `.gitattributes` (`eol=lf`) existed. Git did
  not rewrite them when the checkout moved to `e2dd49a`, because they did not change between
  the two commits. Fixed locally with `git checkout HEAD -- data BRIEF.md TICKET-142.md
  dbfile.h screen.h`; no commit was needed. A fresh clone is not affected. To record in
  `assumptions.md` (T027) next to F-18.
- **Required check name.** With a matrix, GitHub names the checks `checks (gcc)` and
  `checks (clang)`. Added the aggregate job `build-and-test` so the required check has one
  stable name. `contracts/workflows.md`, `data-model.md`, `research.md` R-10 and
  `RELEASING.md` were updated to match.
- **T004:** WSL has `make`, `gcc`, `git`, `sha256sum`, `mktemp`, but **not** `zip`, `unzip`,
  `clang` or `actionlint`. `zip` is needed from T015 (the GitHub runner has it); installing
  it in WSL needs `sudo apt install zip unzip`, which waits for approval.
- **T009:** not linted (`actionlint` not installed; no Go or Docker to build it, and
  downloading a binary needs approval). The file was compared by hand against
  `contracts/workflows.md`, and the contract was updated where the file improved on it.
- `make test-release` is not part of `check.sh` yet (added in T020).

**Still open from this phase**
- T011 to T013 need approval (push, pull request, draft PR proof, required check setting).
- The clang leg and the artifact upload are untested until the first run on GitHub.

### 2026-11-16 (later): T011 done, T012 half done

- **T011:** branch `003-ci-and-release` pushed (commit `f68c948`), PR
  `fabioscsharp/dealerdesk#2` opened. `checks (gcc)`, `checks (clang)` and `build-and-test`
  all passed on GitHub, so the clang leg works. The artifact upload is untested until a push
  to `master`.
- **Stray PR:** a second PR, `felipeamarante/dealerdesk#1` (upstream of the fork), had been
  opened from the owner's account 1 min 46 s before #2, with a default title and no body. It
  was not created by `gh pr create` here; it was closed with a comment on the owner's say-so.
  GitHub's compare link defaults to the upstream repo for a fork, so always pass
  `--repo fabioscsharp/dealerdesk` to `gh`.
- **T012 (red CI part, done):** throwaway PR #3 (base `003-ci-and-release`, VAT `0.20` to
  `0.21`) made all three checks fail (`unit: 9 passed, 31 failed`, `e2e: 20 passed, 9 failed`,
  `check: FAILED at make test, make test-mutation`). PR closed and branch deleted.
- **T012 (blocked-merge part, open):** needs #2 merged and the `build-and-test` rule on
  `master` first (T013). Then repeat with a throwaway PR whose base is `master` and confirm
  the merge button is disabled.
- `gh` is installed but not on the Git Bash PATH here; use `/c/Program Files/GitHub CLI/gh.exe`.

### 2026-11-16 (later still): T012 and T013 done

- **T013:** the owner applied the rule on `master` (checked through the API at that time): required check `build-and-test`, `enforce_admins` false, no rulesets.
- **T012 (blocked-merge part):** throwaway draft PR #4 (base `master`, built on #2's branch, VAT `0.21`): `checks (gcc)`, `checks (clang)` and `build-and-test` all failed and `mergeStateStatus` was `BLOCKED`. PR closed, branch deleted. A real merge was deliberately not attempted.
- **Note:** `enforce_admins` is false, so repository admins can still merge past a red check. If that should not be possible, tick "Do not allow bypassing the above settings" in the rule.
- **Still open:** PR #2 is not merged. The artifact upload and the `master` push run are untested until it is.

### 2026-11-16 (merge)

- PR #2 merged into `master` (merge commit `ba99a15`, normal merge, no bypass; all three checks
  were green and the merge state `CLEAN`). User story 1 is on `master`. The local submodule
  `master` is fast-forwarded to it.
- Still to check on `master`: the push-triggered CI run and the `dealerdesk-linux-<sha>`
  artifact (US1 scenario 2).

### 2026-11-16 (release scripts): T014 to T024 done locally, branch `003-release-scripts`

- **Written** (in `src/dealerdesk`, on branch `003-release-scripts` from `master` at `ba99a15`, not yet
  committed): `scripts/version.sh`, `package.sh`, `release-notes.sh`, `tests/release/test_release.sh`,
  `.github/workflows/release.yml`, the release sections and the marked notes block in `RELEASING.md`,
  a `test-release` target appended to `Makefile`, the `make test-release` step in `scripts/check.sh`,
  and an appended dated section in `RELEASE.txt` (original text byte-identical, 0 lines removed).
- **Local results (WSL):** `sh scripts/check.sh` prints `check: ALL PASSED`; `make test-release` gives
  5 passed, 0 failed, 1 skipped. The skipped case is 3 (zip and checksum), because `zip` is not installed
  in WSL and `sudo` needs a password. With `CI=true` and no zip the same case fails, so it can never
  be skipped silently in CI.
- **T023 (partly):** the packaging flow was run with a stand-in `zip` (a small Python script kept
  outside the repo): all 6 cases passed, the zip held exactly `dealerdesk-2.4/dealerdesk` and the
  four `data/*.dat`, and `sha256sum -c` printed OK. The real `zip -sf` output format (read by test
  case 3) is **not yet confirmed**; the first CI run is the proof.
- **T022:** `release.yml` was not linted (no `actionlint`). It parses as YAML, has the two triggers
  and the `version` input, and no `run:` script contains a `${{ }}` expression (F-33).
- **Not yet done:** T025 and T026 (an actual release, needs approval), and the polish tasks T027 to T033.

### 2026-11-16 (docs): T027 to T030 done, on submodule branch `notes-and-docs` (not committed)

- T027: `assumptions.md` gained F-35 to F-40 (CRLF checkout finding, aggregate required check,
  zip test policy, existing empty Release `2.4`, stray upstream PR, `enforce_admins`); F-05 set to
  confirmed and F-25 points to F-36. Only those two status cells changed.
- T028: `docs/architecture.md` updated (build and run, release, planned changes).
- T029: `specs/000-baseline/spec.md` BR-082 updated (CI and release workflow exist).
- T030: `NOTES.md` rewritten to the four sections the brief requires, 799 words (about two pages).
- Still open: T025 and T026 (first release, F-38), T031 to T033 (final run, analyse, commits).

### 2026-11-16 (release 2.4): T025 done, T026 partly proved

- **T025:** the empty Release "V 2.4" was deleted with the owner's approval (the tag `2.4` stayed at
  `49fb770`), and the `Release` workflow was run by hand from `master` (`28d376b`) with version `2.4`.
  Before that, `git diff 2.4 origin/master -- '*.c' '*.h' data` was empty. Run 43868527799 passed
  every step and published Release "DealerDesk 2.4" with `dealerdesk-2.4.zip` (13875 B),
  `dealerdesk-2.4.zip.sha256` and the notes; GitHub recorded the SHA256 digests. The zip was not
  downloaded or checked with `sha256sum -c` here.
- **Earlier runs:** run 43866726620 stopped at the existing-release guard (as designed, the version
  check before it passed); run 43866935324 stayed "queued" with no jobs for 10+ minutes and was
  abandoned (a GitHub queue problem; the cancel call said "already completed").
- **T026, still open:** no run with a wrong version (`9.9`) or hostile input yet, and no rerun of `2.4`
  now that a real Release exists. The paths "tag push starts the workflow" and "a manual run creates
  a new tag" need a new version. Recorded as F-41 in `assumptions.md`.
- **Open:** T031 to T033 (final check, `/speckit-analyze`, commits).
