Phase: Architectural Reasoning

Objective:
Identify the repository integration points required for the architectural approach you selected.

Prompt:
Based on the architectural approach you selected previously, identify every location in the FastAPI repository that would need to change to support the implementation.

For each integration point, explain:

- why it must change
- what responsibility it has today
- how your proposed architecture interacts with it
- whether the change is functional, structural, or purely organizational

Also identify any adjacent components that could benefit from the same architectural pattern, even if they are outside the scope of the current work.

Expected engineering knowledge accumulated after completing the prompt:
A complete map of the repository locations affected by the proposed architecture, reducing implementation risk and preventing overlooked integration points.

Why this prompt naturally follows from previous work:
Prompt 10 selected an architectural direction. Prompt 11 determines how that architecture integrates with the existing repository.