# Agentic AI Evaluation Reference Guide

A practical, stage-by-stage framework for evaluating agentic LLM systems in production.

## Overview

This guide provides a systematic approach to evaluating agentic AI systems, emphasizing **per-stage evaluation** rather than end-to-end metrics alone. It balances rigor with practicality by:

- **Calling the model for real** where reasoning quality matters
- **Mocking external systems** to get stable, deterministic ground truth
- **Scoring each stage** with methods that match the crispness of its output
- **Tracking variance** across runs to catch regressions

## Core Principle: Real Model, Mocked Environment

| Component | Real or Mocked? | Why |
|---|---|---|
| LLM reasoning (any model call) | **Real** | Mocking the model scores a fake; the point is to measure actual quality |
| External systems (APIs, databases, tools) | **Mocked** | Gives stable ground truth, removes flakiness and rate limits |

## What to Measure, by Pipeline Stage

### Understanding / Decomposition
Turns ambiguous natural-language input into a structured spec.
- **Output type:** Structured object
- **Scoring method:** Field-level exact/set match, classification accuracy
- **Ground truth:** Crisp (well-defined structure)

### Planning / Tool Selection
Converts the spec into an executable plan or sequence of tool calls.
- **Output type:** Multiple valid solutions exist
- **Scoring method:** Constraint satisfaction, invariant checks
- **Ground truth:** Constraint-satisfiable (rarely single-answer)

### Execution
Calls tools/APIs against the environment.
- **Output type:** Deterministic once mocked
- **Scoring method:** Standard assertions (integration test style)
- **Ground truth:** Deterministic (not a model-quality question)

### Synthesis / Composition
Produces the final user-facing output (natural language, structured response, etc.).
- **Output type:** Natural language grounded in data
- **Scoring method:** Programmatic grounding checks first; LLM-as-judge for residual quality
- **Ground truth:** Soft (subjective elements)

### End-to-End Task Success
Full pipeline, input to output.
- **Output type:** Binary or graded success
- **Scoring method:** Simple success rate on curated case set
- **Ground truth:** Mixed (proxy signal; use with per-stage scores)

## Key Measurement Concerns

### Non-Determinism / Variance
Real model calls produce different outputs on the same input.
- **Solution:** Run each case **N times** (e.g., 5×), report distributions (mean ± std dev)

### Baseline & Regression Detection
Prompt or model changes can silently degrade quality.
- **Solution:** Commit baselines; diff future runs against them; set alert thresholds above expected variance

### Cost & Latency
Every call has real costs.
- **Solution:** Instrument token counts and wall-clock latency on every run; track trends

### Guardrail / Safety Behavior
Agent must stay within bounds and fail safely.
- **Solution:** Build adversarial and out-of-scope fixtures deliberately; don't just test happy paths

### Dataset Design
Quality of evaluation depends on cases, not quantity.
- **Recommendation:** Build a small set of **rich, curated cases**, each with expected intermediate artifacts per stage

## Quick Decision Guide

| If the stage's output is... | Then score it with... |
|---|---|
| A structured object with defined fields | Field-level exact/set match, classification accuracy |
| One of several valid solutions (plan, query, sequence) | Constraint satisfaction / invariant checks |
| A deterministic call into a controlled environment | Standard assertions (integration testing) |
| Natural language grounded in known data | Programmatic grounding checks, then LLM-as-judge |
| The full pipeline, black-box | Coarse task-success rate (sanity check alongside per-stage scores) |

## The One-Line Summary

Call the model for real wherever reasoning is being evaluated; mock the environment wherever ground truth needs to be stable; score each stage with the crispest method its output actually supports; measure variance and track baselines.

## Usage

This guide is intended as a **reference** for:
- Designing evaluation harnesses for agentic LLM systems
- Choosing scoring methods that match the nature of each pipeline stage
- Setting up repeatable, regression-detecting evaluation suites
- Balancing rigor with practicality in production settings

## File Structure

- `agentic_ai_eval_ref_guide.md` — Full reference table with detailed explanations

## Contributing

If you'd like to suggest improvements, additions, or alternative approaches, please open an issue or pull request.

## License

Feel free to use and adapt this guide for your own evaluation needs.
