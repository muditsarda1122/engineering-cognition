# Engineering Memory System (EMS)

Engineering Memory System (EMS) is the first experimental implementation developed during the exploration of Engineering Cognition.

The objective of EMS is not to serve as a production-ready memory system, but to investigate whether accumulated engineering understanding can improve long-horizon software development performed by AI coding agents.

The implementation explores practical questions such as:
- How engineering cognition can be extracted from software engineering activity.
- How candidate cognition should be reviewed before becoming persistent.
- How engineering cognition can be organized hierarchically.
- How relevant cognition can be retrieved for future engineering tasks.
- How accumulated cognition can be injected back into an agent's reasoning process.

EMS should therefore be viewed as an experimental platform rather than a finalized architecture.

Its purpose is to generate evidence that can either support or invalidate the hypotheses proposed in the Engineering Cognition paper.

## Current Status

Current capabilities include:

- Repository history analysis
- Engineering session tracking
- Engineering cognition proposal generation
- Human review workflow
- Persistent cognition hierarchy
- Context retrieval and ranking
- Prompt augmentation using accumulated cognition

## Relationship to Engineering Cognition

The accompanying Engineering Cognition paper deliberately separates the research problem from its implementation.

Engineering Cognition asks questions such as:

- What constitutes engineering understanding?
- Can engineering cognition be extracted automatically?
- Does engineering cognition admit a canonical representation?
- Can accumulated cognition improve long-horizon agent performance?

EMS represents one possible attempt at answering those questions.

Future implementations may take entirely different architectural approaches while still pursuing the same underlying research direction.

## Repository

The research paper can be found in:

```
/paper/
```

The planned benchmark for evaluating engineering cognition can be found in:

```
/ec-bench/
```