model: kimi k2.5

## Integration Points for the Hoist Architecture

### Scope of the Change

Approach 1 (hoist `_serialize_data` and `_serialize_sse_item` to `get_request_handler()` scope) is a **structural change** within `fastapi/routing.py`. It does not alter any public API, any external interface, or any module-level function signatures. The only change is moving two nested function definitions from inside `app()` to inside `get_request_handler()`, and adding `endpoint_ctx` as an explicit parameter where it was previously captured from `app()`'s local scope.

Because the helpers are private (nested, not exported), the blast radius is extremely small. No other module in the repository imports or references these functions.

---

## Files Requiring Changes

### 1. `fastapi/routing.py` — PRIMARY CHANGE

**Lines affected:** 483–688 (the streaming response path inside `app()`), plus the new function definitions at `get_request_handler()` scope.

**What must change:**

#### a) Move `_serialize_data` from inside `app()` to inside `get_request_handler()` (lines 487–510)

**Why it must change:** This is the core of the architecture. The function is currently defined inside `app()` on line 487, which means Python recreates the function object on every request. Moving it to `get_request_handler()` scope (before `async def app`) eliminates this per-request allocation.

**Responsibility today:** Serializes a single data item to JSON bytes. If `stream_item_field` is set, validates against the Pydantic model and calls `serialize_json()` (Pydantic v2 Rust fast path). Otherwise falls back to `jsonable_encoder()` + `json.dumps()`.

**How the new architecture interacts:** The function is created once at route setup time, not per-request. It captures the 7 route-static variables (`stream_item_field`, response model filters) from `get_request_handler()`'s scope via closure, but receives `endpoint_ctx` as an explicit parameter instead of capturing it from `app()`.

**Change type:** STRUCTURAL. No logic changes, only scope change and parameter addition.

**Specific code change:**
```python
# BEFORE (line 487, inside app()):
def _serialize_data(data: Any) -> bytes:
    if stream_item_field:
        value, errors_ = stream_item_field.validate(data, {}, loc=("response",))
        if errors_:
            ctx = endpoint_ctx or EndpointContext()  # captured from app() local scope
            raise ResponseValidationError(errors=errors_, body=data, endpoint_ctx=ctx)
        # ... serialize_json call ...

# AFTER (inside get_request_handler(), before app()):
def _serialize_data(data: Any, endpoint_ctx: EndpointContext) -> bytes:
    if stream_item_field:
        value, errors_ = stream_item_field.validate(data, {}, loc=("response",))
        if errors_:
            ctx = endpoint_ctx or EndpointContext()
            raise ResponseValidationError(errors=errors_, body=data, endpoint_ctx=ctx)
        # ... serialize_json call (identical) ...
```

#### b) Move `_serialize_sse_item` from inside `app()` to inside `get_request_handler()` (lines 516–543)

**Why it must change:** Same rationale as `_serialize_data`. The function is recreated on every SSE request.

**Responsibility today:** Serializes a single item for Server-Sent Events. If the item is a `ServerSentEvent` instance, formats its fields directly. Otherwise validates via `_serialize_data` and wraps in `format_sse_event()`.

**How the new architecture interacts:** The function is created once at route setup time. It calls `_serialize_data(item, endpoint_ctx)` instead of `_serialize_data(item)`, so it also needs `endpoint_ctx` as a parameter to pass through.

**Change type:** STRUCTURAL. Scope change and parameter addition.

**Specific code change:**
```python
# BEFORE (line 516, inside app()):
def _serialize_sse_item(item: Any) -> bytes:
    # ... calls _serialize_data(item) at line 542 ...

# AFTER (inside get_request_handler(), before app()):
def _serialize_sse_item(item: Any, endpoint_ctx: EndpointContext) -> bytes:
    # ... calls _serialize_data(item, endpoint_ctx) ...
```

#### c) Update call sites inside `app()` (lines 542, 575, 641)

**Why it must change:** Because `_serialize_data` and `_serialize_sse_item` now require `endpoint_ctx` as an explicit parameter, all call sites must pass it.

**Responsibility today:** These lines call the serialization helpers to convert yielded items to bytes for the response stream.

**How the new architecture interacts:** The call sites inside `app()` have access to `endpoint_ctx` (it is computed at line 406 inside `app()`), so passing it is straightforward.

**Change type:** STRUCTURAL. Only function call signatures change.

**Specific code changes:**
- Line 542: `data_str=_serialize_data(item).decode("utf-8")` → `data_str=_serialize_data(item, endpoint_ctx).decode("utf-8")`
- Line 575: `await send_stream.send(_serialize_sse_item(raw_item))` → `await send_stream.send(_serialize_sse_item(raw_item, endpoint_ctx))`
- Line 641: `return _serialize_data(item) + b"\n"` → `return _serialize_data(item, endpoint_ctx) + b"\n"`

#### d) Update `_serialize_item` (line 640–641) and its callers

**Why it must change:** `_serialize_item` (line 640) is defined inside `app()` and calls `_serialize_data`. It is used by `_async_stream_jsonl` (line 645) and `_sync_stream_jsonl` (line 657). Since `_serialize_data` now requires `endpoint_ctx`, `_serialize_item` must also accept and pass `endpoint_ctx`. The stream generators (`_async_stream_jsonl`, `_sync_stream_jsonl`) must also receive and pass `endpoint_ctx`.

**Responsibility today:** `_serialize_item` wraps `_serialize_data(item)` with a newline byte for JSONL format. `_async_stream_jsonl` and `_sync_stream_jsonl` iterate the endpoint generator and yield serialized items.

**How the new architecture interacts:** These helpers can also be hoisted to `get_request_handler()` scope (they don't capture any per-request variables except through `_serialize_data`). They would receive `endpoint_ctx` as a parameter and pass it through to `_serialize_data`.

**Change type:** STRUCTURAL.

**Specific code changes:**
```python
# _serialize_item (hoisted, before app()):
def _serialize_item(item: Any, endpoint_ctx: EndpointContext) -> bytes:
    return _serialize_data(item, endpoint_ctx) + b"\n"

# _async_stream_jsonl (hoisted, before app()):
async def _async_stream_jsonl(
    gen, _serialize_item, endpoint_ctx: EndpointContext
) -> AsyncIterator[bytes]:
    async for item in gen:
        yield _serialize_item(item, endpoint_ctx)
        await anyio.sleep(0)
```

#### e) `_sse_producer_cm` remains inside `app()` (line 551)

**Why it does NOT change:** `_sse_producer_cm` captures `sse_aiter`, which is per-request (computed at line 546 from `gen`, which is per-request). Moving it to `get_request_handler()` scope would require passing `sse_aiter` as a parameter, which would mean `_sse_producer_cm` cannot be an `@asynccontextmanager` that captures `sse_aiter` from closure. The complexity of threading `sse_aiter` through the context manager's `__aenter__` and `__anext__` calls outweighs the benefit.

**Responsibility today:** Sets up the SSE producer/consumer pipeline with memory streams, task group, and keepalive inserter.

**How the new architecture interacts:** `_sse_producer_cm` remains inside `app()` and continues to capture `sse_aiter` and `_serialize_sse_item` from `app()`'s local scope. Since `_serialize_sse_item` is now pre-created at `get_request_handler()` scope, `_sse_producer_cm` captures the pre-created function instead of creating it. This still saves one function object allocation per request.

**Change type:** NONE for `_sse_producer_cm` itself. The function definition stays in place. The only change is that the captured `_serialize_sse_item` is now a pre-created function object instead of a newly created one.

#### f) `_async_stream_raw` remains inside `app()` (line 674)

**Why it does NOT change:** Captures `gen` (per-request generator). Cannot be hoisted without threading `gen` as a parameter.

**Change type:** NONE.

---

### 2. `tests/test_stream_cancellation.py`

**Why it must change:** Not strictly required, but recommended. The test currently validates the cancellation safety of tight generator loops. If the helper hoisting changes the call stack depth (fewer nested frames), the test should still pass, but the test serves as a regression guard for the streaming path. No code changes needed, but the test must be run after the refactoring.

**Responsibility today:** Guards against hanging on tight generator loops (fastapi#14680).

**How the new architecture interacts:** The test exercises the full streaming pipeline including `_sse_with_checkpoints` and `_async_stream_jsonl`. After hoisting, the pipeline behavior is identical — only the function object allocation changes. The test should pass unchanged.

**Change type:** ORGANIZATIONAL (must be run to verify, but no test code changes needed).

---

### 3. `tests/test_stream_json_validation_error.py`

**Why it must change:** Same as above — no code changes needed, but must be run. This test specifically validates that `ResponseValidationError` is raised with `endpoint_ctx` populated. After hoisting, `endpoint_ctx` is still passed to `_serialize_data` and used in the error, so the behavior is preserved.

**Responsibility today:** Guards that `ResponseValidationError` is raised when a streaming item fails Pydantic validation.

**How the new architecture interacts:** The test exercises the `_serialize_data` → `stream_item_field.validate()` → `ResponseValidationError` path. After hoisting, the path is identical — only the function scope changes.

**Change type:** ORGANIZATIONAL.

---

### 4. `tests/test_stream_bare_type.py`

**Why it must change:** Same as above — no code changes needed, but must be run. This test validates bare `AsyncIterable` / `Iterable` return types. After hoisting, `_serialize_data` still handles the `stream_item_field is None` fallback path identically.

**Responsibility today:** Guards that bare type streaming serializes to JSONL correctly.

**How the new architecture interacts:** The test exercises the `_serialize_data` fallback path (`jsonable_encoder` + `json.dumps`). After hoisting, identical.

**Change type:** ORGANIZATIONAL.

---

### 5. `tests/test_sse.py`

**Why it must change:** If this test exists and tests SSE endpoints, it must be run to verify the SSE streaming path still works correctly after `_serialize_sse_item` is hoisted. No code changes needed.

**Responsibility today:** Tests Server-Sent Events functionality.

**How the new architecture interacts:** The SSE path involves `_sse_producer_cm`, `_producer`, `_serialize_sse_item`. After hoisting, `_serialize_sse_item` is pre-created but the behavior is identical.

**Change type:** ORGANIZATIONAL.

---

### 6. `tests/benchmarks/test_general_performance.py`

**Why it must change:** Not strictly required for the implementation, but highly recommended. The benchmark suite currently has zero streaming coverage. Adding streaming benchmarks (as recommended in a prior /context execution) would provide before/after performance data to measure the impact of the hoisting.

**Responsibility today:** Measures standard request/response performance.

**How the new architecture interacts:** The benchmark suite is unaffected by the hoisting because it doesn't measure streaming endpoints. However, adding streaming benchmarks would quantify the performance gain.

**Change type:** ORGANIZATIONAL (new file recommended: `tests/benchmarks/test_streaming_performance.py`).

---

## Files That Do NOT Need Changes

1. **`fastapi/applications.py`** — No references to `_serialize_data` or streaming helpers.
2. **`fastapi/dependencies/utils.py`** — The dependency injection system is independent of response serialization.
3. **`fastapi/middleware/`** — Middleware operates at the ASGI layer, above the request handler.
4. **`fastapi/openapi/`** — OpenAPI schema generation does not depend on runtime serialization helpers.
5. **`fastapi/exceptions.py`** — `ResponseValidationError` class definition is unchanged.
6. **`fastapi/testclient.py`** — TestClient is a generic HTTP client, not aware of internal helper functions.
7. **`fastapi/responses.py`** — `StreamingResponse`, `JSONResponse`, etc. are independent.
8. **`fastapi/sse.py`** — SSE formatting utilities are independent.
9. **`tests/test_dependency_*.py`** — Dependency injection tests are unaffected.
10. **`tests/test_main.py`**, **`tests/test_application.py`** — General application tests are unaffected.

---

## Adjacent Components That Could Benefit

### 1. `_sse_producer_cm` (routing.py:551)

**Why it could benefit:** `_sse_producer_cm` is also recreated on every request because it captures `sse_aiter` (per-request). The function is ~160 bytes plus closure cells. However, it captures `sse_aiter` from `app()`'s local scope, which is the per-request generator iterator. Moving it to `get_request_handler()` scope would require threading `sse_aiter` through the context manager's `__aenter__`, which is complex because the context manager is entered on the `async_exit_stack` at line 606–607.

**Scope decision:** OUT OF SCOPE for current work. The benefit is smaller than `_serialize_data` because `_sse_producer_cm` is called once per request (not per item), and the complexity of threading `sse_aiter` through the context manager protocol is high.

### 2. `_async_stream_raw` (routing.py:674)

**Why it could benefit:** This async generator wrapper adds `anyio.sleep(0)` checkpoints for cancellation safety. It is recreated per-request because it captures `gen` (per-request). Hoisting it would require passing `gen` as a parameter.

**Scope decision:** OUT OF SCOPE. The function is small (~10 lines) and the benefit is minimal compared to `_serialize_data`.

### 3. `_sse_with_checkpoints` (routing.py:613)

**Why it could benefit:** Similar to `_async_stream_raw`, adds `anyio.sleep(0)` checkpoints for SSE streams. Captures `sse_receive_stream` from `app()` local scope.

**Scope decision:** OUT OF SCOPE. Small function, minimal benefit.

### 4. `serialize_response()` (routing.py:293)

**Why it already follows the pattern:** `serialize_response()` is the module-level equivalent that was already extracted from `get_request_handler()`. It takes all parameters explicitly (`field`, `response_content`, `include`, `exclude`, `by_alias`, etc.). This is the existing precedent for Approach 1.

**Why it doesn't need change:** It already follows the recommended architecture. No action needed.

### 5. `run_endpoint_function()` (routing.py:336)

**Why it already follows the pattern:** Extracted from `get_request_handler()` for profiling (per comment at line 340: "since inner functions are harder to profile"). Takes `dependant`, `values`, `is_coroutine` explicitly.

**Why it doesn't need change:** Already follows the recommended architecture.

### 6. `_extract_endpoint_context()` (routing.py, near line 406)

**Why it could benefit:** This helper is called at line 406 inside `app()` to construct `endpoint_ctx`. It is a module-level function (not nested), so it doesn't have the per-request allocation problem. However, its result (`endpoint_ctx`) is passed to multiple downstream functions.

**Why it doesn't need change:** Already a module-level function — no per-request allocation issue.

---

## Summary Table

| Location | Must Change | Type | Responsibility | Impact |
|----------|------------|------|----------------|--------|
| `fastapi/routing.py` — `_serialize_data` definition | YES | Structural | Serialize item to bytes | Move from `app()` to `get_request_handler()`, add `endpoint_ctx` param |
| `fastapi/routing.py` — `_serialize_sse_item` definition | YES | Structural | Serialize SSE item | Move from `app()` to `get_request_handler()`, add `endpoint_ctx` param |
| `fastapi/routing.py` — call sites (lines 542, 575, 641) | YES | Structural | Call serialization helpers | Pass `endpoint_ctx` explicitly |
| `fastapi/routing.py` — `_serialize_item` definition | YES | Structural | Add newline wrapper | Move from `app()` to `get_request_handler()`, add `endpoint_ctx` param |
| `fastapi/routing.py` — `_async_stream_jsonl` | YES | Structural | JSONL generator wrapper | Move from `app()` to `get_request_handler()`, add `endpoint_ctx` param |
| `fastapi/routing.py` — `_sync_stream_jsonl` | YES | Structural | Sync JSONL generator | Move from `app()` to `get_request_handler()`, add `endpoint_ctx` param |
| `fastapi/routing.py` — `_sse_producer_cm` | NO | — | SSE producer/consumer | Stays inside `app()`, but captures pre-created `_serialize_sse_item` |
| `fastapi/routing.py` — `_async_stream_raw` | NO | — | Raw streaming wrapper | Stays inside `app()` |
| `tests/test_stream_*.py` | Run only | Organizational | Streaming tests | No code changes, must run to verify |
| `tests/benchmarks/` | Add new | Organizational | Performance baselines | Add `test_streaming_performance.py` to measure gain |
| `fastapi/applications.py` | NO | — | App initialization | No references to helpers |
| `fastapi/dependencies/utils.py` | NO | — | DI resolution | Independent of serialization |
| `fastapi/middleware/` | NO | — | ASGI middleware | Operates above request handler |
| `fastapi/openapi/` | NO | — | Schema generation | Independent of runtime |
| `fastapi/responses.py` | NO | — | Response classes | Independent |
| `fastapi/sse.py` | NO | — | SSE formatting | Independent |

**Total blast radius:** 6 structural changes in 1 file (`fastapi/routing.py`), plus 4 test files to run, plus 1 new benchmark file to add. Zero public API changes. Zero changes to other modules.