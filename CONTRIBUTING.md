# Contributing

## Development environment

This repo uses [`uv`](https://docs.astral.dev/uv/) to manage the Python
environment and dev tooling (ruff, pytest).

```bash
uv sync            # create .venv and install runtime + dev dependencies
uv run pytest      # run the tests
uv run ruff check .    # lint
uv run ruff format .   # format
```

`make check` runs the CI-equivalent gate locally (lint + format check + tests).

## Commit-time hooks (run once per clone)

Commit-time hook **activation** is a separate contract from the committed
`.pre-commit-config.yaml` and from CI execution. Because `.git/hooks` is not
versioned, and because this machine uses a global `core.hooksPath` dispatcher
(under which a bare `pre-commit install` silently refuses to wire anything),
every fresh clone must run the repo-native installer once:

```bash
make hooks          # or: scripts/install-hooks.sh
```

This writes managed `pre-commit` and `pre-push` hooks into the clone-local hook
path the dispatcher delegates to, proves git actually dispatches to them, then
runs `pre-commit run --all-files`. It never touches global git config. Re-run it
after changing `.pre-commit-config.yaml`.

- **pre-commit**: general file checks, ruff lint (`--fix`) + format, gitleaks.
- **pre-push**: the test suite (`uv run pytest`).

It is normal for the formatters to make changes on the first run — stage them
and commit.

## Continuous integration

`.github/workflows/ci.yml` runs on every pull request and on every push to `dev`
and `main`. It is the full gate, and the same for every change: no job is
skipped by path, by change scope, or by the size of a change.

| Job | Verifies | Starts | Reproduce locally |
| --- | --- | --- | --- |
| `audit` | No known vulnerability in the locked runtime dependencies (`pip-audit --strict`) | At once | `uv export --frozen --no-dev --all-extras --no-emit-project --no-hashes --format requirements-txt -o audit-requirements.txt`, then `uvx pip-audit --strict --no-deps --requirement audit-requirements.txt` |
| `lint` | `ruff check` and `ruff format --check` | At once | `uv run ruff check .` and `uv run ruff format --check .` |
| `test` | The whole pytest suite with coverage; uploads `coverage.xml` | At once | `uv run pytest -n auto` (as CI) or `uv run pytest` (serial) |
| `sonar` | SonarCloud analysis and quality gate over the sources and `coverage.xml` (Dependabot pull requests skip the scan step: the token is not available to them) | When `test` passes | CI only (needs `SONAR_TOKEN`) |

The PR title guard (`pr-title.yml`) runs separately on pull requests into `dev`.

**Fast feedback.** `audit`, `lint` and `test` start together, and `sonar` waits
only for the coverage report it analyses. A lint or format failure no longer
stops the tests or the analysis from running.

**Test ownership.** There are no shards. The `test` job owns the whole suite:
pytest-xdist schedules every collected test exactly once across the runner's
CPUs, and pytest-cov merges the workers' coverage into the single
`coverage.xml`, the same input a serial run produces. To debug a failure, rerun
just that test serially, for example
`uv run pytest tests/test_auth.py::test_login_view_authenticates`.

## Branching, releases, and versioning

- Feature branches open PRs into `dev`; `dev` is promoted to `main` by a PR.
- PR titles into `dev` follow Conventional Commits (`feat:`, `fix:`, `docs:`,
  `chore:`, …) and are enforced by `tools/check_pr_title.py`. The squash-merge
  title becomes the commit release-please reads.
- **release-please owns versioning and `CHANGELOG.md`.** Do not hand-edit
  `[project].version` in `pyproject.toml` or `CHANGELOG.md`. On merge to `main`,
  release-please maintains a `chore(main): release X.Y.Z` PR; merging it tags the
  release, publishes to PyPI (OIDC trusted publishing), and a back-merge PR
  syncs `main → dev`.
- Version mapping: `feat:` → minor, `fix:`/`perf:` → patch, `feat!:` or a
  `BREAKING CHANGE:` footer → major (demoted to minor pre-1.0). `docs`, `chore`,
  `refactor`, `test`, `ci`, `build` do not cut a release.

Publishing requires a one-time owner setup: a PyPI project with a Trusted
Publisher for this repo's `release-please.yml` and a GitHub Environment named
`pypi`.

## Ground Control

This repository is onboarded to Ground Control for requirements and workflow
automation. Its project, workflow commands, SonarCloud settings, and plan rules
live in `.ground-control.yaml` and `.gc/plan-rules.md`. See `AGENTS.md`.
