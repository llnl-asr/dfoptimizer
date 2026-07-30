# Migration Plan: dfoptimizer

Selected: 2026-07-30. Source: git@github.com:llnl/dfoptimizer.git (default branch **main**). Target: ssh://git@czgitlab.llnl.gov:7999/dftracer/dfoptimizer.git (repo exists on czgitlab).

## Findings

- Pure-Python package (`dfoptimizer/`: runtime, planner, streaming, knobs, types, optimizer) + `main.py` service entry; deps: `structlog` (core), `pyzmq` (streaming extra), `pytest` (dev). Built with setuptools via pyproject.
- One GitHub workflow: `.github/workflows/ci.yml` (build-and-test) — matrix ubuntu-22.04 x python 3.10/3.11/3.12 via astral-sh/setup-uv; `uv sync --extra streaming --dev`, `pytest -v tests/`, then offline findings replay of `tests/data/findings.jsonl`. No publish/docker workflows.
- No `docs/` existed — authored fresh host-neutral Sphinx docs (conf.py rtd→alabaster fallback, index/introduction/api rst, requirements.txt) from README + code.
- No dftracer-group GitHub dependency URLs anywhere in code/pyproject/uv.lock. Only `github.com/LLNL/dfdiagnoser` in README prose (documentation link, not a dependency) — left untouched. **No in-place source changes.**

## Steps

1. [x] Fresh clone (main)
2. [x] gitlab remote added + verified
3. [x] CI: converted `.github/workflows/ci.yml` → `.gitlab-ci.yml` on `.corona-batch` (module python/3.13.2 + venv + pip instead of uv/setup-python; matrix collapsed to module toolchain with comment; same pytest + offline-replay commands; rules push + merge_request_event)
4. [x] Docs: authored `docs/` (conf.py, index.rst, introduction.rst, api.rst, requirements.txt) + `pages` job (corona batch; rules main + temporary gitlab-migration)
5. [x] In-place GitLab URL switch — **none needed** (no dftracer-group dep URLs); REVERT.md updated with a dfoptimizer "no in-place changes" bullet
6. [x] Local tests (YAML OK; sphinx build succeeded; pytest 43 passed, 2 xfailed; offline replay produced 2 plans)
7. [x] Commit on gitlab-migration; sync .migration/ into repo
8. [x] Push main, tags, gitlab-migration
9. [ ] Pipeline: <https://czgitlab.llnl.gov/dftracer/dfoptimizer/-/pipelines>
10. [ ] User merges after green pipeline

## Executed changes (what to undo on revert)

- `gitlab` remote added: `ssh://git@czgitlab.llnl.gov:7999/dftracer/dfoptimizer.git`
- Branch `gitlab-migration` (from main) adds only: `.gitlab-ci.yml`, `docs/` (5 files), `.migration/REVERT.md`, `.migration/dfoptimizer.md`
- No in-place source changes; `.github/` untouched; nothing pushed to origin
- Pushed to gitlab: `main`, tags (none existed), `gitlab-migration` (SHAs recorded in status log)
- Revert: remove remote + delete `gitlab-migration`; if merged, `git rm .gitlab-ci.yml` (and optionally `git rm -r docs/ .migration/`)

## Status log

- 2026-07-30: cloned, remote added, plan created.
- 2026-07-30: CI converted, docs authored, tests green (43 passed / 2 xfailed; sphinx + YAML + offline replay OK), no in-place dep changes needed, committed on gitlab-migration, pushed main + tags + gitlab-migration to czgitlab.
