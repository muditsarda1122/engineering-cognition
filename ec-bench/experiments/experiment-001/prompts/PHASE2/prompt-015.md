Phase: Implementation

Objective:
Begin implementing the architectural approach selected during the design phase.

Prompt:
Implement the first production-quality component required by the architecture you previously selected.

This implementation should establish the primary abstraction upon which the remainder of the refactor will build.

Focus on:

- clean repository integration
- appropriate type annotations
- preserving existing behaviour
- minimizing disruption to the current implementation

Do not yet integrate the new abstraction into the request execution path.

Only implement the foundational component that future prompts will build upon.

Expected engineering knowledge accumulated after completing the prompt:
The primary architectural abstraction exists as an independent production-quality component, ready to be integrated into the request handling workflow.

Why this prompt naturally follows from previous work:
Prompts 10–14 established the architecture, integration strategy, behavioural constraints, and validation plan. Prompt 15 begins implementing that design incrementally.