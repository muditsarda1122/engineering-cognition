# Evaluation Metrics

EC-Bench evaluates whether accumulated engineering cognition improves software engineering over long-horizon development workflows.

Every response is evaluated independently across multiple dimensions. Scores are continuous from **0–10** and should reflect engineering quality rather than writing quality.

No metric is intended to measure factual correctness in isolation. Instead, the metrics collectively evaluate whether accumulated engineering cognition changes **how software engineering is performed over time**.

---

# 1. Architectural Continuity

## Intuition

Does the agent continue engineering the same system rather than repeatedly redesigning it?

## Definition

Architectural Continuity measures whether the agent preserves and extends previously established architectural decisions throughout a long-running engineering workflow.

The judge should evaluate whether the response naturally builds upon earlier engineering decisions instead of introducing unnecessary architectural changes, conflicting abstractions, or inconsistent system behavior.

## Measurement

Evaluate whether the response:

- preserves previously established architecture
- reuses existing abstractions where appropriate
- extends the current design naturally
- avoids unnecessary redesign
- remains consistent with prior implementation decisions

## Score Range

Scores range continuously from **0 to 10**.

Anchor points:

- **0** — Completely contradicts or ignores previously established architecture.
- **5** — Partially consistent but introduces avoidable architectural inconsistencies.
- **10** — Extends the existing architecture coherently while preserving engineering continuity.

Intermediate scores (e.g. 3.5, 6, 8.5) should be used whenever appropriate.

## Judge Guidance

The judge should explain why the assigned score was given, referencing concrete architectural decisions whenever possible.

## Why It Matters

Experienced engineers rarely redesign systems unnecessarily. They evolve existing architecture. Engineering cognition should encourage the same behavior.

---

# 2. Repository Groundedness

## Intuition

Does the agent engineer *this repository*, rather than producing generic software engineering advice?

## Definition

Repository Groundedness measures how well the response understands and utilizes the actual repository under development.

The judge should evaluate whether the response demonstrates awareness of repository structure, existing implementation patterns, naming conventions, dependencies, and architecture.

## Measurement

Evaluate whether the response:

- references existing repository components appropriately
- follows repository conventions
- builds upon existing implementations
- demonstrates understanding of repository architecture
- avoids generic recommendations disconnected from the codebase

## Score Range

Scores range continuously from **0 to 10**.

Anchor points:

- **0** — Generic response with little or no repository awareness.
- **5** — Some repository grounding but inconsistent integration.
- **10** — Deeply grounded in the repository's existing implementation and design.

Intermediate scores should be used whenever appropriate.

## Judge Guidance

The judge should cite repository-specific evidence supporting the score.

## Why It Matters

Engineering cognition should make the agent increasingly aware of the repository it is engineering, rather than repeatedly approaching it as an unfamiliar project.

---

# 3. Engineering Cognition Reuse

## Intuition

Does previously accumulated engineering understanding influence the current engineering task?

## Definition

Engineering Cognition Reuse measures whether the response meaningfully builds upon engineering understanding accumulated during previous work instead of rediscovering the same knowledge.

This metric evaluates whether accumulated cognition changes the agent's engineering reasoning.

## Measurement

Evaluate whether the response:

- builds upon previously established engineering understanding
- avoids rediscovering earlier conclusions
- leverages prior implementation rationale
- incorporates previous debugging discoveries
- demonstrates continuity of engineering understanding across sessions

## Score Range

Scores range continuously from **0 to 10**.

Anchor points:

- **0** — Behaves as though no previous engineering understanding exists.
- **5** — Reuses some previous understanding but still reconstructs significant context.
- **10** — Naturally incorporates accumulated engineering understanding throughout the solution.

Intermediate scores should be used whenever appropriate.

## Judge Guidance

The judge should identify examples where accumulated engineering understanding clearly influenced the response.

## Why It Matters

This is the core hypothesis of EC-Bench.

If Engineering Cognition does not improve knowledge reuse, the entire premise of accumulated engineering understanding becomes questionable.

---

# 4. Engineering Quality

## Intuition

Independent of cognition, is this simply good engineering?

## Definition

Engineering Quality measures the overall quality of the engineering solution produced.

This metric intentionally evaluates engineering craftsmanship rather than repository familiarity or accumulated cognition.

## Measurement

Evaluate whether the response:

- proposes sound engineering decisions
- demonstrates good software design
- balances simplicity and extensibility
- considers maintainability
- produces technically robust solutions

## Score Range

Scores range continuously from **0 to 10**.

Anchor points:

- **0** — Poor engineering with major design flaws.
- **5** — Reasonable engineering but with notable weaknesses.
- **10** — Excellent engineering demonstrating strong software design principles.

Intermediate scores should be used whenever appropriate.

## Judge Guidance

Focus on engineering merit rather than implementation correctness alone.

## Why It Matters

Engineering cognition should improve engineering—not merely preserve context.

---

# 5. Debugging and Investigation Efficiency

## Intuition

Does accumulated engineering understanding make investigation and debugging more systematic?

## Definition

Debugging and Investigation Efficiency measures how effectively the response investigates problems, forms hypotheses, narrows the search space, and identifies root causes.

The metric evaluates engineering process rather than merely arriving at a correct answer.

## Measurement

Evaluate whether the response:

- investigates systematically
- forms reasonable engineering hypotheses
- narrows the debugging search efficiently
- avoids unnecessary exploration
- reaches root causes using previous engineering understanding where appropriate

## Score Range

Scores range continuously from **0 to 10**.

Anchor points:

- **0** — Random or inefficient debugging process.
- **5** — Reasonable investigation but with avoidable inefficiencies.
- **10** — Highly systematic investigation demonstrating mature engineering reasoning.

Intermediate scores should be used whenever appropriate.

## Judge Guidance

Evaluate the quality of the debugging methodology, not simply whether the bug was fixed.

## Why It Matters

One of the strongest manifestations of accumulated engineering cognition should be increasingly efficient investigation and debugging over time.