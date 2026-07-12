# KnightEval Roadmap

*"pytest for RAG" — evaluate any HTTP RAG endpoint against a JSONL dataset. One
config, one command. Local-first, no SaaS.*

This is the public roadmap. The engineering rationale and stable contracts live
in [DESIGN.md](DESIGN.md); the internal build sequence is in
[plans/def.md](plans/def.md). No calendar dates — this is a solo/early-maintainer
project, so releases ship when they're ready. **Star the repo to follow along.**

## Milestone map

| Build phase | Release | Track |
|---|---|---|
| P0 | **v0.1 (alpha)** | Deterministic core |
| P1 | **v0.2 (beta)** — *public launch* | + LLM judges |
| P2 | **v0.3** | Reports + onboarding |
| P3 | **v0.4** | History + regression |
| P4 | **v0.5 → v1.0** | Provider breadth + contract freeze |

**Launch decision:** ship a real **v0.1 on P0** (deterministic, zero-network) to
de-risk the CLI/UX plumbing and gather early feedback — but the *loud* public
launch waits for **v0.2 (P1)**, when faithfulness + hallucination land. "RAG
eval" in the market means LLM-graded quality; launching deterministic-only invites
"that's not real RAG eval" and burns the one-shot attention spike. Deterministic
gates you can trust with **zero network** *plus* judge metrics when you want them
is the lovable, defensible story.

## Releases

### v0.1 — alpha (P0) · size L
- **Headline:** Deterministic RAG scoring with zero config and zero network.
- **Proof demo:** `pip install knighteval && knighteval run` → Rich scorecard +
  a non-zero exit code that fails CI on a regression, no API key required.
- **For:** backend/platform engineers who want a RAG endpoint smoke-test in CI today.

### v0.2 — beta / public launch (P1) · size L
- **Headline:** LLM-judge metrics — faithfulness, hallucination, relevance,
  correctness — with a cost preflight.
- **Proof demo:** the same one command now flags a hallucinated answer with a
  faithfulness score and an estimated cost you confirm before spend.
- **For:** RAG/AI engineers who need quality signal, not just plumbing checks.

### v0.3 (P2) · size M
- **Headline:** Shareable self-contained HTML report + guided `init` onboarding.
- **Proof demo:** `knighteval init` interrogates your endpoint and writes the
  config; a run produces an HTML report you can send to a PM (no server).
- **For:** team leads showing results to non-CLI stakeholders.

### v0.4 (P3) · size M
- **Headline:** History, baselines, and regression detection.
- **Proof demo:** `knighteval run --compare-baseline` fails the build because
  faithfulness dropped 8% vs the last release.
- **For:** teams running RAG eval on every PR who must catch quality regressions.

### v0.5 (P4) · size M
- **Headline:** Judge provider breadth — Anthropic, Gemini, Ollama — plus
  concurrency/rate-limit hardening.
- **Proof demo:** swap OpenAI for a local Ollama judge by changing one config
  key; run fully offline.
- **For:** privacy/cost-sensitive teams that can't send data to hosted judges.

### v1.0 — stable · size S
- **Headline:** frozen public contracts; safe to build CI pipelines on.
- **Proof demo:** a v0.2-era config + dataset runs unchanged on v1.0.

## What v1.0 means

v1.0 is a **compatibility promise, not a feature milestone.** It ships when these
are stable and change only under a SemVer major bump (see DESIGN.md §15):

- **Dataset fields** — JSONL/CSV/JSON schema + field names.
- **Config keys** — the YAML schema (endpoint mapping, evaluator config, thresholds).
- **Metric names** — canonical evaluator IDs (`exact_match`, `faithfulness`, …).
- **Result schema** — the JSON export shape (per-case + aggregate) + status enum.
- **Exit codes** — `0` pass · `1` threshold fail · `2` usage/config · `3` endpoint · `4` judge integrity.

Requires: all contracts documented + versioned, a published deprecation policy,
and one full minor cycle (v0.2→v0.5) with no breaking change to any of them. No
new capability beyond P4 is required — 1.0 is stabilization.

## How we prioritize

Solo-maintained, local-first, deliberately narrow.

**Yes if it:** keeps "one config, one command" intact · works local-first with no
account/service/telemetry · serves the core loop (dataset → HTTP endpoint →
metrics → scorecard/report → exit code) · has a stable, testable contract.

**No / deferred if it:** needs a backend, SaaS, or accounts · belongs to
observability/tracing, prompt management, synthetic-data, or human-review (adjacent
products) · adds a framework-native adapter (we speak HTTP — keep the surface
small) · adds streaming/real-time concerns · grows config surface faster than user
value.

A clear "out of scope" is a first-class, respectful answer — it protects the
maintainer and the product's focus.

## Adoption risks & mitigations

1. *"Deterministic-only isn't real RAG eval"* → lead launch messaging with v0.2
   judge metrics; keep v0.1 low-key.
2. *Judge API cost/spend fear* → cost preflight + explicit confirm + result
   caching, on by default (P1).
3. *Endpoint mapping friction (every RAG API differs)* → interactive endpoint-aware
   `init` + JSONPath mapping + actionable errors (P2).
4. *Judge non-determinism erodes CI trust* → caching, regression tolerances,
   baseline compare so scores are stable and diffs explainable (P1/P3).
5. *Crowded eval market* → own a sharp niche: HTTP-endpoint-first, CI-stable exit
   codes, zero-config deterministic mode, no SaaS.
