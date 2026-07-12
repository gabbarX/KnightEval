# KnightEval — Pre-Implementation Design

Status: design draft for review. No production code should be written until the
decisions marked **[STABLE]** are agreed, because they leak into datasets, config
files, saved runs, and cache keys that users depend on across versions.

---

## 0. Assumptions challenged (read this first)

The product definition asks to identify anything that makes the "zero-config"
claim misleading and to challenge unclear assumptions. That is the most important
section, so it comes first.

### 0.1 "Zero-config" is misleading as written

The headline promise and the intro example conflict with the metric set.

The intro shows this producing a full scorecard:

```bash
knighteval run --endpoint http://localhost:8000/query --dataset tests/rag_cases.jsonl
```

That command, with no config file and no judge, **cannot** produce the example
terminal scorecard (Faithfulness 91.4%, Context Precision 86.2%, etc.). Reasons:

1. **Every headline quality metric except Exact Match needs an LLM judge.**
   Faithfulness, Context Precision, Context Recall, Answer Relevance,
   Hallucination Rate, Answer Correctness are all LLM-based. No judge configured →
   6 of 6 quality rows are `SKIPPED` or `ERROR`. What actually runs with zero
   config is Latency + Endpoint Reliability + (if `expected_answer` present)
   Exact Match. That is not the advertised experience.

2. **A judge needs a provider, a model, an API key, and implies cost.** That is
   configuration by definition, and it also means data leaves the machine. The
   "local-first / nothing leaves your machine" principle and the "full scorecard
   in one command" promise cannot both be true on first run.

3. **Response field mapping is almost never inferable for free.** Real RAG APIs
   return `answer` under arbitrary paths (`result`, `data.output`,
   `choices[0].message.content`). Without `answer_path`/`contexts_path`, the
   answer extraction fails or the contexts are missing.

4. **Context metrics need retrieved contexts, which many endpoints do not
   return.** If the endpoint returns only an answer string, Context Precision /
   Context Recall / Faithfulness have no evidence to score against. 3–4 of the 6
   headline metrics silently die.

5. **Reference metrics need reference data.** Answer Correctness needs
   `expected_answer`; Context Recall wants `expected_context`. A dataset of bare
   `question`s yields none of these.

**Recommended reframing (decision needed):**

- Positioning: change "Zero-config RAG evaluation" → **"Minimal-config RAG
  evaluation. One config file, one command."** Keep "zero-config" only for the
  *deterministic* subset (latency, reliability, exact match, regex, keyword),
  which genuinely needs no judge.
- Make `knighteval init` **interactive and endpoint-aware**: it should run
  `inspect` against the endpoint, propose `answer_path`/`contexts_path`, ask which
  judge provider to use (or "none — deterministic only"), and write a config that
  actually works. This is what makes the first run honest.
- The tool must **loudly report the difference** between "metric skipped because
  no judge configured" and "metric skipped because endpoint returned no contexts."
  Silent skips are the fastest way to make the product feel like it lies.

### 0.2 Other assumptions to fix

- **"Deterministic exit codes" + LLM judges is a contradiction.** LLMs are not
  deterministic even at temperature 0. Exit codes are deterministic *given a set
  of scores*; the scores themselves are not reproducible run-to-run. State this
  honestly. Caching (§Caching) is what buys practical reproducibility, not
  determinism. Do not claim byte-stable scores.

- **Hallucination is double-counted.** The spec says hallucination is an
  independent risk metric *and* influences faithfulness. Pick one primary source.
  Recommendation: hallucination detection and faithfulness are **the same LLM
  pass** (claim extraction → support check). Faithfulness = supported/total
  claims. Hallucination rate = derived aggregate (cases with ≥1 unsupported
  claim). One judge call, two views. Do not run two separate judges that can
  disagree.

- **Context Recall from `expected_answer` conflates recall with correctness.**
  Using the gold answer as the recall target measures "do contexts contain the
  answer," which overlaps Answer Correctness. Prefer `expected_context` for
  recall; only fall back to `expected_answer` with an explicit lower-confidence
  flag in the result. Define this precisely, don't leave it "when available."

- **Overall score with missing metrics needs a hard rule, not a vibe.** See
  §Scoring. Must define: minimum required metrics, weight renormalization, and
  the "too many failed → no overall score" threshold numerically.

- **Cost can surprise users.** 100 cases × 6 LLM metrics = 600+ judge calls per
  run. Non-CI runs must show an **estimated cost preflight** and (for large runs)
  require confirmation. CI must never prompt.

- **CSV cannot cleanly hold `expected_context` (a list).** Keep JSONL primary;
  CSV support is best-effort with a documented convention (e.g. `||`-joined), and
  flagged as lossy.

---

## 1. Product Requirements Document (concise)

**What:** A pip-installable CLI that evaluates any HTTP RAG endpoint against a
JSONL dataset and emits a terminal scorecard, a self-contained HTML report, JSON
results, and CI-friendly exit codes.

**Why:** Existing frameworks (Ragas, DeepEval, TruLens, LangSmith) require
learning pipelines/tracing/experiment abstractions before getting a number.
KnightEval trades depth for a fast, honest first result.

**Who:** RAG/ML engineers, backend engineers adding retrieval, LLM-app
prototypers, teams wanting RAG gates in CI, OSS maintainers, early startups.

**MVP success:** a developer who knows their endpoint gets a useful HTML report
from **one config file + one command**, understands whether failures are
retrieval- or generation-side, and can add it to CI — without writing Python.

**Core principles (unchanged):** endpoint-agnostic, minimal-config,
transparent scoring (score + reason + evidence + evaluator + error, always),
CI-friendly, extensible, local-first.

**Explicit non-goals (MVP):** SaaS, accounts, tracing, monitoring, prompt mgmt,
synthetic data gen, human review, framework-native adapters, streaming.

---

## 2. MVP user journeys

1. **First evaluation (happy path).** `pip install` → `init` (interactive, probes
   endpoint, picks judge) → `run` → read terminal scorecard → open HTML → find
   worst cases. Time-to-first-report target: < 5 min.

2. **Deterministic-only / offline.** No judge. `init` chooses "deterministic
   only." `run` yields latency, reliability, exact-match, keyword/regex. Zero
   external calls. This is the true zero-config path — market it as such.

3. **CI gate.** `run --ci` in GitHub Actions with `OPENAI_API_KEY` secret. No
   prompts, thresholds enforced, exit code drives the job, optional
   `--junit`.

4. **Debug a failing endpoint.** `validate` (config + dataset + reachability +
   mapping + credentials) → `inspect` (sample request, show response tree,
   suggest paths) → fix config → re-run.

5. **Regression check.** `baseline set latest` once → `run --compare-baseline`
   thereafter → report shows deltas, new failures, new hallucinations.

6. **Cost/scope control.** `run --limit 10 --tags critical` for a cheap smoke
   run before the full sweep.

---

## 3. Final CLI design

```
knighteval init            # interactive scaffold; probes endpoint, picks judge
knighteval validate        # config + dataset + endpoint + mapping + creds
knighteval inspect --endpoint URL [--question TEXT]   # infer answer/context paths
knighteval run [flags]     # the main event
knighteval compare A B     # e.g. latest baseline
knighteval history         # list local runs
knighteval report [RUN|latest]   # (re)open HTML for a stored run
knighteval baseline set RUN|latest
knighteval cache clear [--all|--metric M]
```

`run` flags (stable surface):

```
--config PATH              (default ./knighteval.yaml)
--endpoint URL             (override config)
--dataset PATH             (override config)
--limit N
--tags a,b,c
--output {terminal,json,both}     (default both)
--no-html
--junit PATH
--ci                        (no prompts, no color unless forced, quiet-ish)
--compare-baseline
--verbose / -v              (repeatable)
--quiet / -q
--no-cache
```

Rules: `--ci` disables all interactive prompts and cost confirmations (assumes
consent via config/env). `run` with neither config nor endpoint → error exit 2
with an actionable message. Precedence: flags > env > config > defaults.

---

## 4. Configuration schema  **[STABLE — keys are a compatibility contract]**

Pydantic-validated YAML. Additive changes only after v1; renames need aliases.

```yaml
project:
  name: string                      # required

endpoint:
  url: string                       # required (or via --endpoint)
  method: POST                      # default POST
  timeout_s: 30                     # default
  headers: {Content-Type: application/json}
  sensitive_headers: [Authorization]   # redacted in saved runs/reports
  request:
    body: { query: "{{ question }}" }  # Jinja-ish templating over case fields
  response:
    answer_path: answer             # dot-path or JSONPath ($.a.b[0])
    contexts_path: contexts         # optional; missing → context metrics SKIPPED
    latency_path: metadata.latency_ms  # optional; else measured wall-clock

evaluation:
  mode: hybrid                      # reference-based | reference-free | hybrid
  judge:
    provider: openai                # openai | anthropic | gemini | openai_compatible | ollama
    model: gpt-4.1-mini
    base_url: null                  # for openai_compatible / ollama
    temperature: 0
    timeout_s: 60
    max_retries: 3
    concurrency: 8
  metrics: [faithfulness, context_precision, context_recall,
            answer_relevance, hallucination_rate, answer_correctness,
            exact_match, latency]

thresholds:
  overall_score: 0.80               # scalar → min
  faithfulness: 0.85
  hallucination_rate: { max: 0.05 } # object form for max-style gates
  # per-metric: scalar = min; {max: x} = ceiling; {warn: x, fail: y} = bands

regression:                          # optional
  overall_score: { max_drop: 0.03 }
  latency_p95:   { max_increase_pct: 20 }

output:
  directory: .knighteval
  html: true
  json: true
  redact_fields: []                 # dataset/response fields stripped from reports
```

Notes:
- **Path syntax:** support dot-path (`a.b.0`) and a JSONPath subset (`$.a.b[0]`).
  Document exactly which JSONPath features are supported; reject the rest with a
  clear error rather than half-supporting.
- Secrets (API keys) come only from env vars, never config. Config referencing
  `${OPENAI_API_KEY}` is expanded at load, never persisted.

---

## 5. Dataset schema  **[STABLE]**

Primary: JSONL, one case per line.

```json
{
  "id": "refund-policy-001",
  "question": "Can a customer receive a refund after 30 days?",
  "expected_answer": "Refunds are only available within 30 days of purchase.",
  "expected_context": ["Customers may request a refund within 30 calendar days..."],
  "tags": ["compliance"],
  "metadata": { "category": "refund-policy", "difficulty": "easy" }
}
```

- Required: `question`.
- Optional: `id` (auto-assigned stable index+hash if absent), `expected_answer`,
  `expected_context` (list of strings), `tags` (list), `metadata` (object).
- CSV/JSON accepted; JSONL recommended. `expected_context` in CSV via a documented
  delimiter, flagged lossy.
- **Dataset checksum** (sha256 of normalized rows) recorded per run for
  reproducibility/regression pairing.

---

## 6. Endpoint adapter contract  **[STABLE interface]**

```python
class EndpointResponse(BaseModel):
    answer: str | None
    contexts: list[str] | None
    latency_ms: float
    status_code: int | None
    raw: dict | None          # full body, for inspect/debug
    error: EndpointError | None   # transport/mapping failure, NOT a quality signal

class EndpointAdapter(Protocol):
    async def query(self, case: Case) -> EndpointResponse: ...
```

- MVP ships one adapter: `GenericHTTPAdapter` (HTTPX async, bounded exponential
  backoff via Tenacity, per-run concurrency cap, per-request timeout).
- Mapping failures (missing `answer_path`, malformed JSON, HTTP 5xx after retries)
  produce `EndpointResponse.error`, which the engine records as a **system
  failure** — never as score 0. See §Failure handling.
- `raw` is what powers `inspect` and the "Received top-level fields: ..." error.

---

## 7. Evaluator interface  **[STABLE interface]**

Both deterministic and LLM evaluators implement the same protocol and return the
same result schema (§8). This uniformity is the extensibility hinge.

```python
class EvalContext(BaseModel):
    case: Case
    response: EndpointResponse
    judge: JudgeProvider | None

class Evaluator(Protocol):
    metric: str
    version: str            # bump → cache invalidation
    requires: Requirements  # needs_contexts / needs_expected_answer /
                            # needs_expected_context / needs_judge
    async def evaluate(self, ctx: EvalContext) -> EvalResult: ...
```

- The engine consults `requires` **before** running. If prerequisites are missing
  (no contexts, no judge, no reference), the evaluator is `SKIPPED` with a reason
  string — it is never invoked and never fabricates a 0.
- Deterministic MVP evaluators: exact_match, token_overlap, keyword_presence,
  regex_assertion, json_schema, citation_presence, max_answer_length,
  latency_limit, http_reliability.
- LLM MVP evaluators: faithfulness (+hallucination, same pass), answer_relevance,
  answer_correctness, context_relevance (→ precision/recall).

---

## 8. Normalized result schemas  **[STABLE]**

Per-evaluator result:

```json
{
  "metric": "faithfulness",
  "score": 0.91,                 // null when SKIPPED/ERROR
  "status": "PASS",              // PASS|WARN|FAIL|SKIPPED|ERROR
  "reason": "All material claims supported by retrieved context.",
  "evidence": { "supported_claims": ["..."], "unsupported_claims": [] },
  "evaluator": { "type": "llm_judge", "provider": "openai",
                 "model": "gpt-4.1-mini", "version": "1",
                 "prompt_version": "1" },
  "error": null,                 // {type, message} when status=ERROR
  "cost": { "input_tokens": 812, "output_tokens": 96, "usd": 0.0004 }
}
```

Per-case result: `{case, response(redacted), results: [EvalResult], case_score}`.

Run result (`run.json`): version, config snapshot (secrets stripped), dataset
checksum, evaluator versions, judge provider/model, start/end times, aggregate
metrics, per-case results, system errors, total cost/tokens.

**All LLM judge outputs are validated against strict structured-output schemas**
(provider structured-output / tool-calling where available; otherwise strict JSON
parse + Pydantic + retry). A judge that returns unparseable output after retries →
`status: ERROR`, not a guessed score.

---

## 9. Scoring and threshold rules

**Per-metric normalization:** all quality metrics ∈ [0,1]. Latency in seconds
(natural units). Hallucination rate ∈ [0,1] where lower is better.

**Status bands (defaults, configurable):** from thresholds. Scalar threshold =
min for PASS; a band `{warn, fail}` defines WARN/FAIL zones; `{max}` for
lower-is-better metrics (hallucination, latency).

**Overall score** = weighted mean over the *available, non-errored* contributing
metrics, weights **renormalized** across whatever is present:

```
default weights:
  faithfulness 0.30, answer_relevance 0.25, context_precision 0.15,
  context_recall 0.15, answer_correctness 0.15   (correctness only if references)
```

- exact_match, latency, hallucination_rate, reliability are **not** in the overall
  score (exact_match too brittle for open-ended; latency/reliability are separate
  axes; hallucination is a derived risk view of faithfulness).
- **No-overall-score rule [STABLE]:** if fewer than **2** contributing metrics are
  available, or if >50% of attempted contributing evaluators ERROR (system
  failures, not low scores), overall score = `N/A` and the run reports an
  evaluation-integrity failure (exit 4), not a quality pass/fail.
- The report always lists which metrics contributed and their renormalized
  weights.

**Threshold gate → exit code** (see §10). Quality FAIL is distinct from system
ERROR.

---

## 10. Exit codes  **[STABLE]**

```
0  completed, all quality gates passed
1  one or more quality thresholds failed (scores valid, just below bar)
2  invalid configuration or dataset (nothing ran)
3  endpoint execution failure (too many cases failed to get a response)
4  evaluator/judge failure (integrity: overall score not computable / judge down)
```

Precedence when multiple apply: 2 > 3 > 4 > 1 > 0 (config errors first, quality
last). Define numerically what "too many" means for code 3 (e.g. > 50% of cases
have `response.error`, configurable).

---

## 11. Repository architecture

```
knighteval/
  cli/            # Typer commands ONLY — thin; parse, call engine, render
  config/         # Pydantic models, loader, env expansion, validation
  datasets/       # JSONL/CSV/JSON loaders, checksum, schema validation
  endpoints/      # EndpointAdapter protocol + GenericHTTPAdapter
  evaluators/
    deterministic/
    llm/
  judges/         # JudgeProvider protocol + openai/anthropic/gemini/compat/ollama
  scoring/        # aggregation, weight renormalization, status bands
  thresholds/     # gate evaluation, exit-code mapping
  reports/
    terminal/     # Rich renderer
    html/         # Jinja2 + embedded assets
  storage/        # RunStore (local JSON layout), history, baselines
  caching/        # judge-result cache (key composition)
  models/         # shared Pydantic: Case, EvalResult, RunResult, ...
  engine.py       # orchestrates: load → query → evaluate → aggregate → gate
  utils/
tests/ { unit, integration, fixtures }
examples/  docs/
```

Hard rule: **the engine + evaluators + scoring have zero dependency on CLI or
reporting.** CLI and reporters consume `RunResult`. This is what lets the roadmap
add a Python SDK / other frontends later.

---

## 12. HTML report information architecture

Single self-contained file (inlined CSS/JS, embedded chart lib or hand-rolled
SVG — no server, no CDN). Sections:

1. **Header:** project, dataset, case count, endpoint, run time, KnightEval
   version, judge model + evaluator versions, overall score + status.
2. **Metric scorecards:** each with score, status, contribution weight, reason.
3. **Distributions:** per-metric score histogram; latency (avg/median/p95);
   hallucination rate; endpoint failures (count + reasons).
4. **Case table:** filterable by status / metric / tag / metadata; columns for
   question, generated answer, worst metric.
5. **Case detail (expand):** question, expected answer, generated answer,
   retrieved contexts, expected contexts, per-metric reason + evidence
   (supported/unsupported claims, irrelevant chunks), request error + evaluation
   errors.
6. **Config snapshot** (secrets redacted) + reproducibility metadata.
7. **Regression panel** (when `--compare-baseline`): deltas, new failures, new
   hallucinations, resolved, latency/reliability change.

Must answer at a glance: which questions failed, retrieval vs generation, which
claims unsupported, which chunks irrelevant, did it regress, did the endpoint get
slower/less reliable.

---

## 13. Phased implementation plan

- **P0 — Skeleton (walking, honest):** package + Typer CLI, config load/validate,
  JSONL loader, GenericHTTPAdapter, response mapping, **deterministic evaluators
  only** (exact_match, latency, reliability, keyword/regex), terminal scorecard,
  JSON output, exit codes, local run storage. This is the true zero-config,
  no-network product. Ship it first — it works with no judge.
- **P1 — Judges + core LLM metrics:** JudgeProvider (OpenAI + openai_compatible),
  structured-output validation, faithfulness(+hallucination), answer_relevance,
  answer_correctness, context precision/recall, judge-result caching, cost
  preflight. Now the full scorecard exists.
- **P2 — Reports + onboarding:** HTML report, `inspect`, interactive `init`,
  `--junit`, `--ci` polish, verbose/quiet.
- **P3 — History/regression:** `history`, `baseline set`, `compare`,
  `--compare-baseline`, regression tolerances, report regression panel.
- **P4 — Provider breadth:** Anthropic, Gemini, Ollama; concurrency/rate-limit
  hardening.

Each phase is releasable. Reliability of P0–P1 gates all later work.

---

## 14. Key technical risks and mitigations

| Risk | Mitigation |
|---|---|
| Judge cost/latency blowup on large sets | Concurrency cap; caching; cost preflight + confirm (non-CI); `--limit` |
| Judge non-determinism undermines "stable" claims | Cache keyed on full input; temp 0; document that scores aren't byte-stable; gate on bands not exact values |
| Structured-output drift/invalid JSON | Provider structured-output/tool-calls; strict Pydantic; bounded retries; ERROR not fabricated score |
| Silent metric skips read as lies | Explicit SKIPPED status + reason ("no contexts" vs "no judge"); surfaced in terminal + HTML |
| Response mapping mismatch | `inspect` + actionable "received fields" errors + `validate` |
| System failure counted as quality 0 | Separate `EndpointResponse.error` path; §Failure handling; distinct exit codes |
| Rate limits / flaky endpoints | Tenacity bounded exponential backoff; per-request timeout; partial-result retention |
| Secret leakage into saved runs | Env-only secrets; redact sensitive_headers; config snapshot scrub; test asserts no key in run.json |
| Sending private context to third-party judge | One-time per-run warning + config consent flag; Ollama/local path; documented data-egress list |
| Cache staleness after prompt/logic change | Cache key includes evaluator version + prompt_version; `cache clear` |

---

## 15. Stable decisions (backward-compatibility contract)

Changing any of these after v1 requires a migration/alias and a version note:

1. **Dataset field names** (`question`, `expected_answer`, `expected_context`,
   `id`, `tags`, `metadata`) and JSONL-line-per-case format.
2. **Config keys** (§4) — additive only; renames need aliases.
3. **Metric names** (`faithfulness`, `context_precision`, `context_recall`,
   `answer_relevance`, `hallucination_rate`, `answer_correctness`, `exact_match`,
   `latency`, `endpoint_reliability`).
4. **EvalResult / RunResult schema** and status enum values
   (`PASS|WARN|FAIL|SKIPPED|ERROR`).
5. **Exit codes** (§10) and their precedence.
6. **Cache-key composition** (metric, question, answer, contexts, expected_*,
   provider, model, evaluator version, prompt version).
7. **Run directory layout** (`.knighteval/runs/<ISO8601Z>/{run.json, cases.jsonl,
   report.html, config.snapshot.yaml}`, `baselines/`, `cache/`).
8. **Overall-score composition + renormalization rule + no-score threshold**
   (§9) — because CI gates depend on the number's meaning.
9. **Dataset checksum algorithm** (sha256 of normalized rows) — regression pairing
   depends on it.

---

## Open questions for the user

1. Accept the reposition to **"minimal-config"** (and reserve "zero-config" for the
   deterministic-only path)? This is the biggest honesty fix.
2. OK to make `faithfulness` and `hallucination_rate` share one judge pass?
3. Confirm the overall-score weight set and the "≥2 metrics / >50% error" rule.
4. Should non-CI large runs **require** cost confirmation, or just warn?
5. Interactive `init` that probes the endpoint — in scope for MVP, or keep `init`
   dumb and push probing into `inspect`?
