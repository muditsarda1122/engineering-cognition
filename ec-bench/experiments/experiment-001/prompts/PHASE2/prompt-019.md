Phase: Implementation  

Objective: Clean up dead code and update docstrings.  

Prompt:  
Delete any now-unused inner function definitions and imports inside get_request_handler. Add docstrings to _StreamSerializers and its public methods following the annotated_doc / Doc style used elsewhere in FastAPI. Run ruff check fastapi/routing.py and fix any lint issues.

Expected engineering knowledge accumulated after completing the prompt:  
Clean, maintainable code that matches FastAPI's documentation-first style.

Why this prompt naturally follows from previous work:  
Prompts 15–18 wrote the code; Prompt 19 polishes it to repo standards.