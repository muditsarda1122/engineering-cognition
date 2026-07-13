model: kimi k2.5

## Context Assembly

### Repository Summary
Repository: fastapi — High-performance Python web framework for building APIs with automatic data validation, serialization, dependency injection, and OpenAPI documentation generation.

### Current Session Context
Active session: session-20260711-001 on branch master. Previous work analyzed streaming serialization closure recreation in `get_request_handler()` and recommended hoisting helpers to eliminate per-request allocation (~320 bytes/request).

### Relevant Architecture
FastAPI's streaming pipeline creates helper functions (`_serialize_data`, `_serialize_sse_item`, etc.) inside the per-request `app()` closure. These are recreated on every request. Route-static cells are shared; only `endpoint_ctx` cell is per-request.

### Relevant Decisions
Selected approach: Hoist streaming serialization helpers from `app()` scope to `get_request_handler()` scope (or module level, following `serialize_response` precedent), passing `endpoint_ctx` explicitly. Rejected alternatives: module-level factories, helper classes, functools.partial.

## Implementation

Five module-level streaming serialization functions were created in `fastapi/routing.py`, placed immediately after `run_endpoint_function` (following the established pattern of extracting helpers from `get_request_handler()`):

1. `serialize_stream_data(data, *, stream_item_field, include, exclude, by_alias, exclude_unset, exclude_defaults, exclude_none, endpoint_ctx) -> bytes`
2. `serialize_sse_item(item, *, stream_item_field, include, exclude, by_alias, exclude_unset, exclude_defaults, exclude_none, endpoint_ctx) -> bytes`
3. `serialize_stream_item(item, *, stream_item_field, include, exclude, by_alias, exclude_unset, exclude_defaults, exclude_none, endpoint_ctx) -> bytes`
4. `async_stream_jsonl(gen, *, stream_item_field, include, exclude, by_alias, exclude_unset, exclude_defaults, exclude_none, endpoint_ctx) -> AsyncIterator[bytes]`
5. `sync_stream_jsonl(gen, *, stream_item_field, include, exclude, by_alias, exclude_unset, exclude_defaults, exclude_none, endpoint_ctx) -> Iterator[bytes]`

### Design Rationale
- All route-static parameters are explicit keyword-only arguments (matching `serialize_response` signature style)
- `endpoint_ctx` is passed explicitly rather than captured from closure scope
- `serialize_stream_data` is the core function; `serialize_sse_item` and `serialize_stream_item` are thin wrappers
- `async_stream_jsonl` and `sync_stream_jsonl` consume the generator and yield serialized lines, preserving the `anyio.sleep(0)` cancellation checkpoint

### Verification
- Module imports successfully: `import fastapi.routing` passes
- All streaming tests pass: 32/32 (test_stream_cancellation.py, test_stream_json_validation_error.py, test_stream_bare_type.py, test_sse.py)
- New functions are importable individually but NOT yet wired into the `app()` closure
- Zero public API changes, zero changes to other modules

### Status
Foundational component complete. Ready for future integration into `get_request_handler()` request execution path.
