# Evaluation Metrics

EC-Bench evaluates whether accumulated engineering cognition improves software engineering over long-horizon development workflows.

Every response is evaluated independently across multiple dimensions. Scores are continuous from **0–10** and should reflect engineering quality rather than writing quality. No metric is intended to measure factual correctness in isolation. Instead, the metrics collectively evaluate whether accumulated engineering cognition changes how software engineering is performed over time.

---

# 1. Architectural Consistency

## Intuition

Does the answer remain consistent with architectural decisions established earlier in the engineering workflow?

## Definition

Architectural Consistency measures whether the agent preserves previously established system design decisions while extending the software.

The judge should evaluate whether the response naturally builds upon earlier engineering decisions instead of contradicting them or inventing unnecessary architectural changes.

## Measurement

Evaluate whether the response:

- reuses existing abstractions
- respects previous architectural decisions
- avoids introducing contradictory system behavior
- extends the existing design naturally

## Score Range

Scores range continuously from **0 to 10**.

Anchor points:

- **0** — Completely contradicts previously established architecture.
- **5** — Partially consistent but introduces avoidable contradictions.
- **10** — Fully consistent while extending the architecture coherently.

Intermediate scores (e.g. 3.5, 6, 8.5) should be used whenever appropriate.

## Judge Guidance

The judge should explain *why* the assigned score was given, citing concrete examples from the response whenever possible.

## Why It Matters

Long-running engineering depends on preserving design decisions over time. If agents repeatedly change architecture unnecessarily, engineering cognition has failed.

---

# 2. Repository Groundedness

## Intuition

Is the response grounded in the actual repository rather than generic software knowledge?

## Definition

Repository Groundedness measures whether the agent reasons using the repository's actual implementation, naming conventions, abstractions, and history.

## Measurement

Evaluate whether the response:

- references existing files correctly
- uses repository terminology
- follows existing implementation patterns
- avoids generic assumptions

## Score Range

Scores range continuously from **0 to 10**.

Anchor points:

- **0** — Almost entirely generic.
- **5** — Mixes repository-specific reasoning with generic assumptions.
- **10** — Completely grounded in repository reality.

Intermediate scores should be assigned when appropriate.

## Judge Guidance

Explain which parts of the answer demonstrate repository-specific reasoning and which parts appear generic.

## Why It Matters

Engineering cognition should become increasingly repository-specific as knowledge accumulates.

---

# 3. Context Reconstruction Cost

## Intuition

How much context did the agent have to reconstruct before solving the task?

## Definition

This metric estimates the amount of reasoning spent rebuilding previously known engineering understanding instead of progressing the task.

## Measurement

Look for evidence that the agent:

- repeats earlier reasoning
- rediscovers previous conclusions
- asks itself questions already answered earlier
- spends effort rebuilding context

## Score Range

Scores range continuously from **0 to 10**.

Anchor points:

- **0** — Almost the entire response reconstructs old knowledge.
- **5** — Some unnecessary reconstruction occurs.
- **10** — Minimal reconstruction; immediately builds on existing understanding.

## Judge Guidance

Identify repeated reasoning that could have been avoided if engineering cognition had been available.

## Why It Matters

Reducing context reconstruction is one of the primary motivations behind Engineering Cognition.

---

# 4. Knowledge Reuse Rate

## Intuition

How effectively does the agent reuse previously accumulated engineering knowledge?

## Definition

Knowledge Reuse measures how much previously accumulated engineering understanding actively influences the current solution.

## Measurement

Evaluate whether the response:

- recalls earlier decisions
- builds on previous debugging work
- reuses earlier implementation insights
- connects prior engineering observations

## Score Range

Scores range continuously from **0 to 10**.

Anchor points:

- **0** — No previous knowledge reused.
- **5** — Some previous knowledge reused.
- **10** — Previous engineering understanding is heavily and appropriately reused.

## Judge Guidance

Explicitly identify reused knowledge whenever possible.

## Why It Matters

If cognition exists, later engineering should increasingly depend on earlier engineering.

---

# 5. Reasoning Efficiency

## Intuition

How much reasoning duplicates earlier reasoning?

## Definition

Repeated Reasoning measures how often the agent independently derives conclusions that have already been established earlier.

## Measurement

Look for:

- repeated architectural analysis
- repeated debugging
- repeated repository exploration
- repeated implementation planning

## Score Range

Scores range continuously from **0 to 10**.

Anchor points:

- **0** — Almost everything is repeated reasoning.
- **5** — Noticeable but acceptable repetition.
- **10** — Very little repeated reasoning.

## Judge Guidance

Whenever repetition is observed, identify exactly what reasoning was unnecessarily repeated.

## Why It Matters

One of the central hypotheses of Engineering Cognition is that accumulated understanding should reduce repeated reasoning.

---

# 6. Design Quality

## Intuition

Independent of previous knowledge, is the engineering solution itself good?

## Definition

Design Quality evaluates the intrinsic quality of the proposed engineering solution.

## Measurement

Evaluate:

- modularity
- maintainability
- readability
- correctness
- simplicity
- engineering tradeoffs

## Score Range

Scores range continuously from **0 to 10**.

Anchor points:

- **0** — Poor engineering solution.
- **5** — Acceptable engineering.
- **10** — Excellent engineering solution.

## Judge Guidance

Justify why the proposed solution represents good or poor engineering.

## Why It Matters

Engineering cognition should improve engineering quality, not merely memory.

---

# 7. Longitudinal Consistency

## Intuition

Does the agent behave like the same engineer over time?

## Definition

Longitudinal Consistency measures whether decisions remain internally coherent across the entire engineering workflow.

## Measurement

Evaluate whether:

- later answers align with earlier answers
- implementation evolves naturally
- priorities remain stable
- engineering philosophy remains consistent

## Score Range

Scores range continuously from **0 to 10**.

Anchor points:

- **0** — Behaves like a completely different engineer.
- **5** — Some continuity exists.
- **10** — Strong engineering continuity throughout.

## Judge Guidance

Look across the entire workflow instead of judging the response in isolation.

## Why It Matters

Engineering Cognition should produce continuity across weeks of engineering work rather than isolated intelligent responses.

---

# 8. Debugging Efficiency

## Intuition

How efficiently does the agent diagnose and resolve failures?

## Definition

Debugging Efficiency measures whether the agent converges toward the root cause using previous engineering understanding instead of blindly exploring possibilities.

## Measurement

Evaluate whether the response:

- identifies likely causes quickly
- avoids unnecessary debugging paths
- leverages previous debugging knowledge
- converges efficiently

## Score Range

Scores range continuously from **0 to 10**.

Anchor points:

- **0** — Random trial-and-error.
- **5** — Moderate debugging efficiency.
- **10** — Fast, systematic debugging driven by accumulated understanding.

## Judge Guidance

Explain whether the debugging process appears systematic or exploratory.

## Why It Matters

Experienced engineers debug faster because they accumulate understanding. Engineering cognition should enable the same behavior.

---

# 9. Decision Stability

## Intuition

Does the agent maintain stable engineering judgement throughout the workflow?

## Definition

Decision Stability measures whether the agent remains consistent in its engineering decisions across multiple prompts.

As engineering understanding accumulates, an engineer should become more confident and internally consistent rather than repeatedly changing opinions without new evidence.

A change in engineering direction is not inherently negative; however, changes should be justified by new information rather than inconsistent reasoning.

## Measurement

Evaluate whether the response:

- remains consistent with earlier engineering decisions
- avoids unnecessary reversals of previous conclusions
- changes direction only when new evidence justifies it
- demonstrates increasing confidence as understanding accumulates

## Score Range

Scores range continuously from **0 to 10**.

Anchor points:

- **0** — Frequently contradicts previous engineering decisions without justification.
- **5** — Decisions are moderately stable but exhibit occasional unexplained reversals.
- **10** — Engineering decisions remain highly consistent, with any changes clearly justified by new evidence.

Intermediate scores (e.g. 2, 4.5, 7.5, 9) should be assigned whenever they more accurately reflect the quality of the response.

## Judge Guidance

Identify whether the response changes previously established engineering conclusions.

If the decision changes, determine whether the change was justified by genuinely new information or resulted from inconsistent reasoning.

The reasoning behind the assigned score should explicitly describe why the decision appears stable or unstable.

## Why It Matters

Engineering Cognition should not merely help an agent remember previous work—it should help the agent develop stable engineering judgement over time.

Experienced engineers rarely oscillate between contradictory design decisions. Instead, accumulated understanding increases confidence while remaining adaptable when new evidence appears.