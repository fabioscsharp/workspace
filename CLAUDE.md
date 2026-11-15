# CLAUDE.md

Rules and guidance for AI agents working in this workspace. This file holds **no**
architecture, requirements or business rules; read the files in "Where things live".

## Where things live

| Topic | File |
|-------|------|
| Non-negotiable principles, constraints, workflow | `.specify/memory/constitution.md` |
| Current behaviour and pricing rules (baseline requirements `BR-xxx`) | `specs/000-baseline/spec.md` |
| Architecture, modules, data file formats, build, release | `docs/architecture.md` |
| Assumptions, open questions, doc-versus-code conflicts | `src/dealerdesk/assumptions.md` |
| Feature specs (plans and tasks live beside each spec) | `specs/00N-<name>/` |
| Application code (git submodule) | `src/dealerdesk` |
| Original brief and ticket (read-only) | `src/dealerdesk/BRIEF.md`, `src/dealerdesk/TICKET-142.md` |

Read the constitution first, and the baseline spec before touching pricing, quotes,
invoices or data files.

## Rules

1. **Never edit existing documentation in `src/dealerdesk`** (`README.md`, `BRIEF.md`,
   `RELEASE.txt`, `TICKET-142.md`, ...). Append only; revisions go in a dated section at
   the bottom. New documentation goes in new files.
2. **Record every assumption** in `src/dealerdesk/assumptions.md` (ID, statement, why,
   status) when something is not explicit or raises a question. Append entries, never
   delete; only a status may change in place.
3. **Code beats docs** when they disagree (the docs are stale). Do not "fix" existing
   behaviour unless a spec says so.
4. **Keep the knowledge files current**: when behaviour changes, update
   `specs/000-baseline/spec.md` in the same change; when structure or data formats change,
   update `docs/architecture.md`. Do not copy their content into this file.
5. Put new requirements in a spec under `specs/`, not in code comments or this file.
6. Work in `src/dealerdesk` from its repo root; restore sample data after manual runs
   (`git checkout -- data`).
7. Commit in the submodule first, then commit the updated submodule pointer here. Do not
   commit or push unless asked.

## Workflow

Use the spec-kit skills in `.claude/skills`, in order: `/speckit-specify`,
`/speckit-clarify`, `/speckit-plan`, `/speckit-tasks`, `/speckit-analyze`,
`/speckit-implement`, `/speckit-converge`. Existing specs were written by hand and still
have `[NEEDS CLARIFICATION]` markers: run `/speckit-clarify` before planning them.

## Environment (Windows)

`specify` and `uv` are installed per user. Set `UV_PYTHON_INSTALL_DIR=C:\uv-python`
(the profile path has a non-ASCII character that breaks uv's default). Non-interactive
spec-kit init needs `--non-interactive --ignore-agent-tools --script ps`.
