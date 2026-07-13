Phase: Debugging  

Objective: Fix linter issues.  

Prompt:  
Run ruff check fastapi/routing.py and ruff format --check fastapi/routing.py. Fix any line-length, import-order, or unused-variable issues. Ensure the file still matches the existing code style (e.g., no trailing whitespace, single quotes for non-docstring strings where appropriate).

Expected engineering knowledge accumulated after completing the prompt:  
Code that meets FastAPI's lint bar and is ready for review.

Why this prompt naturally follows from previous work:  
Prompt 25 fixed types; Prompt 26 fixes style.