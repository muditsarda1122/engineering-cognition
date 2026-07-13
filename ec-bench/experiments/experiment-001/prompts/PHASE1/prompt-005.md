Phase: Investigation 

Objective: Understand what variables each closure captures and which of those are truly per-request.  

Prompt:  
For the three most expensive closures—_serialize_data, _serialize_sse_item, and _sse_producer_cm—list every free variable they reference from the enclosing scope. Then classify each variable as either route-static (known when the route is created) or per-request (only available when app(request) runs).

Expected engineering knowledge accumulated after completing the prompt:  
A precise categorization that tells us exactly what can be hoisted and what must remain dynamic.

Why this prompt naturally follows from previous work:  
Prompt 4 found the closures; Prompt 5 analyzes their capture semantics, which is the prerequisite for any refactoring.