Phase: Investigation  

Objective: Orient yourself in the repository and identify the core modules that participate in the HTTP request path.  

Prompt:  
I just joined the FastAPI team. Before I touch any code, I need to understand the repository layout. Walk me through the responsibilities of fastapi/routing.py, fastapi/dependencies/utils.py, fastapi/applications.py, and fastapi/openapi/utils.py. For each module, tell me the single most important class or function that an incoming HTTP request flows through.

Expected engineering knowledge accumulated after completing the prompt:  
Understanding of the four pillars of FastAPI (routing, DI, app factory, OpenAPI) and which symbols are on the critical path.

Why this prompt naturally follows from previous work:  
This is the foundational orientation step; nothing precedes it.