# Quickstart: validate 003 (CI and release)

**Prerequisites**: the 001 setup (Linux, macOS or WSL, `make`, gcc or clang), plus `zip`
and `sha256sum`. The GitHub steps need push access to `fabioscsharp/dealerdesk` and admin
rights to set branch protection. Commands run from the submodule root, `src/dealerdesk`,
unless noted.

## 1. Local checks (US1, FR-001, no GitHub needed)

```bash
sh scripts/check.sh
```

Expected: every step passes, the last line is `check: ALL PASSED`, and `git status`
shows nothing new (SC-001).

```bash
sh scripts/version.sh
sh scripts/version.sh --check 2.4; echo "exit=$?"
sh scripts/version.sh --check v2.4; echo "exit=$?"
```

Expected: `2.4`, then `exit=0`, then a reason and `exit=1` (FR-002).

## 2. Package and notes (US2, FR-005)

```bash
sh scripts/package.sh 2.4
(cd dist && sha256sum -c dealerdesk-2.4.zip.sha256)
unzip -l dist/dealerdesk-2.4.zip
sh scripts/release-notes.sh 2.4
```

Expected: the checksum verifies; the zip lists `dealerdesk-2.4/dealerdesk` and the four
`data/*.dat` files; the notes contain the backup step and the rollback warning (E-05).
Remove `dist/` afterwards (it is git-ignored).

## 3. CI on a pull request (US1, SC-001, SC-002)

1. Push the work branch and open a pull request. Expected: `build-and-test` runs for
   `gcc` and `clang` and both pass; `master` gets a binary artifact after the merge.
2. Draft PR proof: change one pricing constant in `calc.c` (for example VAT `0.20` to
   `0.21`) on a throwaway branch. Expected: both legs fail and, with the required check on,
   the merge button is blocked. Close the PR without merging.

## 4. Required check (FR-007, manual)

In GitHub: Settings, Branches, rule for `master`, "Require status checks to pass", add
`build-and-test`. Confirm by looking at the draft PR from step 3.

## 5. Release by tag (US2, SC-003)

Tag `2.4` already exists on the remote (lightweight, on `49fb770`, which has no workflow
files), so pushing it again starts nothing. Release it by hand from `master`:

```bash
git diff 2.4 HEAD -- '*.c' '*.h' data
```

Expected: no output (the application and data are the same as in the tag). Then Actions,
Release, Run workflow, version `2.4`. Expected: the workflow passes and a GitHub Release
`2.4` exists, on the existing tag, with the zip, the `.sha256` file and the notes. Download
both and check `sha256sum -c`. A new tag (for example `2.5`) pushed later is the first test
of the tag-push path.

## 6. Release by manual run, and the mismatch guard (US2 1a)

1. Actions, Release, Run workflow, version `9.9`. Expected: it fails at the version check,
   nothing is published and no tag is created.
2. Run it again with `2.4` after step 5. Expected: it fails with "Release already exists"
   and changes nothing.
3. Run it with `2.4; echo pwned`. Expected: rejected by the version check, and the extra
   command does not run.
4. Creating a new tag from a manual run is first exercised at the next real version.

## 7. Append-only docs (SC-004)

```bash
git diff a2ba708 -- RELEASE.txt | grep '^-' | grep -v '^---'
```

Expected: no output, so nothing in the original text was removed or changed (only the
2.4 section from spec 002 and this feature's dated section were appended).
