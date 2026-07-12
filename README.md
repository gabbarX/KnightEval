# KnightEval

**pytest for RAG.** Point it at any HTTP RAG endpoint, give it a JSONL dataset,
get a scorecard — in the terminal, as a self-contained HTML report, as JSON, and
as a CI-friendly exit code.

> **Minimal-config RAG evaluation — one config file, one command.**

[![License: Apache 2.0](https://img.shields.io/badge/License-Apache_2.0-blue.svg)](LICENSE)
[![Python](https://img.shields.io/badge/python-3.12+-blue.svg)](https://www.python.org)
[![Status: pre-alpha](https://img.shields.io/badge/status-pre--alpha-orange.svg)](ROADMAP.md)
[![PRs welcome](https://img.shields.io/badge/PRs-welcome-brightgreen.svg)](CONTRIBUTING.md)

> 🚧 **Pre-alpha — built in public.** The design is locked ([DESIGN.md](DESIGN.md));
> the code is landing phase by phase (see the [roadmap](ROADMAP.md)). Nothing is
> on PyPI yet. **Star the repo to follow the first release**, or jump into
> [good first issues](https://github.com/GabbarX/knighteval/labels/good%20first%20issue).

---

## Why

You built a RAG endpoint. Is it any good? Did that prompt change make retrieval
worse? Existing tools — Ragas, DeepEval, TruLens, LangSmith — are powerful, but
they ask you to learn pipelines, tracing, and experiment abstractions *before*
you get a single number.

KnightEval trades depth for a fast, **honest** first result:

- **Works against a black-box HTTP endpoint.** No SDK, no instrumentation, no
  code changes to your app. If it answers HTTP, KnightEval can grade it.
- **Honest about what needs a judge.** Latency, reliability, exact match, regex,
  keyword and schema checks run **deterministically with zero config**. The
  LLM-graded metrics (faithfulness, hallucination, context precision/recall,
  answer relevance/correctness) need a judge you configure — and KnightEval
  tells you loudly *why* a metric was skipped instead of silently scoring zero.
- **Made for CI.** One command, a stable exit code, JUnit XML. Add a RAG quality
  gate to your pipeline without writing Python.
- **Local-first.** No account, no SaaS, no telemetry. Your data leaves your
  machine only when *you* configure an external judge.

## The idea (target experience)

```bash
pip install knighteval

# interactive, endpoint-aware: probes your API, proposes field paths, picks a judge
knighteval init

# run the evaluation
knighteval run --config knighteval.yaml
```

```text
KnightEval — 100 cases against http://localhost:8000/query

  Metric                Score    Status
  ─────────────────────────────────────
  Faithfulness          91.4%    PASS
  Context Precision     86.2%    PASS
  Context Recall        79.9%    WARN
  Answer Relevance      88.1%    PASS
  Hallucination Rate     3.2%    PASS
  Latency (p95)         820ms    PASS
  Endpoint Reliability  100.0%   PASS
  ─────────────────────────────────────
  Overall               87.6%    PASS      exit 0

  HTML report → runs/2026-07-12T18-30-00/report.html
```

Deterministic-only (no judge, nothing leaves your machine):

```bash
knighteval run --config knighteval.yaml   # latency, reliability, exact_match, regex, keyword...
```

## What you write (the one config file)

```yaml
# knighteval.yaml
endpoint:
  url: http://localhost:8000/query
  answer_path: data.answer          # dot-path or JSONPath-subset into the response
  contexts_path: data.contexts[*].text
dataset: tests/rag_cases.jsonl
judge:
  provider: openai                  # or openai_compatible, or omit for deterministic-only
  model: gpt-4o-mini
  # api key comes from the environment, never this file
thresholds:
  faithfulness: { warn: 0.8, fail: 0.6 }
  context_recall: { warn: 0.8, fail: 0.6 }
```

Dataset is JSONL, one case per line:

```jsonl
{"id": "q1", "question": "What is our refund window?", "expected_answer": "30 days", "expected_context": ["Refunds are accepted within 30 days."]}
```

## In CI

```bash
knighteval run --config knighteval.yaml --ci --junit results.xml
# exit 0 = pass · 1 = threshold failure · 2 = usage/config error · 3 = endpoint failure · 4 = judge integrity failure
```

The `--ci` flag never prompts and never asks for cost confirmation.

## How it compares

| | KnightEval | Ragas / DeepEval | LangSmith / TruLens |
|---|---|---|---|
| Evaluates a black-box HTTP endpoint | ✅ | ⚠️ needs wiring | ⚠️ needs tracing |
| Zero-config deterministic metrics | ✅ | ❌ | ❌ |
| Self-contained HTML report (no server) | ✅ | ❌ | ❌ (hosted) |
| CI exit codes + JUnit out of the box | ✅ | ⚠️ | ⚠️ |
| Local-first, no account | ✅ | ✅ | ❌ |
| Deep tracing / experiment tracking | ❌ (non-goal) | ⚠️ | ✅ |

KnightEval is not trying to be a platform. If you need tracing, experiment
tracking, or a hosted dashboard, reach for the tools above. If you want a
number and an HTML report *today*, start here.

## Metrics

**Deterministic (zero-config, no network):** exact match, token overlap, keyword
presence, regex assertion, JSON schema, citation presence, max answer length,
latency limit, endpoint reliability.

**LLM-judged (needs a configured judge):** faithfulness (+ hallucination rate,
one shared judge pass), answer relevance, answer correctness (reference-gated),
context precision, context recall.

## Status & roadmap

Pre-alpha. Build sequence and release milestones are in **[ROADMAP.md](ROADMAP.md)**;
the full design rationale (schemas, scoring rules, stable contracts) is in
**[DESIGN.md](DESIGN.md)**.

## Contributing

Early contributors shape the tool. Start with **[CONTRIBUTING.md](CONTRIBUTING.md)**
and the [`good first issue`](https://github.com/GabbarX/knighteval/labels/good%20first%20issue)
label. Be excellent to each other — [Code of Conduct](CODE_OF_CONDUCT.md).

## License

[Apache 2.0](LICENSE).
