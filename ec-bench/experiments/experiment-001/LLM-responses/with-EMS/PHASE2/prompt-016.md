model: kimi k2.5

## Context Assembly

### Repository Summary
Repository: fastapi — High-performance Python web framework for building APIs with automatic data validation, serialization, dependency injection, and OpenAPI documentation generation.

### Current Session Context
Active session: session-20260711-001 on branch master. Previous work analyzed streaming serialization closure recreation in `get_request_handler()` and created five module-level streaming serialization functions (`serialize_stream_data`, `serialize_sse_item`, `serialize_stream_item`, `async_stream_jsonl`, `sync_stream_jsonl`). These functions were verified as importable but not yet integrated into the request execution path.

### Relevant Architecture
FastAPI's streaming pipeline creates helper functions inside the per-request `app()` closure. The JSONL path uses `_serialize_item` → `_serialize_data` and `_async_stream_jsonl`/`_sync_stream_jsonl`. The SSE path is more complex with `_sse_producer_cm` context manager, task groups, and keepalive inserter.

### Relevant Decisions
Selected approach: Hoist streaming serialization helpers from `app()` scope to module level, passing `endpoint_ctx` explicitly. The four-phase validation strategy recommends starting with the lowest-risk path for incremental validation.

---

## Integration

### Why the JSONL path was selected first

The JSONL streaming path (`elif is_json_stream:`) is the **least risky** integration point for three reasons:

1. **Simplicity:** The JSONL path has no context managers, no task groups, and no keepalive inserter. It consists of straightforward generator iteration with serialization.
2. **Limited blast radius:** Only three closures are involved (`_serialize_item`, `_async_stream_jsonl`, `_sync_stream_jsonl`), compared to the SSE path which involves `_serialize_sse_item`, `_sse_producer_cm`, `_producer`, `_keepalive_inserter`, and `_sse_with_checkpoints`.
3. **Test coverage:** JSONL paths are well-covered by existing tests (cancellation safety, validation error propagation, bare type streaming), providing immediate validation feedback.

The SSE path remains completely untouched, acting as a control for behavioral comparison.

### What existing code was replaced or modified

**In `fastapi/routing.py` (inside `app()` closure, `elif is_json_stream:` block):**

**Removed:**
- `_serialize_item` closure definition (line 818)
- `_async_stream_jsonl` closure definition (line 823)
- `_sync_stream_jsonl` closure definition (line 835)

**Replaced with:**
- `async_stream_jsonl(gen, stream_item_field=..., include=..., exclude=..., by_alias=..., exclude_unset=..., exclude_defaults=..., exclude_none=..., endpoint_ctx=...)` call
- `sync_stream_jsonl(gen, stream_item_field=..., include=..., exclude=..., by_alias=..., exclude_unset=..., exclude_defaults=..., exclude_none=..., endpoint_ctx=...)` call

All route-static parameters (`stream_item_field`, `response_model_include`, `exclude`, `by_alias`, `exclude_unset`, `exclude_defaults`, `exclude_none`) and the per-request `endpoint_ctx` are now passed explicitly rather than captured from closure scope.

### How existing behaviour is preserved

1. **Logic equivalence:** The module-level `async_stream_jsonl` and `sync_stream_jsonl` contain identical logic to the removed closures — same generator iteration, same `serialize_stream_item` call chain, same `anyio.sleep(0)` cancellation checkpoint.
2. **Parameter values:** The explicit parameters receive the same values that were previously captured as closure cells (`stream_item_field` from `get_request_handler()` scope, `endpoint_ctx` from `app()` scope).
3. **Type annotations:** The `jsonl_stream_content` type annotation pattern (`AsyncIterator[bytes] | Iterator[bytes]`) is preserved.
4. **Test verification:** All 32 streaming tests pass (test_stream_cancellation.py, test_stream_json_validation_error.py, test_stream_bare_type.py, test_sse.py), confirming behavioral preservation across both JSONL and SSE paths.

### What remains intentionally unchanged for later implementation steps

**SSE path (completely untouched):**
- `_serialize_data` closure (line 665) — still recreated per-request, still used by `_serialize_sse_item`
- `_serialize_sse_item` closure (line 694) — still recreated per-request
- `_sse_producer_cm` context manager (line 729) — still recreated per-request
- `_sse_with_checkpoints` wrapper (line 752) — still recreated per-request
- All SSE machinery (task group, keepalive inserter, memory streams)

**Future work:**
1. Replace `_serialize_sse_item` closure with module-level `serialize_sse_item` call
2. Update `_sse_producer_cm` to call `serialize_sse_item` explicitly
3. Remove `_serialize_data` closure once no longer referenced by SSE path
4. Verify per-request allocation reduction for both JSONL and SSE requests
5. Add streaming benchmarks to quantify performance improvement

### Status
JSONL path is now fully integrated with the module-level streaming serialization abstraction. SSE path remains on the legacy closure-based architecture, ready for migration in the next incremental step.
