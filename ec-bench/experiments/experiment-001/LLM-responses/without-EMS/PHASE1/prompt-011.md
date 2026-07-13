model: kimi k2.5

Integration Points Requiring Change
1. fastapi/routing.py — get_request_handler factory scope (lines 367-397)
Responsibility today: Empty space where setup-time booleans are computed (is_coroutine, is_body_form, is_sse_stream, actual_response_class, actual_strict_content_type at lines 385-396). No helper functions are defined here.
Why it must change: This is where _serialize_data, _serialize_sse_item, and _serialize_item must move to. They currently live inside the per-request app function (lines 487, 516, 640) but only need route-static variables (the 7 serialization parameters from get_request_handler scope) plus endpoint_ctx, which can be passed explicitly.
How the proposed architecture interacts: The factory scope becomes the new home for the three serialization helpers. They are created once when get_request_handler runs (at route registration time) and reused for the lifetime of the route. Their signatures gain an endpoint_ctx: EndpointContext | None = None parameter to replace the implicit closure capture.
Change type: Structural — code is repositioned, not rewritten.
2. fastapi/routing.py — SSE streaming branch (lines 512-635)
Responsibility today: Defines _serialize_sse_item (line 516), _sse_producer_cm (line 551), _producer (line 572), _keepalive_inserter (line 581), and _sse_with_checkpoints (line 613). The _producer coroutine calls _serialize_sse_item(raw_item) at line 575.
Why it must change: _serialize_sse_item is removed from this branch (it's now at factory scope). _producer must pass endpoint_ctx to the factory-scope _serialize_sse_item explicitly.
How the proposed architecture interacts: _producer gains one additional closure capture (endpoint_ctx from app scope) to pass it to _serialize_sse_item. The streaming infrastructure (_sse_producer_cm, memory streams, task group) remains unchanged because it depends on per-request variables (gen, sse_aiter) that cannot be hoisted.
Change type: Structural — call site plumbing.
3. fastapi/routing.py — JSONL streaming branch (lines 636-668)
Responsibility today: Defines _serialize_item (line 640), _async_stream_jsonl (line 645), and _sync_stream_jsonl (line 657). The async/sync generators call _serialize_item(item) at lines 647 and 659.
Why it must change: _serialize_item is removed from this branch. The _async_stream_jsonl and _sync_stream_jsonl generators must pass endpoint_ctx to the factory-scope _serialize_item explicitly.
How the proposed architecture interacts: The JSONL generators are per-request (they capture gen from app scope), so they can also capture endpoint_ctx and forward it. The + b"\n" logic and anyio.sleep(0) checkpoints remain untouched.
Change type: Structural — call site plumbing.
4. fastapi/routing.py — Non-streaming response path (lines 689-734)
Responsibility today: Calls serialize_response (line 293, a module-level function) with explicit parameters including endpoint_ctx at line 721.
Why it must change: It does not. This path already uses the exact pattern the proposal introduces: a module/factory-scope function that receives endpoint_ctx explicitly rather than capturing it via closure.
How the proposed architecture interacts: The streaming path becomes consistent with the non-streaming path. Both use explicit endpoint_ctx parameter passing to the serializer.
Change type: None — this is the existing model.
What Does NOT Need to Change
Component	Why unchanged
fastapi/routing.py: get_request_handler signature	The function still takes the same 16 parameters. Only its internal layout changes.
fastapi/routing.py: APIRoute.__init__	Still calls get_route_handler() which calls get_request_handler() with the same arguments.
fastapi/routing.py: request_response	Still wraps the app closure. No change to its interface.
fastapi/routing.py: solve_dependencies	Returns SolvedDependency with the same fields. The errors list is checked the same way.
fastapi/applications.py	FastAPI → APIRouter → APIRoute delegation chain is unchanged.
tests/test_stream_cancellation.py	Behavior is preserved (streaming, cancellation, checkpoints).
tests/test_stream_json_validation_error.py	ResponseValidationError path is preserved.
tests/test_stream_bare_type.py	JSONL serialization output is unchanged.
No new imports are needed — EndpointContext (line 63), ResponseValidationError (line 66), jsonable_encoder (line 61), and json (line 7) are already imported.
Adjacent Components That Could Benefit
1. fastapi/routing.py: serialize_response (line 293) — Already uses the pattern
This is the non-streaming counterpart. It is a module-level function that receives endpoint_ctx explicitly (line 721 in the non-streaming path). It validates the exact same way (field.validate(..., loc=("response",))) and raises ResponseValidationError with endpoint_ctx. The fact that the non-streaming path already uses explicit parameter passing validates that the proposed pattern is consistent with the codebase's established style.
2. fastapi/routing.py: get_websocket_app (line 748) — Candidate for future application
Currently simple (no nested serialization helpers), but if WebSocket streaming endpoints are ever added (e.g., streaming JSON over WebSocket), the same factory-scope helper pattern should be applied rather than nesting helpers inside the per-request app(websocket) function.
3. fastapi/routing.py: run_endpoint_function (line 336) — Already extracted
This helper runs the endpoint function and is defined at module level, not inside app. It demonstrates the codebase's general preference for extracting pure logic out of the per-request closure when possible. The proposed change brings the streaming serialization helpers in line with this existing pattern.
4. fastapi/openapi/utils.py: get_openapi_path (line 260) — Pattern consistency
The OpenAPI path generation logic is already organized as a series of module-level functions (get_openapi_security_definitions, _get_openapi_operation_parameters, get_openapi_operation_request_body). While outside the scope of request handling, it shows the codebase's organizational preference for explicit parameter passing over deep closure nesting — the same principle the proposal applies to get_request_handler.