# Contract: release scripts and `make test-release` (`src/dealerdesk`)

All scripts are POSIX `sh`, run from the submodule root, and need only `sed`, `grep`,
`zip`, `sha256sum` (or `shasum -a 256`), `mktemp` and `make`. None of them modifies a
tracked file.

| Command | Does | Output | Exit status |
|---------|------|--------|-------------|
| `sh scripts/version.sh` | prints the header version from `screen.c` | `2.4` on stdout | 0; 2 if the header cannot be read |
| `sh scripts/version.sh --check <ver>` | compares `<ver>` with the header version | nothing on success; a one-line reason on stderr | 0 equal; 1 differs, has a leading "v", or is not `X.Y[.Z]` |
| `sh scripts/check.sh` | runs the check list (`make clean`, `make test`, `make test-mutation`, `make test-compat`, `make test-release`, then the clean-tree check) | each step's output; `check: ALL PASSED` or `check: FAILED at <step>` | 0 only if every step passes |
| `sh scripts/package.sh <ver>` | builds with `make`, writes `dist/dealerdesk-<ver>.zip` and `dist/dealerdesk-<ver>.zip.sha256` | the two file names | 0; 1 if `--check <ver>` fails or a file is missing |
| `sh scripts/release-notes.sh <ver>` | prints the Markdown release notes | notes on stdout | 0; 1 if the `notes:begin/end` block in `RELEASING.md` is missing |
| `make test-release` | runs `tests/release/test_release.sh` | pass/fail summary | 0 only if every case passes |

## `check.sh` rules

- Continues to the end and reports every failing step, but exits non-zero if any failed.
- Honours the `CC` variable (passed through to `make`).
- The clean-tree check saves `git status --porcelain` before the first step and compares it
  with the output after the last step; any difference fails. It therefore works in a
  development tree that already has untracked or modified files, and in a clean CI checkout
  (where it proves the checks leave nothing behind). Build outputs listed in `.gitignore`
  (`dealerdesk`, `*.o`, `tests/unit/test_calc`, `dist/`) never appear in that output.

## `package.sh` rules

- Takes the data from the committed `data/` directory, never from a test run.
- Refuses to run if `data/` differs from the committed copy (`git diff --quiet -- data`).
- The zip holds exactly the layout in [data-model.md](../data-model.md); the `.sha256` file
  is verifiable with `sha256sum -c` from inside `dist/`.

## `test_release.sh` cases

1. `version.sh` prints the number in `screen.c`.
2. `--check` accepts the same number; rejects a different number, `v2.4`, and `abc`.
3. `package.sh` creates the zip and checksum; the zip lists exactly the expected files and
   the checksum verifies.
4. `package.sh` fails when the version does not match.
5. `release-notes.sh` output contains the version, the "back up" step and the rollback
   warning text; it fails if the markers are missing (tested on a temp copy).
6. `git status --porcelain` is the same before and after the test run.
