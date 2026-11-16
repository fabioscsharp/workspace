# Feature Specification: CI and repeatable release for DealerDesk

**Feature Branch**: `003-ci-and-release` (in `src/dealerdesk`)

**Created**: 2026-11-15 (split from the original `001-delivery-pipeline`)

**Status**: Clarified, planned and tasked 2026-11-16; analysed, ready for
`/speckit-implement`. Depends on `001-build-and-tests`.

**Baseline**: changes no `BR-` behaviour. Structure: `docs/architecture.md`.

**Related**: `001-build-and-tests` provides the build command and the test suite this spec
runs automatically. `002-fleet-discount` does not wait for this spec (its tests run locally
until CI exists).

**Input**: BRIEF.md task 1: "Improve how changes get delivered on this codebase", the part
about no CI and hand-made releases (`RELEASE.txt`).

## Clarifications

### Session 2026-11-16

- Q: What should start a release: pushing a version tag, or a person manually running the
  release workflow? → A: Both. A version tag starts it, and a manual run is allowed as a
  fallback; the manual run applies the same version check as a tag.
- Q: What format should a release tag have, with or without a leading "v"? → A: Plain
  version numbers such as `2.4`, matching the existing tag. The version check compares the
  tag (or the version typed for a manual run) with the number in the on-screen header
  (`v2.4`, without the "v").
- Q: Besides `make test`, which other checks should CI run? → A: Both `make test-mutation`
  and `make test-compat`; all three are required to pass. CI checks out the full git
  history, because the compatibility check builds the `a2ba708` sources from it.
- Q: What should a release produce? → A: A versioned zip (binary plus sample data), published
  as a GitHub Release together with a SHA256 checksum file and release notes that include
  the "back up `data/`" step and the rollback warning (E-05).
- Q: Should a failing CI check block merging into `master`? → A: Yes. CI must be a required
  check. The GitHub setting is outside the repo files, so it is written down as a manual
  step in the new CI/release document, and applying it is confirmed once CI first runs.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - CI runs build and tests on every change (Priority: P2)

Every push to `master` and every pull request builds the app and runs the tests on a clean
machine (GitHub Actions), with the result visible on the PR.

**Why this priority**: the team does not trust changes; automatic evidence replaces
nervous manual testing.

**Independent Test**: push a branch with a deliberately wrong pricing constant and see CI
fail; revert and see it pass.

**Acceptance Scenarios**:

1. **Given** a pull request, **When** CI runs, **Then** it builds with warnings enabled
   and runs all the checks of FR-001, and fails if any of them fails.
2. **Given** a green run on the default branch, **Then** the built binary is available as
   a CI artifact.

---

### User Story 2 - Repeatable release steps (Priority: P3)

The manual release checklist is turned into scripts plus a CI-built release package, so the
steps that can be automated are, and the manual ones (copy to the file server, e-mail
showrooms, rollback copy) are written down in a new file. The existing `RELEASE.txt` is
only appended to, never rewritten.

**Why this priority**: valuable but the team cannot be reached from here (no file server
access).

**Independent Test**: running the packaging step on a clean checkout yields a versioned
archive containing the binary and the sample data, with the version taken from one place.

**Acceptance Scenarios**:

1. **Given** a tagged commit, **When** the release workflow runs, **Then** it publishes a
   GitHub Release with the versioned zip (binary plus sample data), its SHA256 file and
   the release notes. A tag that already exists on the remote does not start the workflow
   again; it is released by a manual run, using the tag as is (the existing `2.4`).
1a. **Given** a manual run of the release workflow, **When** the requested version equals
   the version in the source header, **Then** it produces the same artifact as a tag run;
   **When** it differs, **Then** the run fails before building the artifact.
2. **Given** `RELEASE.txt`, **When** this feature is done, **Then** its original content
   is intact and any change is an appended, dated section at the bottom.
3. **Given** the release checklist, **Then** it includes backing up `data/` before
   deploying and a warning that rolling back to `dealerdesk.old` loses fleet data (E-05).

---

### Edge Cases

- CI must run from the repo root so `./data` resolves, on a temporary copy of the data.
- The build in CI uses warnings but not `-Werror` (legacy code).
- A release tag with a version different from the header version: the single version
  source decides, and the mismatch fails the release step.
- The workflow file lives inside the dealerdesk repo (`.github/workflows`), a new file.
- A tag that already exists on the remote (the existing `2.4`, on a commit without the
  workflow) starts nothing. It is released by a manual run, the tag is used as is and
  never moved, and the owner first proves the application sources and `data/` are
  identical to the tag.
- The version typed for a manual run, or a tag name, is untrusted text: it is validated
  before any other use and never written into a script as workflow expression text.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: A CI workflow MUST build and run all tests (from spec 001) on every push to
  `master` and every pull request, and fail on any error. "All tests" means `make test`, `make test-mutation` and
  `make test-compat`, each required to pass, with the full git history checked out.
- **FR-002**: A single place MUST define the version string used by the release step.
- **FR-003**: The release procedure MUST be scripted or automated where possible; manual
  steps are documented in a new file and `RELEASE.txt` is append-only.
- **FR-004**: The release checklist MUST include backing up `data/` before deploying (E-05;
  the appended step in `RELEASE.txt` is added by spec 002 when fleet data first exists, and
  is kept consistent with this checklist).
- **FR-005**: Releases MUST produce a versioned artifact (binary plus sample data) from CI.
  A release MUST start from either a version tag or a manual workflow run, and both paths
  MUST run the same version check (FR-002) and the same tests before packaging. The
  release MUST be published as a GitHub Release holding the zip, a SHA256 checksum file and
  notes that repeat the backup step and the rollback warning (FR-004, E-05).
- **FR-006**: All assumptions made MUST be recorded in `assumptions.md`.
- **FR-007**: The CI workflow MUST be configured as a required status check for merging
  into `master`. Because this is a GitHub repository setting, the new CI/release document
  MUST list it as a manual step, and its application MUST be confirmed after CI first runs.

### Key Entities

- **CI workflow**: the GitHub Actions definition that builds and tests.
- **Release artifact**: the built `dealerdesk` binary plus sample data, versioned.

## Success Criteria *(measurable)*

- **SC-001**: Every pull request shows a pass/fail result for build and tests, and a
  failing result blocks the merge into `master`.
- **SC-002**: Deliberately breaking a pricing constant makes CI fail.
- **SC-003**: A tagged release yields a downloadable, versioned artifact: a GitHub Release
  with the zip and a SHA256 file that matches it.
- **SC-004**: `RELEASE.txt` original text is byte-identical apart from appended sections.

## Assumptions

See `src/dealerdesk/assumptions.md`, section F (F-05, F-06) and E-05:

- GitHub Actions is the CI provider (F-05); copying to `\\fileserver01\apps\dealerdesk\`
  and e-mailing showrooms remain manual (F-06).
- The platform is Linux (Ubuntu runner) with gcc; Windows/Borland builds stay out of scope.
- Releases start from a version tag or a manual run (clarified 2026-11-16).
- A Windows build is deferred to a later spec (decided 2026-11-16, F-24 and F-31); this
  spec releases the Linux binary only.
- Release tags are plain version numbers like `2.4`, with no leading "v" (clarified
  2026-11-16).
