Phase: Investigation  

Objective: Trace the exact code path for a plain non-streaming GET request.  

Prompt:  
Now that I know the modules, I want to trace a single GET /items/ request end-to-end. Starting from FastAPI.__call__, walk me through every significant function call until the endpoint function returns and the response is sent. Include build_middleware_stack, request_response, get_request_handler, solve_dependencies, and serialize_response. I want to see the full call stack in order.

Expected engineering knowledge accumulated after completing the prompt:  
A mental map of the exact call chain for a normal request, including where Starlette ends and FastAPI begins.

Why this prompt naturally follows from previous work:  
Prompt 1 gave the module names; Prompt 2 connects them into a chronological execution trace.