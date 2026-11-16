# Data Model: CI and release (003)

There is no application data change. These are the artefacts the delivery tooling creates
and checks. Fields marked (derived) are computed, not stored.

## Version

| Field | Source | Rule |
|-------|--------|------|
| `header_version` | `DEALERDESK vX.Y` in `screen.c` | the single source (R-1); digits and dots only, format `X.Y` or `X.Y.Z` |
| `requested_version` | pushed tag name, or the `version` input of a manual run | same format, no leading "v" |
| match | (derived) | `requested_version == header_version`; otherwise the run fails before packaging |

## CI run

| Field | Value |
|-------|-------|
| trigger | push to `master`, or pull request |
| matrix | `CC` in `gcc`, `clang` |
| steps | checkout (full history), `scripts/check.sh` |
| result | pass only if every check passes and the tree is unchanged by the checks |
| artifact | the `dealerdesk` binary, uploaded only for gcc on `master` |

## Check list (`scripts/check.sh`)

| Order | Command | Fails when |
|-------|---------|------------|
| 1 | `make clean && make test` | any unit or e2e case fails |
| 2 | `make test-mutation` | the mutated build passes (the suite would miss a pricing error) |
| 3 | `make test-compat` | the `a2ba708` binary cannot read data written by the current one |
| 4 | `make test-release` | the version, package or notes script misbehaves |
| 5 | `git status --porcelain`, compared with its output before step 1 | the output differs, so a check changed or left behind a file |

## Release run

| Field | Value |
|-------|-------|
| trigger | tag `[0-9]*.[0-9]*` push, or manual run with `version` |
| gate | version match, then the full check list (gcc) |
| package | `dealerdesk-<ver>.zip` + `dealerdesk-<ver>.zip.sha256` |
| publish | GitHub Release named `<ver>` with the two files and the notes |
| refuses | an existing Release for `<ver>`; a version mismatch; any failed check |

## Release package (zip layout)

```text
dealerdesk-<ver>/
├── dealerdesk            # built with the default make (gcc), executable
└── data/
    ├── stock.dat
    ├── customers.dat
    ├── invoices.dat
    └── next_ids.dat      # the committed sample data, unmodified
```

The `.sha256` file holds one line, `<hash>  dealerdesk-<ver>.zip`.

## Release notes (derived)

Header line with the version and platform note, the block from `RELEASING.md` between the
`notes:begin` and `notes:end` markers (backup step and E-05 rollback warning), then the
checksum file name.

## Required check (GitHub setting, not a file)

Name `build-and-test` (the aggregate job that passes only when both compiler legs pass),
target branch `master`. Applied by hand and
confirmed after CI first runs (FR-007).
