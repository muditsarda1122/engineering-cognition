model: kimi k2.5

## Context Assembly

### Repository Summary
Repository: fastapi — High-performance Python web framework for building APIs with automatic data validation, serialization, dependency injection, and OpenAPI documentation generation.

### Current Session Context
Active session: session-20260711-001 on branch master. Previous work created five module-level streaming serialization functions and integrated the JSONL path. The SSE path remained on the legacy closure-based architecture.

### Relevant Architecture
FastAPI's SSE streaming pipeline involves a context manager (`_sse_producer_cm`), task groups, memory streams, a keepalive inserter, and a checkpoint wrapper. The `_producer` task calls `_serialize_sse_item` which in turn calls `_serialize_data`. Both `_serialize_data` and `_serialize_sse_item` were per-request closures.

### Relevant Decisions
Selected approach: Hoist streaming serialization helpers from `app()` scope to module level, passing `endpoint_ctx` explicitly. The SSE path was identified as the most complex due to its nested closure hierarchy and task group integration.

---

## SSE Path Integration

### Integration Point

The SSE streaming path (`if is_sse_stream:` block, routing.py:662) is the most behaviorally complex streaming execution path. It was integrated after the JSONL path validated successfully (32/32 tests pass).

### Request-Specific State That Must Remain Dynamic

The following closures capture per-request state and **must remain** as closures:

1. **`_sse_producer_cm` context manager** — Creates per-request `anyio` memory streams (`send_stream`/`receive_stream`, `send_keepalive`/`receive_keepalive`) and a task group. Cannot be hoisted because `sse_aiter` is per-request.

2. **`_producer` closure** — Must capture `send_stream` (created inside `_sse_producer_cm`) and `sse_aiter` (computed at request time based on whether the generator is async or sync). Modified to call module-level `serialize_sse_item(...)` explicitly instead of `_serialize_sse_item(raw_item)`.

3. **`_keepalive_inserter` closure** — Must capture `send_keepalive` and `receive_stream` (both created inside `_sse_producer_cm`). Reads from producer stream and inserts keepalive comments on timeout via `anyio.fail_after(_PING_INTERVAL)`.

4. **`_sse_with_checkpoints` closure** — Must capture `sse_receive_stream` (entered on the request-scoped `AsyncExitStack`). Guarantees cancellation checkpoints via `anyio.sleep(0)`.

### Lifecycle Management Preservation

All lifecycle management is preserved exactly:

- **`_sse_producer_cm`** is still entered on the request-scoped `async_exit_stack` (line 739)
- **Task group cancellation** is still triggered by `tg.cancel_scope.cancel()` when the context manager exits
- **Receive stream cleanup** is still pushed as an async callback (`sse_receive_stream.aclose`, line 744)
- **Structured teardown** via `AsyncExitStack` is preserved, avoiding `GeneratorExit` thrown into async generators
- **Keepalive mechanism** is preserved: `anyio.fail_after(_PING_INTERVAL)` triggers `KEEPALIVE_COMMENT` on timeout

### Additional Engineering Challenges vs JSONL Integration

1. **Nested closure depth:** The serialization call is inside `_producer` (line 693) which is inside `_sse_producer_cm` (line 671) which is inside `app()`. The module-level `serialize_sse_item` call must thread through three closure layers with explicit parameters.

2. **Async task group context:** The serialization happens inside a producer task running in `anyio.create_task_group()`. The `serialize_sse_item` call must not introduce new async boundaries that would interfere with the task group's cancellation semantics. The function is called synchronously (it returns bytes), so no async boundary is introduced.

3. **Per-request stream objects:** `_producer` captures `send_stream` and `sse_aiter` which are created at request time. These cannot be hoisted because they depend on the specific generator instance for this request.

4. **Type normalization of `gen`:** The `gen` variable has different types depending on `dependant.is_async_gen_callable` (AsyncIterator vs Iterator), and `sse_aiter` normalizes this. The module-level function receives the already-normalized `sse_aiter` through `_producer`'s closure capture.

### Behavioral Equivalence

- **Logic equivalence:** `serialize_sse_item` contains identical logic to the removed `_serialize_sse_item` closure — same `ServerSentEvent` branch (raw_data / data / model_dump_json fallback), same plain-object branch calling `serialize_stream_data`.
- **Parameter values:** Explicit parameters receive the same values previously captured as closure cells.
- **Test verification:** All 32 streaming tests pass (test_stream_cancellation, test_stream_json_validation_error, test_stream_bare_type, test_sse), confirming behavioral preservation across all streaming paths.
- **Cancellation checkpoints:** `anyio.sleep(0)` is preserved in `_sse_with_checkpoints` (line 754).
- **Error handling:** `ResponseValidationError` construction with `endpoint_ctx` is preserved through `serialize_stream_data`.

### Code Replaced

**Removed:**
- `_serialize_data` closure (previously lines 665-688) — eliminated entirely
- `_serialize_sse_item` closure (previously lines 694-721) — eliminated entirely

**Modified:**
- `_producer` closure — updated to call `serialize_sse_item(raw_item, stream_item_field=..., include=..., exclude=..., by_alias=..., exclude_unset=..., exclude_defaults=..., exclude_none=..., endpoint_ctx=...)` instead of `_serialize_sse_item(raw_item)`

**Preserved:**
- `_sse_producer_cm`, `_keepalive_inserter`, `_sse_with_checkpoints` — all unchanged

### Status
Both JSONL and SSE paths are now fully integrated with the module-level streaming serialization abstraction. All per-request serialization closures (`_serialize_data`, `_serialize_item`, `_serialize_sse_item`, `_async_stream_jsonl`, `_sync_stream_jsonl`) have been eliminated. Remaining per-request closures (`_producer`, `_keepalive_inserter`, `_sse_with_checkpoints`, `_sse_producer_cm`) capture only genuinely per-request state (streams, iterators, task groups).
