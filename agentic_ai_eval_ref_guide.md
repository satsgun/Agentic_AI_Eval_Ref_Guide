# Agentic AI Evaluation — Reference Table
 
A general-purpose reference for evaluating agentic LLM systems. The organizing principles are: **evaluate per stage, not just end-to-end**, and **match the scoring method to how crisp the ground truth is** at each stage — exact-match where possible, constraint satisfaction where multiple correct answers exist, and grounded/judged scoring where the output is natural language.
 
---
 
## Core principle: real model, mocked environment
 
Before the table — the distinction the scoring approach depends on:
 
| Component | Real or mocked | Why |
|---|---|---|
| The LLM / agent reasoning (any stage that calls the model) | **Real** | Mocking the model would score a fake; the whole point is to measure actual model quality. Non-determinism here is expected, not a bug. |
| External systems the agent acts on (APIs, databases, file systems, tools) | **Mocked / fixture-controlled** | Gives stable, known ground truth to score against; removes flakiness and rate limits; allows constructing adversarial conditions the real environment won't reliably produce. |
 
Anything that calls the model should be run for real. Anything the model *acts on* should generally be a controlled fixture, at least for the harness's core suite.
 
---
 
## What to measure and how, by stage
 
| Pipeline stage | What it does | Ground-truth crispness | What to measure | How to score | Notes |
|---|---|---|---|---|---|
| **Understanding / decomposition** (parsing intent, extracting structured requirements from a natural-language input) | Turns an ambiguous request into a structured spec of what's needed | **Crisp** — output is structured | Field-level correctness: entity extraction, classification fields, required-parameter selection | Exact match or set match per field; classification accuracy + confusion matrix; set precision/recall for multi-value fields | Highest-value, most defensible metrics in the harness — invest rigor here first. Flag any fields that are inherently fuzzy (e.g. time-window resolution) and score those with an explicit tolerance rather than forcing false precision. |
| **Planning / tool selection** (deciding what actions or tool calls to make and in what order) | Turns the structured spec into an executable plan | **Constraint-satisfiable**, rarely single-answer | Tool-set correctness, ordering/dependency constraints, parallelism where independent, no redundant or missing steps | Check invariants and constraints, not a single golden plan — score set match on tools plus boolean checks per constraint | Multiple valid plans are normal; scoring against one golden plan punishes correct-but-different solutions. This is where "did it reason correctly," not "did it match," should be tested. |
| **Execution** (actually calling tools / APIs against an environment) | Carries out the plan against real or mocked systems | **Deterministic once mocked** — not a model-quality question | Correct dispatch, correct handling of edge-case environment responses (empty result, multiple matches, error) | Ordinary integration testing — assertions, not LLM eval scoring | Not model evaluation. Its only role in the harness is to hand the next stage known, controlled data. Don't spend eval effort here. |
| **Synthesis / composition** (producing the final natural-language or user-facing output) | Turns retrieved/derived data into an answer or action for the user | **Soft** — natural language, rarely exact-matchable | Factual grounding against known input data; task-relevant properties (directness, completeness, tone); avoidance of contradiction or fabrication | Split in two: (1) **programmatic grounding checks** against known fixture data for a curated subset — cheap, rigorous, catches factual contradiction; (2) **LLM-as-judge** with a fixed rubric for subjective quality, used for relative/regression comparison, not absolute truth | The grounding check is the higher-value half — it converts part of a "soft" problem back into something crisp because the upstream data is known. Don't let the judge stand in as an oracle. |
| **End-to-end task success** | The full pipeline, input to output | **Mixed** — proxy signal | Binary or graded task success on full-pipeline cases | Simple success rate over a curated case set | A sanity layer on top of per-stage scores, not a substitute for them — end-to-end success alone can't localize *where* a failure happened. |
 
---
 
## Cross-cutting measurement concerns
 
| Concern | What it addresses | How to handle it |
|---|---|---|
| **Non-determinism / variance** | Real model calls produce different output across runs on the same input | Run each case **N times** (e.g. 5) and report distributions (mean ± spread), not single pass/fail. Deterministic (mocked) stages don't need this. |
| **Baseline & regression detection** | Prompt or model changes can silently degrade quality | Commit a baseline of averaged per-stage scores; diff future runs against it; set alert thresholds **above** measured run-to-run variance so noise doesn't trigger false alarms. Score per stage so a regression **localizes** to a specific prompt/component. |
| **Cost & latency** | Every call has a real, measurable price in money and time | Instrument token counts and wall-clock latency on every eval run (nearly free, since the pipeline runs anyway). Use this to validate architectural cost tradeoffs empirically and to justify model-routing decisions (cheap model where quality holds, expensive model where it doesn't) with real numbers rather than assumption. |
| **Guardrail / safety behavior** | Agent must stay within permitted actions, resist adversarial input, fail safely | Construct adversarial and out-of-scope fixtures deliberately (not just "happy path" cases); assert the agent declines, escalates, or fails safely rather than fabricating or overstepping permissions. |
| **Dataset design** | What to actually build the golden set from | Prefer a small number of **rich, curated cases** (each recording expected *intermediate* artifacts per stage, not just input→output) over a large number of shallow ones. Cover the realistic category taxonomy for the task, including edge cases (ambiguous input, not-found/zero-match, multi-match, fallback-routing) — not just the common path. |
 
---
 
## Quick decision guide
 
| If the stage's output is... | Then score it with... |
|---|---|
| A structured object with defined fields | Field-level exact/set match, classification accuracy, confusion matrices |
| One of several valid solutions (a plan, a query, a sequence) | Constraint satisfaction / invariant checks, not golden-output matching |
| A deterministic call into a controlled environment | Ordinary assertions (integration testing), not eval scoring |
| Natural language grounded in known data | Programmatic grounding checks first; LLM-as-judge second, for the residual subjective quality |
| The full pipeline, black-box | A coarse task-success rate, used as a sanity check alongside (not instead of) per-stage scores |
 
---
 
## The one-line summary
 
Call the model for real wherever reasoning is being evaluated; mock the environment wherever ground truth needs to be stable; score each stage with the crispest method its output actually supports; measure variance before trusting a baseline; and capture cost and latency for free while doing all of the above.
 
