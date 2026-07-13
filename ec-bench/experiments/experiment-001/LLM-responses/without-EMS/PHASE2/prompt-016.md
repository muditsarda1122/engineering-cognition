model: kimi k2.5

Done. I integrated the factory-scope _serialize_data abstraction into the streaming response path (SSE and JSONL).
Why this integration point first
The streaming serialization path (SSE + JSONL) is the least risky because:
- Smallest blast radius: It only affects generator endpoints that return AsyncIterator/Iterator with response_class=EventSourceResponse or the default JSONL placeholder. Regular coroutine endpoints that return concrete objects are completely untouched.
- Behaviourally identical: The factory-scope function has the exact same validation and serialization logic as the per-request closure. The only difference is when the function object is created (once at route registration vs. once per request).
- No API surface change: This is a pure internal refactor. End users see no difference in response format, validation errors, or error messages.
- Easy to verify: The existing SSE and streaming test suite validates the end-to-end behaviour, and all 39 relevant tests passed.
What existing code is replaced or modified
Replaced: The per-request _serialize_data inner function defined inside app() (lines 516–542) was removed. This function was recreated on every single request to any streaming endpoint.
Modified: Two call sites inside app() were updated to use the factory-scope helper and pass endpoint_ctx explicitly:
1. _serialize_sse_item() — now calls _serialize_data(endpoint_ctx, item)
2. _serialize_item() (JSONL) — now calls _serialize_data(endpoint_ctx, item)
What was already in place (from the previous session): The factory-scope _serialize_data was already defined at lines 398–428 inside get_request_handler, outside the app() coroutine. It takes endpoint_ctx as an explicit parameter instead of closing over it.
How existing behaviour is preserved
- Same validation logic: stream_item_field.validate(...) with identical loc=("response",) and error handling.
- Same serialization parameters: include, exclude, by_alias, exclude_unset, exclude_defaults, exclude_none are all forwarded identically.
- Same error messages: endpoint_ctx is still passed into ResponseValidationError, preserving request-specific path info in error output.
- Same fallback: When stream_item_field is absent, it still falls back to jsonable_encoder + json.dumps.
- Tests confirm: All SSE, streaming cancellation, bare type, JSON validation error, and dependency-after-yield streaming tests pass.
What remains intentionally unchanged for later steps
1. serialize_response() for non-streaming responses: The coroutine endpoint path (line 743) still uses the standalone serialize_response() async function. This is a larger refactor because it involves async/await semantics and the dump_json fast path. It should be tackled separately after proving the streaming abstraction is stable.
2. _serialize_sse_item() itself: This function is still defined per-request inside app(). It handles ServerSentEvent transport wrapping and SSE-specific formatting (format_sse_event). It could potentially be hoisted to factory scope later, but it has more complex closure dependencies (e.g., format_sse_event constants) and mixed-type logic, so it's left for a second pass.
3. Raw streaming path: The StreamingResponse branch for custom response_class (line 701) doesn't use _serialize_data at all, so it's unaffected and doesn't need changes.
4. WebSocket handling: get_websocket_app() has its own dependency solving and error handling flow. It shares no serialization code with the HTTP response path, so it's out of scope for this abstraction.