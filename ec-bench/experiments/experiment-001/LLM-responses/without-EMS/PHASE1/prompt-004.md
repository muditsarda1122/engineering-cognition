model: kimi k2.5

Here is the complete inventory of all 10 nested helpers inside async def app(request) in get_request_handler (routing.py:398-745).
1. _serialize_data
- Location: routing.py:487, at the top of the if not errors: block (before any streaming branch).
- Type: def
- Responsibility: Serialize a single data item to JSON bytes. If stream_item_field is set, validates via Pydantic and calls serialize_json (the Rust fast path). Otherwise falls back to jsonable_encoder + json.dumps.
- Created: On every request that passes dependency/validation checks (if not errors:).
- Captures from enclosing scope:
- stream_item_field, response_model_include, response_model_exclude, response_model_by_alias, response_model_exclude_unset, response_model_exclude_defaults, response_model_exclude_none — from get_request_handler’s outer scope (setup-time parameters).
- endpoint_ctx — from app’s local scope (computed at line 406).
2. _serialize_sse_item
- Location: routing.py:516, inside the if is_sse_stream: branch.
- Type: def
- Responsibility: Serialize one item for SSE output. Handles ServerSentEvent instances specially (extracts raw data or serializes the .data payload) versus plain objects (delegates to _serialize_data).
- Created: Only on SSE streaming execution paths (is_sse_stream is true for this route).
- Captures from enclosing scope:
- _serialize_data — from app’s local scope (defined at line 487).
3. _sse_producer_cm
- Location: routing.py:551, inside the if is_sse_stream: branch.
- Type: @asynccontextmanager
- Responsibility: Context manager that creates two anyio memory streams, starts a task group with _producer and _keepalive_inserter, and yields the consumer-side stream. Its __aexit__ cancels the task group.
- Created: Only on SSE streaming execution paths.
- Captures from enclosing scope:
- sse_aiter — from the if is_sse_stream: block in app (line 546/548).
- _serialize_sse_item — from the if is_sse_stream: block in app (line 516).
4. _producer
- Location: routing.py:572, nested inside _sse_producer_cm.
- Type: async def
- Responsibility: Pulls items from sse_aiter (the endpoint generator) and pushes serialized bytes into send_stream.
- Created: Only on SSE streaming execution paths (inside _sse_producer_cm).
- Captures from enclosing scope:
- send_stream, receive_stream — from _sse_producer_cm’s local scope (lines 568-570).
- sse_aiter, _serialize_sse_item — from the if is_sse_stream: block in app (accessible via _sse_producer_cm’s closure chain).
5. _keepalive_inserter
- Location: routing.py:581, nested inside _sse_producer_cm.
- Type: async def
- Responsibility: Reads from the producer stream and forwards to the output stream, inserting SSE keepalive comments when anyio.fail_after(_PING_INTERVAL) times out.
- Created: Only on SSE streaming execution paths (inside _sse_producer_cm).
- Captures from enclosing scope:
- send_keepalive, receive_keepalive, receive_stream — from _sse_producer_cm’s local scope (lines 577-570).
- _PING_INTERVAL, KEEPALIVE_COMMENT — module-level imports from fastapi.sse (not from app’s scope, but still non-local to this helper).
6. _sse_with_checkpoints
- Location: routing.py:613, inside the if is_sse_stream: branch.
- Type: async def
- Responsibility: Wraps the SSE consumer stream and yields items with await anyio.sleep(0) after each one, guaranteeing ASGI cancellation checkpoints even when the producer is faster than the consumer.
- Created: Only on SSE streaming execution paths.
- Captures from enclosing scope:
- None from app or get_request_handler. Its parameter stream is passed explicitly at line 624.
7. _serialize_item
- Location: routing.py:640, inside the elif is_json_stream: branch.
- Type: def
- Responsibility: Serialize a single JSONL item by calling _serialize_data and appending a newline byte.
- Created: Only on JSONL streaming execution paths (is_json_stream is true for this route).
- Captures from enclosing scope:
- _serialize_data — from app’s local scope (defined at line 487).
8. _async_stream_jsonl
- Location: routing.py:645, inside the elif is_json_stream: branch, under if dependant.is_async_gen_callable:.
- Type: async def
- Responsibility: Async generator that pulls items from the endpoint generator and yields _serialize_item(item) with await anyio.sleep(0) checkpoints.
- Created: Only on async JSONL streaming execution paths.
- Captures from enclosing scope:
- gen — from the elif is_json_stream: block in app (line 638).
- _serialize_item — from the elif is_json_stream: block in app (line 640).
9. _sync_stream_jsonl
- Location: routing.py:657, inside the elif is_json_stream: branch, under the else: (sync generator) path.
- Type: def
- Responsibility: Sync generator that pulls items from the endpoint generator and yields _serialize_item(item).
- Created: Only on sync JSONL streaming execution paths.
- Captures from enclosing scope:
- gen — from the elif is_json_stream: block in app (line 638).
- _serialize_item — from the elif is_json_stream: block in app (line 640).
10. _async_stream_raw
- Location: routing.py:674, inside the elif dependant.is_async_gen_callable or dependant.is_gen_callable: branch, under if dependant.is_async_gen_callable:.
- Type: async def
- Responsibility: Wraps a raw async generator with await anyio.sleep(0) checkpoints to allow cancellation delivery.
- Created: Only on async raw streaming execution paths (when is_sse_stream and is_json_stream are both false, but the endpoint returns an async generator).
- Captures from enclosing scope:
- None from app or get_request_handler. Its parameter async_gen is passed explicitly at line 683.
Key observation about closure depth
_producer and _keepalive_inserter are nested two levels deep (inside _sse_producer_cm, which is inside app). Their immediate enclosing scope is _sse_producer_cm, but through the closure chain they also access sse_aiter and _serialize_sse_item from app’s scope. This pattern — where the SSE producer context manager captures route-specific helpers and streams, and its inner coroutines capture those in turn — is what allows the entire SSE pipeline to be pre-bound at import time while still being instantiated fresh per request.