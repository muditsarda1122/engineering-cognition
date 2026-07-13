model: kimi k2.5

## Design

`get_request_handler` (`fastapi/routing.py:367`) defines 10 nested helpers inside the per-request `app` closure. Three of them — `_serialize_data`, `_serialize_sse_item`, and `_serialize_item` — capture only route-static serialization parameters (7 filter fields plus `stream_item_field`) plus `endpoint_ctx`, which flows indirectly through `_serialize_data`. On every streaming request these helpers are recreated, producing 3 function objects and 10 closure cells unnecessarily.

The non-streaming path already avoids this: `serialize_response` (routing.py:293) is defined outside the closure and receives `endpoint_ctx` explicitly. We apply the same pattern to the streaming path.

### Change

Move `_serialize_data`, `_serialize_sse_item`, and `_serialize_item` to factory scope (inside `get_request_handler`, outside `app`). They retain only route-static captures. `endpoint_ctx` becomes an explicit parameter. The per-request streaming generators (`_producer`, `_async_stream_jsonl`, `_sync_stream_jsonl`) capture `endpoint_ctx` from `app` scope and forward it at call time.

Net: −3 function objects and −7 closure cells per streaming request.

### Why this approach

Module-level extraction was rejected: it would expose route-specific serialization internals at module scope and require passing all 7 filter parameters on every call. A closure wrapper class was rejected: it adds indirection without measurable benefit over explicit parameter passing. Factory scope preserves locality while matching the existing non-streaming pattern.

### Guarantees

- `endpoint_ctx` remains request-specific: path construction from `request.scope["root_path"]` stays in `app`. `ResponseValidationError` messages show the correct endpoint.
- `gen` and `sse_aiter` are not hoisted: they stay per-request to prevent generator sharing and data leakage.
- `async_exit_stack` lifecycle is untouched: SSE context manager entry (line 606) and aclose callback (line 611) remain in `app`, preserving PEP 789 structured teardown.
- Cancellation checkpoints (`anyio.sleep(0)` at lines 621, 650, 681) are not moved.
- No test changes are required.