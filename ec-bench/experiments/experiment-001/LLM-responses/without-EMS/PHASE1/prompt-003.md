model: kimi k2.5

get_request_handler (routing.py:367) is a factory because all route metadata is bound at import-time, leaving only the Request object to be resolved per-request. This design separates static route configuration from dynamic request processing, which is critical for performance and for plugging into Starlette’s request_response wrapper.
Why a factory?
The function is called once per route during startup (via APIRoute.get_route_handler → APIRoute.__init__ wiring self.app = request_response(self.get_route_handler())). The closure it returns must match Starlette’s expected signature:
Callable[[Request], Coroutine[Any, Any, Response]]
request_response (routing.py:113) takes exactly that — a single-argument function f(request) — and wraps it into an ASGI app. If get_request_handler were not a factory, every piece of route metadata would have to be passed on every request, or looked up dynamically from the route object. The factory freezes everything into the closure, so the hot-path inner function only needs to receive the Request and produce a Response.
Setup-time parameters (captured as closures)
These are passed to get_request_handler once and never change:
Parameter	How it is used at setup time
dependant	The Dependant tree built by get_dependant at import time
body_field	Used to decide is_body_form (line 387)
status_code	Default status for the response
response_class	Unwrapped from DefaultPlaceholder → actual_response_class (lines 388-391)
response_field	Pydantic field for response validation/serialization
response_model_include	Captured into serialize_response call
response_model_exclude	Captured into serialize_response call
response_model_by_alias	Captured into serialize_response call
response_model_exclude_unset	Captured into serialize_response call
response_model_exclude_defaults	Captured into serialize_response call
response_model_exclude_none	Captured into serialize_response call
dependency_overrides_provider	Passed to solve_dependencies
embed_body_fields	Passed to solve_dependencies
strict_content_type	Unwrapped from DefaultPlaceholder → actual_strict_content_type (lines 393-396)
stream_item_field	Used for SSE/JSONL item validation
is_json_stream	Determines if this is a JSONL streaming route
Plus setup-time computed booleans that avoid re-computation on every request:
- is_coroutine = dependant.is_coroutine_callable (line 386)
- is_body_form = body_field is a params.Form (line 387)
- is_sse_stream = actual_response_class is a subclass of EventSourceResponse (line 392)
Per-request resolved inside async def app(request)
The inner app function (line 398) receives only the Request and resolves everything else from it:
Variable	Source
file_stack	request.scope["fastapi_middleware_astack"] (line 400) — created by AsyncExitStackMiddleware
endpoint_ctx	_extract_endpoint_context(dependant.call) (line 407) — source/line metadata for error messages
mount_path	request.scope.get("root_path", "") (line 414) — for mounted sub-apps
body	await request.body() / await request.json() / await request.form() (lines 421-442)
async_exit_stack	request.scope["fastapi_inner_astack"] (line 469) — created by request_response
solved_result	await solve_dependencies(...) (line 473) — fully dynamic: resolves deps + validates params
raw_response	await run_endpoint_function(...) (line 690) — the actual endpoint function result
content	await serialize_response(...) (line 711) — validates and serializes the return value
response	Constructed per-request from content + response_args (lines 725/731)
The nested closure pattern
Even inside the inner app, there is a second level of closure capture. The _serialize_data helper (line 487) is defined inside app and captures stream_item_field and all response_model_* parameters. This means the serialization logic is also pre-bound — when _serialize_data is called later (inside _serialize_sse_item or _serialize_item), it does not re-resolve those parameters; they are already frozen in the closure.