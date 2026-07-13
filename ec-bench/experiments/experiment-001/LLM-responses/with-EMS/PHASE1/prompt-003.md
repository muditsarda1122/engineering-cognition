model: kimi k2.5

Now I have the complete picture. Let me answer and then update the EMS artifacts.
Why get_request_handler is a Factory Function
get_request_handler (routing.py:367) is a factory because it produces a per-request handler closure that is constructed once at route-registration time and then reused for every request hitting that route.
The factory receives the entire route configuration (dependant, body_field, response_class, etc.), performs one-time setup logic, and returns app(request) — the inner function that actually runs when a request arrives.
This separation matters for performance and correctness:
- Setup-time logic runs once: Determining whether the endpoint is coroutine vs. sync (is_coroutine = dependant.is_coroutine_callable), whether the body is a form (is_body_form), which concrete response class to use (actual_response_class), and whether this is an SSE stream (is_sse_stream). These are expensive introspection operations that would be wasteful to repeat on every request.
- Request-time logic runs per-request: Reading the body, solving dependencies, calling the endpoint, and serializing the response. The inner function receives only the Request object and resolves everything else from the closure-captured variables.
Setup-Time Parameters (Captured in the Closure)
These are passed to get_request_handler() at line 1217 in APIRoute.get_route_handler():
Parameter	What it carries
dependant: Dependant	The complete dependency graph (built by get_dependant() at route registration). Contains the endpoint function, all parameter fields, sub-dependencies, path, etc.
body_field: ModelField | None	The Pydantic model field describing the request body (if any).
status_code: int | None	Default status code for this route.
response_class: type[Response] | DefaultPlaceholder	The response class (e.g., JSONResponse, StreamingResponse).
response_field: ModelField | None	The Pydantic model field for response validation/serialization.
response_model_include / exclude / by_alias / exclude_unset / exclude_defaults / exclude_none	Filters controlling how the response model is serialized.
dependency_overrides_provider: Any | None	The app object that holds dependency_overrides for testing.
embed_body_fields: bool	Whether body parameters are embedded.
strict_content_type: bool | DefaultPlaceholder	Whether to enforce strict content-type checking.
stream_item_field: ModelField | None	Field for stream-item validation (SSE/JSONL).
is_json_stream: bool	Whether this is a JSONL streaming endpoint.
Also computed once and captured:
- is_coroutine = dependant.is_coroutine_callable (line 386)
- is_body_form = body_field and isinstance(body_field.field_info, params.Form) (line 387)
- actual_response_class (resolved from DefaultPlaceholder at lines 388–391)
- is_sse_stream (line 392)
- actual_strict_content_type (resolved from DefaultPlaceholder at lines 393–396)
Per-Request Data (Inside the Inner app Function)
The inner async def app(request: Request) -> Response (line 398) receives only:
Source	What it is
request: Request	The sole explicit parameter — the incoming HTTP request.
request.scope["fastapi_middleware_astack"]	The outermost AsyncExitStack (for file cleanup).
request.scope["fastapi_inner_astack"]	The request-scoped AsyncExitStack (for dependency cleanup).
request.body() / request.form()	The raw request body bytes or form data.
request.path_params / request.query_params / request.headers / request.cookies	Parsed request parameters.
solved_result.values	The dictionary of resolved dependency values and validated parameters (from solve_dependencies).
solved_result.background_tasks	Accumulated background tasks.
solved_result.response.headers	Response headers set by dependencies.
Everything the inner function needs to execute — the endpoint function, the response model, the status code, the serialization filters — comes from the closure-captured variables set up at route registration time.
Where the Closure Lives
APIRoute.__init__ (line 1208) wires it into the ASGI pipeline:
self.app = request_response(self.get_route_handler())
get_route_handler() (line 1210) calls get_request_handler(...) with all the route's configuration fields. The returned app(request) closure is then wrapped by request_response() to create the final ASGI app. This entire chain is built once per route at app startup.