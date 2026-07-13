# Experiment Configuration

## Benchmark

EC-Bench (Engineering Cognition Benchmark)

---

## Repository

FastAPI

---

## Prompt Generation

Hand-written benchmark prompts designed to simulate a continuous software engineering workflow.

Total prompts: **30**

---

## Prompt Phases

| Phase | Focus |
|-------|-------|
| Phase 1 | Investigation & Architectural Reasoning |
| Phase 2 | Implementation & Debugging |
| Phase 3 | Planning |

Each phase represents a **new coding session**.

---

## Implementation Model

**Model:** Kimi K2.5

The implementation model was responsible for completing every engineering prompt under both experimental conditions.

---

## Judge Model

**Model:** GLM-5.2

The judge independently evaluated every response using the benchmark metrics defined in `metrics.md`.

---

## Comparison

Two identical engineering environments were evaluated.

| Condition | Description |
|-----------|-------------|
| Without EMS | Stateless reasoning. No accumulated Engineering Cognition between phases. |
| With EMS | Engineering Memory System enabled. Previously accumulated Engineering Cognition could be retrieved across engineering sessions. |

The **only independent variable** was the availability of accumulated Engineering Cognition.

---

## Repository State

Both experiments began from the same repository state.

Repository modifications were applied independently in each condition.

---

## Evaluation Metrics

Responses were independently evaluated using:

- Architectural Continuity
- Repository Groundedness
- Engineering Cognition Reuse
- Engineering Quality
- Debugging & Investigation Efficiency

See `metrics.md` for complete metric definitions.

---

## Research Hypotheses

The benchmark evaluates the hypotheses described in `hypothesis.md`.

---

## Judge Output

The judge produces:

- numerical scores (0–10)
- metric-wise reasoning
- overall score
- strengths
- weaknesses

---

## Random Seed

Not fixed.

The benchmark evaluates realistic engineering behaviour rather than deterministic decoding.