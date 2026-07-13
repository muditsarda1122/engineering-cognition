model: kimi k2.5

Request-Specific State Analysis for get_request_handler Refactoring
Summary
The get_request_handler factory (fastapi/routing.py:367) creates a per-request app closure containing 10 nested helpers. Under the proposed architecture, 3 helpers (_serialize_data, _serialize_sse_item, _serialize_item) move to factory scope, eliminating their per-request function-object creation. The remaining 7 helpers must stay per-request because they capture generator state (gen, sse_aiter) or request-scoped resources.
Only one request-specific variable is currently captured by the helpers that will move: endpoint_ctx. All other captured variables are route-static (7 serialization filters + stream_item_field). The proposal adds endpoint_ctx as an explicit parameter to the factory-scope helpers, requiring the per-request streaming generators to capture and forward it.
1. Request-Specific State That Must Remain Per-Request
These variables are currently captured by (or flow through) nested helpers, and cannot be hoisted to factory scope.
A. endpoint_ctx (routing.py:406–415)
Dimension	Detail
Why it exists	Provides endpoint path, HTTP method, function name, file, and line number for ResponseValidationError and RequestValidationError messages. Constructed from request.method, request.scope["root_path"], and dependant.path.
Classification	Request-specific. The path field includes the mount prefix from the request scope (line 414).
Current flow	Captured by _serialize_data (line 493). Indirectly captured by _serialize_sse_item (via _serialize_data at line 542) and _serialize_item (via _serialize_data at line 641).
Proposed flow	Removed from _serialize_data closure. Passed explicitly: _serialize_data(endpoint_ctx, data). Per-request streaming generators (_producer, _async_stream_jsonl, _sync_stream_jsonl) capture endpoint_ctx from app scope and forward it at call time.
Behavioral regression if mishandled	Error reporting breakdown: If endpoint_ctx is cached at factory scope, every ResponseValidationError would show the first request's path/method (or empty context if no request processed yet). Developers would see incorrect loc fields in validation errors, making debugging impossible in mounted sub-apps. <br><br> Fallback loss: The current code has ctx = endpoint_ctx or EndpointContext() (line 493). Removing this fallback would strip all endpoint context from streaming validation errors.
B. gen / sse_aiter (routing.py:514, 546–548, 638, 671)
Dimension	Detail
Why it exists	gen is the generator/async generator returned by the endpoint function with per-request dependency values. sse_aiter wraps gen for uniform async iteration in SSE (lines 546–548).
Classification	Request-specific. gen is created from dependant.call(**solved_result.values) where solved_result.values are resolved per-request.
Current flow	gen captured by _async_stream_jsonl (line 646) and _sync_stream_jsonl (line 658). sse_aiter captured by _producer (line 574).
Proposed flow	Must remain in app scope. _async_stream_jsonl, _sync_stream_jsonl, and _producer continue to capture them.
Behavioral regression if mishandled	Streaming execution failure: If gen or sse_aiter is cached at factory scope, the same generator executes across requests. Async generators raise RuntimeError: async generator already executing. Sync generators in threadpool race, interleaving data from different requests. <br><br> Data leakage: Yielded values from User A stream to User B's response. <br><br> Lifecycle corruption: Generator finalization (aclose()) is tied to request scope. Sharing causes premature finalization or never-finalized generators, leaking memory and file descriptors.
C. async_exit_stack (routing.py:469–472)
Dimension	Detail
Why it exists	Request-scoped AsyncExitStack from request.scope["fastapi_inner_astack"]. Manages dependency context managers and SSE task group lifecycle.
Classification	Request-specific. Retrieved from request scope.
Current flow	Used to enter _sse_producer_cm (line 606) and push sse_receive_stream.aclose callback (line 611). Not captured by nested helpers, but critical for SSE lifecycle.
Proposed flow	Must remain in app scope. _sse_producer_cm is entered on this stack regardless of helper location.
Behavioral regression if mishandled	Request lifecycle breakdown: If cached at factory scope, cleanup callbacks and task groups leak across requests. SSE streaming does not cancel on client disconnect. <br><br> Resource leaks: ResourceWarning from unclosed memory streams. Task groups from one request interfere with another's. <br><br> Structured teardown violation: The comment at lines 562–567 explains that __aexit__ must run via the exit stack, not via GeneratorExit (PEP 789). Moving async_exit_stack breaks this contract.
D. solved_result (routing.py:473–480)
Dimension	Detail
Why it exists	Contains resolved dependency values (values), validation errors (errors), background tasks (background_tasks), and response configuration (response.headers, response.status_code).
Classification	Request-specific. Depends on request body, headers, path parameters, and dependency resolution.
Current flow	Used directly in app (not captured by nested helpers). Passed to endpoint call (line 514), attached to StreamingResponse (lines 630, 666), headers extended (lines 635, 668, 688, 734).
Proposed flow	Must remain in app scope. No change needed.
Behavioral regression if mishandled	Background task bleed: solved_result.background_tasks attached to StreamingResponse (lines 630, 666). If reused, tasks scheduled for one request run in another's context with wrong data or stale DB connections. <br><br> Header leakage: solved_result.response.headers.raw extended into response (lines 635, 668, 688, 734). Custom headers (auth tokens, request IDs) leak between requests. <br><br> Dependency error cross-contamination: solved_result.errors from one request block another.
E. body (routing.py:419–442)
Dimension	Detail
Why it exists	Parsed request body (FormData, JSON, or raw bytes).
Classification	Request-specific.
Current flow	Passed to solve_dependencies (line 476). Included in RequestValidationError (lines 455, 737). Not captured by nested helpers.
Proposed flow	Must remain in app scope.
Behavioral regression if mishandled	Error reporting with data leakage: Body from one request included in another's validation error. Could leak sensitive data (passwords, tokens) in error responses. <br><br> Form cleanup failure: file_stack.push_async_callback(body.close) at line 423 must remain tied to the request scope.
F. file_stack (routing.py:400–403)
Dimension	Detail
Why it exists	Request-scoped AsyncExitStack from request.scope["fastapi_middleware_astack"]. Manages middleware and file cleanup.
Classification	Request-specific.
Current flow	Used to push body.close callback for form uploads (line 423).
Proposed flow	Must remain in app scope.
Behavioral regression if mishandled	File descriptor leaks: If shared, uploaded files from one request closed while another is reading, or never closed. <br><br> Premature cleanup: Form data from one request cleaned up during another's processing.
2. Route-Static State Currently Captured (Safe to Hoist)
These variables are currently captured by _serialize_data and are route-static — determined at route registration and immutable for the lifetime of the route.
Variable	Factory Parameter	Used In _serialize_data At	Classification
stream_item_field	line 382	line 488	Route-static
response_model_include	line 373	line 501	Route-static
response_model_exclude	line 374	line 502	Route-static
response_model_by_alias	line 375	line 503	Route-static
response_model_exclude_unset	line 376	line 504	Route-static
response_model_exclude_defaults	line 377	line 505	Route-static
response_model_exclude_none	line 378	line 506	Route-static
Behavioral regression if mishandled: None. These are filter/type specifications. Hoisting them to factory scope is purely organizational.
3. How the Proposed Architecture Changes the Flow
Current Architecture (Per-Request Closure Captures)
factory scope: get_request_handler(params...)
  → app(request): creates endpoint_ctx, gen, solved_result, async_exit_stack
      → def _serialize_data(data):        # captures endpoint_ctx + 7 route-static
      → def _serialize_sse_item(item):    # captures _serialize_data
      → def _serialize_item(item):        # captures _serialize_data
      → def _producer():                  # captures sse_aiter + _serialize_sse_item
      → def _async_stream_jsonl():        # captures gen + _serialize_item
      → def _sync_stream_jsonl():         # captures gen + _serialize_item
Per request, 3 helper function objects are created (_serialize_data, _serialize_sse_item, _serialize_item) with 10 total closure cells.
Proposed Architecture (Factory-Scope Helpers + Explicit endpoint_ctx)
factory scope: get_request_handler(params...)
  → def _serialize_data(endpoint_ctx, data):      # captures 7 route-static only
  → def _serialize_sse_item(endpoint_ctx, item):  # captures _serialize_data
  → def _serialize_item(endpoint_ctx, item):      # captures _serialize_data
  → app(request): creates endpoint_ctx, gen, solved_result, async_exit_stack
      → def _producer():                  # captures sse_aiter + _serialize_sse_item + endpoint_ctx
          → calls _serialize_sse_item(endpoint_ctx, raw_item)
      → def _async_stream_jsonl():        # captures gen + _serialize_item + endpoint_ctx
          → calls _serialize_item(endpoint_ctx, item)
      → def _sync_stream_jsonl():         # captures gen + _serialize_item + endpoint_ctx
          → calls _serialize_item(endpoint_ctx, item)
Per request, 0 new helper function objects are created. The per-request streaming generators gain 1 additional closure cell each (+3 cells total). Net: −3 function objects, −7 closure cells per request.
4. Critical Behavioral Regressions to Guard Against
Regression 1: Stale endpoint_ctx in ResponseValidationError
If the factory-scope _serialize_data retains a closure capture of endpoint_ctx instead of receiving it as a parameter, the first request's context is baked into the factory. All subsequent ResponseValidationError messages show the wrong endpoint path.
Mitigation: endpoint_ctx must be an explicit parameter. No closure capture.
Regression 2: Loss of endpoint_ctx Fallback
Current code: ctx = endpoint_ctx or EndpointContext() (line 493). If the explicit parameter default is removed, None endpoint_ctx causes ResponseValidationError with no context.
Mitigation: Factory-scope helper must preserve endpoint_ctx: EndpointContext | None = None and ctx = endpoint_ctx or EndpointContext().
Regression 3: gen / sse_aiter Accidentally Hoisted
If refactoring accidentally moves gen = dependant.call(**solved_result.values) to factory scope, the generator is shared across requests.
Mitigation: gen and sse_aiter must remain inside app. Code review should verify they are not moved.
Regression 4: async_exit_stack Lifecycle Breakage
The SSE task group cancellation depends on async_exit_stack.enter_async_context(_sse_producer_cm()) (line 606) and push_async_callback(sse_receive_stream.aclose) (line 611). These must remain in app scope.
Mitigation: Do not touch the SSE context manager entry or callback registration.
Regression 5: anyio.sleep(0) Checkpoints
Lines 621, 650, 681 are cancellation checkpoints in per-request generators. They must not be moved to factory scope.
Mitigation: Keep checkpoints in _sse_with_checkpoints, _async_stream_jsonl, _async_stream_raw.
Regression 6: Background Task Bleed
solved_result.background_tasks is attached to StreamingResponse at lines 630 and 666. This is per-request and must not be cached.
Mitigation: solved_result remains in app scope. No change.
5. Adjacent Request-Specific State (Not in Closures but Critical)
- response object (line 399): Built incrementally per-request. Mutated with headers, status code, body. Must remain per-request.
- raw_response (line 690): Endpoint function output. Attaching background_tasks at line 697 must remain scoped.
- use_dump_json (line 708): Actually route-static (depends only on response_field and response_class). Currently computed per-request. Could be hoisted to factory scope, but the gain is negligible.
Next Steps
1. Implement factory-scope helpers: Move _serialize_data, _serialize_sse_item, _serialize_item to get_request_handler factory scope with explicit endpoint_ctx parameter.
2. Update call sites: Modify _producer (line 575), _async_stream_jsonl (line 647), _sync_stream_jsonl (line 659) to pass endpoint_ctx explicitly.
3. Preserve fallback: Ensure ctx = endpoint_ctx or EndpointContext() remains in _serialize_data.
4. Verify tests: Run the 8 streaming tests (test_stream_cancellation, test_stream_json_validation_error, test_stream_bare_type) plus the full test suite.
5. Measure: Rerun the allocation micro-benchmark to verify the ~2.5× overhead reduction.