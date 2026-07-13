model: kimi k2.5

## Design

**Problem.** The streaming serialization helpers (`_serialize_data`, `_serialize_sse_item`, `_serialize_item`, `_async_stream_jsonl`, `_sync_stream_jsonl`) are currently defined inside `app()` (`fastapi/routing.py:398`), which means Python creates new function objects on every request. Each `_serialize_data` allocation is ~160 bytes plus closure cells. For high-throughput streaming endpoints this adds gen-0 pressure without functional benefit.

**Observations.** Bytecode inspection confirms:
- Function objects are recreated per-request; `__code__` objects are shared.
- Closure cells have mixed behavior: `endpoint_ctx` (per-request) is recreated, while `stream_item_field` and response-model filters (route-static) are already shared across requests.
- The non-streaming path already follows the same pattern: `serialize_response()` and `run_endpoint_function()` were extracted from `get_request_handler()` for profiling and reuse.

**Selected architecture.** Hoist the five helpers to `get_request_handler()` scope and pass `endpoint_ctx` as an explicit parameter. `_sse_producer_cm`, `_producer`, `_keepalive_inserter`, `_sse_with_checkpoints`, and `_async_stream_raw` remain inside `app()` because they capture per-request state (`sse_aiter`, `gen`) that cannot be threaded practically.

**Why this approach.** It requires only scope changes and one added parameter — zero logic changes, zero new abstractions, zero public API changes. Factory functions, classes, and `functools.partial` were considered but rejected because they add indirection or inconsistency with the existing functional style.

**Behavioural guarantees.**
- `ResponseValidationError` continues to receive the correct `endpoint_ctx` (path, method, endpoint name) because `endpoint_ctx` is still computed inside `app()` and passed explicitly.
- Request lifecycle is unchanged: body reading, `async_exit_stack` cleanup, `file_stack` file closing, dependency resolution order, and background task attachment are untouched.
- Streaming execution is unchanged: generator iteration, `anyio.sleep(0)` checkpoints, keepalive insertion, and SSE formatting remain identical.
- All existing streaming tests pass without modification.