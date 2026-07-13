model: kimi k2.5

Done. I integrated the factory-scope abstraction into the SSE (Server-Sent Events) execution path — the most behaviourally complex streaming branch in get_request_handler. Three per-request closures were hoisted to factory scope: _serialize_sse_item and _sse_with_checkpoints (newly hoisted), on top of the previously hoisted _serialize_data. All 72 streaming tests pass.
1. Request-specific state that must remain dynamic
Three categories of state must stay per-request and cannot be captured at factory time:
a) endpoint_ctx — Extracted inside app(request) from dependant.call and request.scope (lines 474–481 in the current file). It contains the request method, path, source file, and line number. Because every request hits a potentially different URL with different path parameters, this must be passed explicitly into the factory-scope helpers.
b) gen / sse_aiter — The generator instance produced by dependant.call(**solved_result.values) (line 560). This is inherently request-scoped: each request gets its own generator object that yields items from the endpoint function. sse_aiter wraps this generator for async iteration. These are consumed by _producer inside _sse_producer_cm, which remains a per-request context manager because it closes over sse_aiter.
c) sse_receive_stream — The receive end of the memory object stream created inside _sse_producer_cm (line 625). It is request-scoped because it carries the SSE event bytes for this specific HTTP response. It is entered on the request-scoped async_exit_stack so structured teardown (task group cancellation) runs when the request completes.
The factory-scope helpers (_serialize_data, _serialize_sse_item, _sse_with_checkpoints) are pure functions: they take all request-specific data as explicit parameters and mutate nothing.
2. How lifecycle management is preserved
The SSE path has a carefully designed lifecycle involving multiple concurrent tasks and context managers. Every aspect is preserved:
Task group lifecycle — _sse_producer_cm still creates an anyio.create_task_group() (line 615) and starts _producer and _keepalive_inserter. The tg.cancel_scope.cancel() on yield-exit still cancels both tasks. The factory-scope _serialize_sse_item is called from inside _producer, which runs inside the task group. Because _producer is defined inside _sse_producer_cm which is defined inside app(), the Python closure chain correctly resolves endpoint_ctx from app()'s scope — I verified this with an isolated test before editing.
Context-manager stack lifecycle — sse_receive_stream is still entered on async_exit_stack (line 625), which is the request-scoped fastapi_inner_astack from request_response(). The push_async_callback(sse_receive_stream.aclose) (line 629) still runs on stack unwind, preventing ResourceWarning. The factory-scope change does not alter when or how these enter/exit calls happen.
Generator lifecycle — gen is still created per-request and consumed by sse_aiter. The _producer coroutine pulls from sse_aiter and sends serialized bytes to send_stream. Because _serialize_sse_item is now a pure function called synchronously inside the async loop, it cannot suspend or alter the generator's iteration schedule.
3. Additional engineering challenges compared to the previous step
The JSONL integration was straightforward: _serialize_item was a one-line wrapper that called _serialize_data. The SSE path introduced three additional challenges:
a) Multi-type dispatch in _serialize_sse_item — Unlike JSONL, which always serializes the same way, SSE items can be either ServerSentEvent objects (transport wrappers with raw data) or plain objects (validated against stream_item_field). The factory-scope helper must preserve this branch:
- For ServerSentEvent: skip validation, extract raw_data or serialize data manually with model_dump_json() or json.dumps(jsonable_encoder(...))
- For plain objects: delegate to _serialize_data(endpoint_ctx, item) and wrap in format_sse_event
This means the factory-scope helper has more complex internal logic than _serialize_data, but all its dependencies (ServerSentEvent, format_sse_event, jsonable_encoder, json, _serialize_data) are available at factory scope.
b) Closure scope chain through @asynccontextmanager — The call to _serialize_sse_item happens inside _producer, which is defined inside _sse_producer_cm, which is decorated with @asynccontextmanager. I had to verify that Python preserves the closure chain through the decorator so _producer can access endpoint_ctx from the enclosing app() scope. An isolated test confirmed this works, but it was a necessary investigation step that the JSONL path did not require (JSONL calls _serialize_data directly inside app(), not inside a nested context manager).
c) _sse_with_checkpoints is an async generator — Hoisting an async def that yields to factory scope is slightly different from hoisting a regular def because the async generator object must be created per-iteration. However, since _sse_with_checkpoints takes stream as a parameter and captures nothing else, it behaves like a pure coroutine factory. Hoisting it to factory scope is safe because the async generator object is instantiated per-request when called (_sse_with_checkpoints(sse_receive_stream)), not reused across requests.
4. How behavioural equivalence is maintained
Serialization output is byte-identical:
- _serialize_sse_item uses the same isinstance(item, ServerSentEvent) check
- Same raw_data / data extraction logic with model_dump_json() or jsonable_encoder
- Same format_sse_event call with identical keyword arguments
- For plain objects: calls the factory-scope _serialize_data, which has identical logic to the removed per-request version
Checkpoint behaviour is identical:
- _sse_with_checkpoints yields each byte chunk and then await anyio.sleep(0) — exactly the same two lines as before
- The function is still an async iterator wrapper around ObjectReceiveStream[bytes]
Error handling is preserved:
- endpoint_ctx is still passed through to _serialize_data, which creates ResponseValidationError with the same ctx = endpoint_ctx or EndpointContext() fallback
- If stream_item_field validation fails, the same errors_ are raised with the same body=data
Verification: 72 streaming tests pass (test_sse.py, test_stream_cancellation.py, test_stream_bare_type.py, test_stream_json_validation_error.py, test_dependency_after_yield_streaming.py, test_dependency_yield_scope.py, test_dependency_contextmanager.py), including SSE keepalive, mixed plain/SSE events, cancellation under asyncio and trio, JSON validation errors, and dependency cleanup after streaming.
What remains unchanged for later steps
1. _sse_producer_cm and its inner _producer / _keepalive_inserter — These close over sse_aiter and the local memory streams, which are fundamentally request-scoped. Hoisting them would require turning them into parameterized factories, which adds complexity without matching the abstraction's payoff.
2. _serialize_item in the JSONL path — This is a trivial one-line wrapper (_serialize_data(endpoint_ctx, item) + b"\n"). It could be hoisted, but it is defined inside app() because the JSONL stream wrappers (_async_stream_jsonl, _sync_stream_jsonl) close over it. A future step could refactor the JSONL wrappers into parameterized factories.
3. Raw streaming path — Still bypasses all serializers and yields chunks directly.
4. Non-streaming serialize_response() — Still uses the standalone async function. This is the largest remaining refactor because it involves the dump_json fast path and threadpool dispatch.