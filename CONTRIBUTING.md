# Contributing to KnightEval

Thanks for looking! KnightEval is pre-alpha and built in public — early
contributors genuinely shape the tool. This guide gets you productive fast.

## Ways to help right now

- **Try the design.** Read [DESIGN.md](DESIGN.md) and [ROADMAP.md](ROADMAP.md) and
  open an issue if a contract, metric, or default feels wrong. Design feedback
  before code lands is the highest-leverage contribution today.
- **Grab a [`good first issue`](https://github.com/GabbarX/knighteval/labels/good%20first%20issue).**
- **Report a bug** or **request a metric/feature** using the issue templates.
- **Improve docs** — README, examples, error messages.

Before starting non-trivial work, open (or comment on) an issue so we agree on
the approach. It saves everyone a rejected PR.

## Project shape

```
knighteval/         # the package (engine/evaluators/scoring have zero dependency
                    #   on cli/reporting — keep that boundary)
  cli, config, datasets, endpoints, evaluators/{deterministic,llm},
  judges, scoring, thresholds, reports/{terminal,html}, storage, caching,
  models, engine.py, utils
tests/  { unit, integration, fixtures }
examples/           # runnable RAG endpoint + config + dataset
docs/
```

The layering rule matters: `engine`, `evaluators`, and `scoring` must not import
from `cli` or `reports`, so a Python SDK or other frontend can be added later.

## Dev setup

Requires Python 3.12+. We use [uv](https://github.com/astral-sh/uv) (a
`.python-version` pins 3.12).

```bash
git clone https://github.com/GabbarX/knighteval
cd knighteval
uv venv && uv sync           # or: python -m venv .venv && pip install -e ".[dev]"
```

## Quality gate (run before you push)

```bash
ruff check . && ruff format --check .   # lint + format
mypy knighteval                          # types
pytest                                   # tests
```

Every PR must pass all three. CI runs the same checks.

## Testing philosophy (TDD)

We write tests first. Schema/scoring/threshold logic and adapters get **unit
tests**; the engine gets an **integration test** against a local mock RAG HTTP
server fixture (no live API calls in CI — judges are mocked). A change to a
[stable contract](DESIGN.md#15) needs a test that pins the contract.

Non-negotiable: **no secrets in saved output.** There is a test that asserts no
API key appears in `run.json`; keep it green.

## Commit & PR conventions

- Branch off `master`; keep PRs focused and small.
- Use [Conventional Commits](https://www.conventionalcommits.org): `feat:`,
  `fix:`, `docs:`, `test:`, `refactor:`, `chore:`.
- Fill out the PR template: what, why, how tested. Link the issue.
- Note any change to a stable contract explicitly — those need extra scrutiny.

## Scope

KnightEval is deliberately narrow (see [ROADMAP.md](ROADMAP.md#how-we-prioritize)).
SaaS/accounts, tracing/monitoring, prompt management, synthetic data, human
review, framework-native adapters, and streaming are **out of scope** for the MVP.
A clear "out of scope" is a respectful, first-class answer — please don't take it
personally.

## Licensing of contributions

By submitting a contribution you agree it is licensed under the project's
[Apache 2.0](LICENSE) license.

## Code of Conduct

Participation is governed by the [Code of Conduct](CODE_OF_CONDUCT.md).
