# Contract: GitHub Actions workflows (`src/dealerdesk/.github/workflows/`)

## `ci.yml` (name: `CI`)

| Item | Value |
|------|-------|
| Triggers | `push` to `master`; `pull_request` (all branches) |
| Permissions | `contents: read` |
| Concurrency | group per workflow and ref; cancel in-progress runs on pull requests |
| Job `checks` | the matrix job, one leg per compiler (`checks (gcc)`, `checks (clang)`) |
| Runner | `ubuntu-latest` |
| Matrix | `cc: [gcc, clang]`, exported as `CC`; `fail-fast: false` so both results show |
| Steps | `actions/checkout@v4` with `fetch-depth: 0`; run `sh scripts/check.sh`; on a push to `master` with `gcc` only, `actions/upload-artifact@v4` of `dealerdesk` named `dealerdesk-linux-<sha>` |
| Fails when | `check.sh` exits non-zero (any test, mutation, compat, release test or changed tree) |
| Timeout | 10 minutes per leg |
| Job `build-and-test` | **the required check (FR-007).** `needs: checks`, `if: always()`, passes only if the `checks` result is `success`. One stable name that does not change if the matrix changes. Timeout 2 minutes. |

## `release.yml` (name: `Release`)

| Item | Value |
|------|-------|
| Triggers | `push` of tags matching `[0-9]*.[0-9]*`; `workflow_dispatch` with required input `version` (string, e.g. `2.4`) |
| Permissions | `contents: write` |
| Concurrency | one release run per version; never cancelled |
| Job | `release`, `ubuntu-latest`, `CC=gcc`, timeout 15 minutes |

Steps, in order (each failure stops the run, nothing is published):

1. Checkout with `fetch-depth: 0`.
2. Resolve `VERSION`: the tag name on a tag push, the `version` input on a manual run. It
   is passed to every step through an `env:` variable and used only as a quoted shell
   variable (`"$VERSION"`). It is never written as `${{ ... }}` text inside a `run:` script,
   so a crafted tag or input cannot inject commands.
3. `sh scripts/version.sh --check "$VERSION"`: exits non-zero if it differs from the header
   version or has a leading "v" (FR-002, US2 1a).
4. Fail if a Release named `$VERSION` already exists (`gh release view`).
5. `sh scripts/check.sh` (the same list as CI).
6. `sh scripts/package.sh "$VERSION"` writes the zip and the `.sha256` into `dist/`.
7. `sh scripts/release-notes.sh "$VERSION" > dist/notes.md`.
8. `gh release create "$VERSION" dist/*.zip dist/*.sha256 --title "DealerDesk $VERSION"
   --notes-file dist/notes.md --target "$GITHUB_SHA"`. On a tag push the tag already
   exists and is used as is; on a manual run `gh` uses the tag if it already exists
   (it is never moved), otherwise creates it at that commit. For a tag that already exists
   on the remote (such as `2.4`, on a commit without this workflow), no push event fires, so
   only a manual run can release it; the build then comes from the commit the run started on,
   and the owner first proves the application sources and `data/` are identical to the tag
   (`git diff <tag> HEAD -- '*.c' '*.h' data` is empty).
9. `actions/upload-artifact@v4` of `dist/` (a copy kept with the run).

Guarantees:

- The release path runs the same `check.sh` as CI, with the same history depth.
- A mismatch between tag (or input) and header never reaches packaging.
- No existing Release is overwritten or edited.
- Secrets used: only the automatic `GITHUB_TOKEN` (as `GH_TOKEN`).
