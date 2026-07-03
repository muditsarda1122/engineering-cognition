# Prompt Taxonomy

Software engineering is not a sequence of independent coding tasks.

Instead, it is a continuous process of accumulating, refining, and reusing engineering understanding.

EC-Bench therefore organizes prompts according to how they interact with engineering cognition rather than simply what engineering activity they represent.

---

## Phase 1 — Investigation

Purpose

Discover facts about the repository.

Typical prompts

- Understand architecture
- Identify module responsibilities
- Explain data flow
- Investigate dependencies
- Trace execution paths

Primary outcome

Produces new engineering observations.

These observations become candidate engineering cognition.

---

## Phase 2 — Architectural Reasoning

Purpose

Transform repository observations into engineering understanding.

Typical prompts

- Explain why the system was designed this way
- Compare multiple implementations
- Infer architectural principles
- Discuss tradeoffs
- Predict consequences of design changes

Primary outcome

Converts observations into higher-level engineering understanding.

---

## Phase 3 — Implementation

Purpose

Apply accumulated understanding to modify the repository.

Typical prompts

- Implement features
- Refactor modules
- Extend APIs
- Improve abstractions

Primary outcome

Engineering cognition should guide implementation decisions.

---

## Phase 4 — Debugging

Purpose

Investigate failures using previously accumulated understanding.

Typical prompts

- Fix failing tests
- Diagnose runtime bugs
- Explain regressions
- Trace unexpected behavior

Primary outcome

Successful debugging should require reusing prior engineering cognition rather than rediscovering repository structure.

---

## Phase 5 — Planning

Purpose

Evaluate whether engineering cognition improves future engineering decisions.

Typical prompts

- Design future features
- Estimate implementation complexity
- Recommend architectural changes
- Identify technical debt
- Produce implementation plans

Primary outcome

This phase evaluates whether accumulated engineering cognition compounds over time.

---

Each EC-Bench experiment intentionally mixes these prompt categories to simulate realistic long-horizon software development.