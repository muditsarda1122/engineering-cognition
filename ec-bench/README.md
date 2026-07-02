# EC-Bench

### Evaluating Accumulated Engineering Cognition in Long-Horizon Software Development

EC-Bench is a benchmark proposed to evaluate whether accumulated engineering cognition improves software engineering performed by AI agents.

Unlike existing benchmarks that primarily evaluate task completion, EC-Bench evaluates whether an agent can accumulate, reuse, and build upon engineering understanding across an extended development workflow.

---

## Motivation

Modern coding agents increasingly solve difficult implementation tasks.

However, software engineering is rarely a collection of isolated tasks.

Instead, engineering work is cumulative.

Developers investigate codebases, form architectural understanding, implement changes, debug failures, review designs, and later reuse the understanding acquired during previous work.

EC-Bench investigates whether AI systems can benefit from accumulated engineering cognition in a similar way.

---

## Central Hypothesis

> Engineering cognition reduces repeated reasoning by enabling agents to accumulate engineering understanding across long-horizon software development tasks.

---

## Repository Structure

benchmark-philosophy.md

Why EC-Bench exists.

benchmark-design.md

Overall benchmark design.

metrics.md

Evaluation metrics.

prompt-taxonomy.md

Prompt categories.

judges/

Judge agent design.

experiments/

Benchmark runs.

---

This benchmark is currently under active development.