model: kimi k2.5

Nested Helper Inventory Inside get_request_handler
Inside async def app(request: Request) -> Response (routing.py:398), there are 10 nested helpers defined via either def or @asynccontextmanager. Here is the complete inventory in order of appearance.
1. _serialize_data
- Location: line 487, inside if not errors: (the success path)
- Created: on every successful request (any endpoint that passes validation)
- Responsibility: Shared serializer for both JSONL and SSE streaming items. If stream_item_field is set, validates the item against it and serializes to JSON bytes via Pydantic v2 (serialize_json). Otherwise falls back to jsonable_encoder + json.dumps.
- Captured from outer scope: stream_item_field, response_model_include, response_model_exclude, response_model_by_alias, response_model_exclude_unset, response_model_exclude_defaults, response_model_exclude_none, endpoint_ctx
2. _serialize_sse_item
- Location: line 516, inside if is_sse_stream: block
- Created: only on requests to SSE streaming endpoints
- Responsibility: Serializes a single item for Server-Sent Events output. If the item is a ServerSentEvent instance, formats its fields directly; otherwise validates via _serialize_data and wraps in format_sse_event().
- Captured from outer scope: _serialize_data (helper #1), endpoint_ctx
3. _sse_producer_cm
- Location: line 550 (decorated @asynccontextmanager), inside if is_sse_stream: block
- Created: only on requests to SSE streaming endpoints
- Responsibility: Async context manager that sets up a producer/consumer pipeline for SSE streaming. Creates memory object streams, spawns a task group with _producer and _keepalive_inserter, and yields the receive stream. Entered on the request-scoped AsyncExitStack for proper structured teardown.
- Captured from outer scope: sse_aiter (the iterator over the generator endpoint), _PING_INTERVAL, KEEPALIVE_COMMENT, _serialize_sse_item (helper #2)
4. _producer (nested inside _sse_producer_cm)
- Location: line 572, inside _sse_producer_cm()
- Created: only when _sse_producer_cm() is entered, which only happens for SSE streaming endpoints
- Responsibility: Pulls items from the generator iterator (sse_aiter) and sends them through the memory stream after serializing via _serialize_sse_item.
- Captured from outer scope: send_stream (local to _sse_producer_cm), sse_aiter, _serialize_sse_item (helper #2)
5. _keepalive_inserter (nested inside _sse_producer_cm)
- Location: line 581, inside _sse_producer_cm()
- Created: only when _sse_producer_cm() is entered, which only happens for SSE streaming endpoints
- Responsibility: Reads from the producer stream and forwards to the output stream, inserting keepalive comment messages on a _PING_INTERVAL timeout to prevent connection idle timeouts.
- Captured from outer scope: send_keepalive, receive_stream (both locals to _sse_producer_cm), _PING_INTERVAL, KEEPALIVE_COMMENT
6. _sse_with_checkpoints
- Location: line 613, inside if is_sse_stream: block
- Created: only on requests to SSE streaming endpoints
- Responsibility: Wraps the SSE receive stream with await anyio.sleep(0) checkpoints between items so that cancellation signals can be delivered even when the producer is faster than the consumer.
- Captured from outer scope: none (takes stream as a parameter)
7. _serialize_item
- Location: line 640, inside elif is_json_stream: block
- Created: only on requests to JSONL streaming endpoints
- Responsibility: Serializes a single item for JSONL output by calling _serialize_data(item) and appending a newline (b"\n").
- Captured from outer scope: _serialize_data (helper #1)
8. _async_stream_jsonl
- Location: line 645, inside if dependant.is_async_gen_callable: within elif is_json_stream:
- Created: only on requests to async JSONL streaming endpoints
- Responsibility: Async generator wrapper that yields JSONL-encoded items from the endpoint generator, with await anyio.sleep(0) checkpoints for cancellation.
- Captured from outer scope: gen (the generator returned by dependant.call(**solved_result.values)), _serialize_item (helper #7)
9. _sync_stream_jsonl
- Location: line 657, inside the else: within elif is_json_stream:
- Created: only on requests to sync JSONL streaming endpoints
- Responsibility: Sync generator wrapper that yields JSONL-encoded items from the endpoint generator.
- Captured from outer scope: gen (the generator returned by dependant.call(**solved_result.values)), _serialize_item (helper #7)
10. _async_stream_raw
- Location: line 674, inside if dependant.is_async_gen_callable: within elif dependant.is_async_gen_callable or dependant.is_gen_callable:
- Created: only on requests to async raw streaming endpoints (e.g., endpoints returning StreamingResponse explicitly)
- Responsibility: Wraps an async generator with await anyio.sleep(0) checkpoints between chunks so cancellation signals can be delivered.
- Captured from outer scope: none (takes async_gen as a parameter)
Summary Table
#	Name	Location	Created On	Captures
1	_serialize_data	487, if not errors:	Every success path	stream_item_field, response filters, endpoint_ctx
2	_serialize_sse_item	516, if is_sse_stream:	SSE only	_serialize_data, endpoint_ctx
3	_sse_producer_cm	550, if is_sse_stream:	SSE only	sse_aiter, _serialize_sse_item, ping constants
4	_producer	572, inside _sse_producer_cm	SSE only	send_stream, sse_aiter, _serialize_sse_item
5	_keepalive_inserter	581, inside _sse_producer_cm	SSE only	send_keepalive, receive_stream, ping constants
6	_sse_with_checkpoints	613, if is_sse_stream:	SSE only	none
7	_serialize_item	640, elif is_json_stream:	JSONL only	_serialize_data
8	_async_stream_jsonl	645, async JSONL branch	Async JSONL only	gen, _serialize_item
9	_sync_stream_jsonl	657, sync JSONL branch	Sync JSONL only	gen, _serialize_item
10	_async_stream_raw	674, async raw branch	Async raw streaming only	none
Key Observations
1. Closure-within-closure: _producer and _keepalive_inserter are nested inside _sse_producer_cm, creating a three-level nesting (factory → app() → _sse_producer_cm() → _producer/_keepalive_inserter).
2. Two helpers are universal: Only _serialize_data is created on every successful request. All other 9 helpers are conditionally created based on endpoint type (SSE, JSONL, raw streaming).
3. The success-path gate: All helpers live inside if not errors: (line 483). If solve_dependencies() returns validation errors, none of these helpers are defined — the function jumps straight to raising RequestValidationError at line 736.
4. Reuse chain: _serialize_sse_item calls _serialize_data; _serialize_item calls _serialize_data; _producer calls _serialize_sse_item. This creates a small call graph of nested closures where the innermost serializer is shared across all streaming paths.