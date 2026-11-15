# Feature Specification: CI and repeatable release for DealerDesk

**Feature Branch**: `003-ci-and-release` (in `src/dealerdesk`)

**Created**: 2026-11-15 (split from the original `001-delivery-pipeline`)

**Status**: Draft, not started. Depends on `001-build-and-tests`; run `/speckit-clarify`
before planning.

**Baseline**: changes no `BR-` behaviour. Structure: `docs/architecture.md`.

**Related**: `001-build-and-tests` provides the build command and the test suite this spec
runs automatically. `002-fleet-discount` does not wait for this spec (its tests run locally
until CI exists).

**Input**: BRIEF.md task 1: "Improve how changes get delivered on this codebase", the part
about no CI and hand-made releases (`RELEASE.txt`).

## User Scenarios & Testing *(mandatory)*

### User Story 1 - CI runs build and tests on every change (Priority: P2)

Every push and pull request builds the app and runs the tests on a clean machine (GitHub
Actions), with the result visible on the PR.

**Why this priority**: the team does not trust changes; automatic evidence replaces
nervous manual testing.

**Independent Test**: push a branch with a deliberately wrong pricing constant and see CI
fail; revert and see it pass.

**Acceptance Scenarios**:

1. **Given** a pull request, **When** CI runs, **Then** it builds with warnings enabled
   and runs all tests, and fails if either step fails.
2. **Given** a green run on the default branch, **Then** the built binary is available as
   a CI artifact.

---

### User Story 2 - Repeatable release steps (Priority: P3)

The manual release checklist is turned into a script plus a CI-built artifact, so the
steps that can be automated are, and the manual ones (copy to the file server, e-mail
showrooms, rollback copy) are written down in a new file. The existing `RELEASE.txt` is
only appended to, never rewritten.

**Why this priority**: valuable but the team cannot be reached from here (no file server
access).

**Independent Test**: running the packaging step on a clean checkout yields a versioned
archive containing the binary and the sample data, with the version taken from one place.

**Acceptance Scenarios**:

1. **Given** a tagged commit, **When** the release workflow runs, **Then** it uploads a
   versioned binary artifact.
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

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: A CI workflow MUST build and run all tests (from spec 001) on push and pull
  request and fail on any error.
- **FR-002**: A single place MUST define the version string used by the release step.
- **FR-003**: The release procedure MUST be scripted or automated where possible; manual
  steps are documented in a new file and `RELEASE.txt` is append-only.
- **FR-004**: The release checklist MUST include backing up `data/` before deploying (E-05;
  the appended step in `RELEASE.txt` is added by spec 002 when fleet data first exists, and
  is kept consistent with this checklist).
- **FR-005**: Releases MUST produce a versioned artifact (binary plus sample data) from CI.
- **FR-006**: All assumptions made MUST be recorded in `assumptions.md`.

### Key Entities

- **CI workflow**: the GitHub Actions definition that builds and tests.
- **Release artifact**: the built `dealerdesk` binary plus sample data, versioned.

## Success Criteria *(measurable)*

- **SC-001**: Every pull request shows a pass/fail result for build and tests.
- **SC-002**: Deliberately breaking a pricing constant makes CI fail.
- **SC-003**: A tagged release yields a downloadable, versioned artifact.
- **SC-004**: `RELEASE.txt` original text is byte-identical apart from appended sections.

## Assumptions

See `src/dealerdesk/assumptions.md`, section F (F-05, F-06) and E-05:

- GitHub Actions is the CI provider (F-05); copying to `\\fileserver01\apps\dealerdesk\`
  and e-mailing showrooms remain manual (F-06).
- The platform is Linux (Ubuntu runner) with gcc; Windows/Borland builds stay out of scope.
- [NEEDS CLARIFICATION: Should releases be triggered by version tags, or by a manual
  workflow run?]
