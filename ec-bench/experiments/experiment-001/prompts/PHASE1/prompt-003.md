Phase: Investigation  

Objective: Locate get_request_handler and understand its factory pattern.  

Prompt:  
Open fastapi/routing.py and read get_request_handler. Explain why it is a factory function that returns another function, and identify what parameters are passed at route-setup time versus what is resolved per-request inside the inner app function.

Expected engineering knowledge accumulated after completing the prompt:  
Clear separation between route-static parameters (e.g., response_field, stream_item_field) and per-request objects (e.g., request, body, endpoint_ctx).

Why this prompt naturally follows from previous work:  
Prompt 2 showed that get_request_handler is central; Prompt 3 zooms in on its signature and closure structure.