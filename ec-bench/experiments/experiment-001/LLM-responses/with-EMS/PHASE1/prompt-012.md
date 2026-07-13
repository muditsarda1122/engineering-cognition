model: kimi k2.5

## Request-Specific State Flow Analysis

### Scope

The proposed architecture hoists `_serialize_data`, `_serialize_sse_item`, `_serialize_item`, `_async_stream_jsonl`, and `_sync_stream_jsonl` from inside `app()` to `get_request_handler()` scope. This changes how request-specific state flows through the streaming pipeline. The analysis below identifies every piece of state whose flow changes, explains the risk, and describes the correct handling.

---

## 1. `endpoint_ctx` — Error Reporting Context

**Why it exists:** `endpoint_ctx` (routing.py:406) carries the endpoint path, method, and function name for error messages. It is constructed by `_extract_endpoint_context(dependant.call)` and augmented with the mount path and HTTP method at line 415: `endpoint_ctx["path"] = f"{request.method} {mount_path}{dependant.path}"`.

**Classification:** REQUEST-SPECIFIC. The mount path and HTTP method come from the request scope. The endpoint function name comes from `dependant.call`, which is route-static, but the path string is per-request.

**Current flow:** `endpoint_ctx` is a local variable of `app()`. It is captured by `_serialize_data` (line 493) via Python closure semantics. When `_serialize_data` raises `ResponseValidationError`, it includes `endpoint_ctx` so the error message shows which endpoint failed.

**Proposed flow:** `endpoint_ctx` must become an **explicit parameter** passed to every hoisted helper in the call chain:

```
app(endpoint_ctx)
  → _serialize_data(data, endpoint_ctx)
  → _serialize_sse_item(item, endpoint_ctx)
    → _serialize_data(item, endpoint_ctx)
  → _serialize_item(item, endpoint_ctx)
    → _serialize_data(item, endpoint_ctx)
  → _async_stream_jsonl(gen, endpoint_ctx)
    → _serialize_item(item, endpoint_ctx)
  → _sync_stream_jsonl(gen, endpoint_ctx)
    → _serialize_item(item, endpoint_ctx)
```

For SSE, the flow continues into the closure chain:
```
app(endpoint_ctx)
  → _sse_producer_cm (captures endpoint_ctx via closure)
    → _producer (captures endpoint_ctx via closure chain)
      → _serialize_sse_item(raw_item, endpoint_ctx)
```

**Behavioral regression if handled incorrectly:**
- **Critical:** If `endpoint_ctx` is not passed to `_serialize_sse_item` inside `_producer`, `_producer` would call `_serialize_sse_item(raw_item)` without the context. When `_serialize_sse_item` calls `_serialize_data(item, endpoint_ctx)`, it would raise a `TypeError` (missing argument) before `ResponseValidationError` can even be constructed.
- **Moderate:** If `endpoint_ctx` is captured from `app()`'s scope into `_sse_producer_cm` correctly but `_producer` captures the wrong cell (e.g., an old cell from a previous request due to closure chain confusion), the error message would show stale endpoint information. This is unlikely in Python's closure semantics but worth noting.
- **Moderate:** If `endpoint_ctx` is `None` or default-constructed inside the helper instead of being passed, `ResponseValidationError` would lack endpoint path and method, making production debugging significantly harder.

---

## 2. `gen` — The Endpoint Generator

**Why it exists:** `gen` (routing.py:514, 638, 671) is the generator object returned by calling the endpoint function: `gen = dependant.call(**solved_result.values)`. It yields the items that are serialized and streamed to the client.

**Classification:** REQUEST-SPECIFIC. The generator is created fresh for each request with the resolved dependency values. It may hold open resources (database connections, file handles) and has a lifecycle tied to the request.

**Current flow:** `gen` is a local variable of `app()` inside `if is_sse_stream:`, `elif is_json_stream:`, and `elif is_async_gen_callable or dependant.is_gen_callable:`. It is captured by `_async_stream_jsonl` (line 646), `_sync_stream_jsonl` (line 658), and `_async_stream_raw` (line 677) via closure.

**Proposed flow:** `gen` must be passed as an **explicit parameter** to the hoisted helpers:

```
app(gen)
  → _async_stream_jsonl(gen, endpoint_ctx)
  → _sync_stream_jsonl(gen, endpoint_ctx)
```

For SSE, `gen` stays inside `app()` because `_sse_producer_cm` (which remains nested) needs it to create `sse_aiter`:
```
app(gen)
  → sse_aiter = gen.__aiter__()  (or iterate_in_threadpool(gen))
  → _sse_producer_cm (captures sse_aiter, not gen directly)
```

**Behavioral regression if handled incorrectly:**
- **Critical:** If `gen` is passed by reference correctly but the hoisted helper holds a reference to it after the request completes, the generator may not be garbage-collected promptly, leaking resources. However, this is the same behavior as today — the generator is held by the streaming response iterator until the response completes.
- **Moderate:** If `gen` is not passed and the hoisted helper captures a stale `gen` from a previous request (impossible in Python closure semantics but worth considering in alternative implementations), the endpoint would stream data from the wrong request.
- **Low:** If `gen` is a sync generator and `_async_stream_jsonl` is called with it (or vice versa), the type mismatch would cause an immediate `AttributeError` (sync generator has no `__aiter__`). This is guarded by `dependant.is_async_gen_callable` at lines 543, 643, 672.

---

## 3. `sse_aiter` — SSE Async Iterator

**Why it exists:** `sse_aiter` (routing.py:546) is the async iterator over the endpoint generator for SSE streams. It is either `gen.__aiter__()` for async generators or `iterate_in_threadpool(gen)` for sync generators.

**Classification:** REQUEST-SPECIFIC. Created per-request from the per-request `gen`.

**Current flow:** `sse_aiter` is a local variable of `app()` inside `if is_sse_stream:`. It is captured by `_sse_producer_cm` (via `_producer` at line 574) via closure.

**Proposed flow:** `sse_aiter` **stays inside `app()`** because `_sse_producer_cm` remains nested inside `app()`. The flow is unchanged:

```
app(sse_aiter)
  → _sse_producer_cm (captures sse_aiter via closure)
    → _producer (captures sse_aiter via closure chain)
```

**Behavioral regression if handled incorrectly:**
- **Low:** If someone attempts to hoist `_sse_producer_cm` to `get_request_handler()` scope (out of scope for current work but worth noting), `sse_aiter` would need to be threaded through the `@asynccontextmanager` protocol, which is complex and error-prone. The context manager's `__aenter__` and `__anext__` don't naturally accept per-request parameters.
- **None for the proposed architecture:** `sse_aiter` flow is unchanged.

---

## 4. `body` — Request Body

**Why it exists:** `body` (routing.py:419) is the parsed request body (form data, JSON, or raw bytes) read from the `Request` object before dependency resolution.

**Classification:** REQUEST-SPECIFIC. Unique to each request.

**Current flow:** `body` is a local variable of `app()`. It is passed to `solve_dependencies()` (line 476) and used in `RequestValidationError` (line 737) if validation fails.

**Proposed flow:** `body` is **not used by any hoisted helper**. It is used only by `app()` directly and by `solve_dependencies()`. The flow is unchanged.

**Behavioral regression if handled incorrectly:**
- **None for the proposed architecture.**

---

## 5. `async_exit_stack` — Request-Scoped Cleanup

**Why it exists:** `async_exit_stack` (routing.py:469) is the `AsyncExitStack` created by `request_response()` for request-scoped dependency cleanup. It is retrieved from `request.scope["fastapi_inner_astack"]`.

**Classification:** REQUEST-SPECIFIC. Created per-request by the request-scoped middleware.

**Current flow:** `async_exit_stack` is a local variable of `app()`. It is passed to `solve_dependencies()` (line 478) and used to enter `_sse_producer_cm()` (line 606).

**Proposed flow:** `async_exit_stack` is **not used by any hoisted helper**. It is used only by `app()` directly and by `solve_dependencies()`. The flow is unchanged.

**Behavioral regression if handled incorrectly:**
- **None for the proposed architecture.**

---

## 6. `solved_result` — Dependency Resolution Result

**Why it exists:** `solved_result` (routing.py:473) is the result of `solve_dependencies()`, containing `values` (resolved dependencies), `errors` (validation errors), `background_tasks`, and `response` (dependency response headers).

**Classification:** REQUEST-SPECIFIC. The resolved values depend on the request parameters.

**Current flow:** `solved_result` is a local variable of `app()`. Its fields are used at multiple points:
- `solved_result.values` → passed to `dependant.call()` (line 514, 638, 671)
- `solved_result.errors` → checked at line 481, 735
- `solved_result.background_tasks` → passed to `StreamingResponse` (line 630, 666)
- `solved_result.response.headers.raw` → merged into response (line 635, 668, 688, 734)

**Proposed flow:** `solved_result` is **not used by any hoisted helper**. It is used only by `app()` directly. The flow is unchanged.

**Behavioral regression if handled incorrectly:**
- **None for the proposed architecture.**

---

## 7. `response` — The Response Object

**Why it exists:** `response` (routing.py:399) is the `Response` object being constructed and returned by `app()`. It is set by each branch of the conditional (SSE, JSONL, raw streaming, or standard response).

**Classification:** REQUEST-SPECIFIC. Created per-request.

**Current flow:** `response` is a local variable of `app()`. It is set in each branch and returned at line 743.

**Proposed flow:** `response` is **not used by any hoisted helper**. The flow is unchanged.

**Behavioral regression if handled incorrectly:**
- **None for the proposed architecture.**

---

## 8. `_serialize_data` Function Object (as State)

**Why it exists:** `_serialize_data` is a helper function that serializes items. In the current implementation, it is a nested function created inside `app()` on every request.

**Classification:** In the current architecture, it is REQUEST-SPECIFIC (new function object per request). In the proposed architecture, it becomes ROUTE-STATIC (one function object per route).

**Current flow:** `_serialize_data` is created inside `app()` on every request. It captures `endpoint_ctx` (per-request) and 7 route-static variables via closure.

**Proposed flow:** `_serialize_data` is created once at `get_request_handler()` scope. It captures 7 route-static variables from `get_request_handler()` scope. `endpoint_ctx` is passed as an explicit parameter on every call.

**Behavioral regression if handled incorrectly:**
- **Low:** If `_serialize_data` is accidentally created inside `app()` (reverting the change), the per-request allocation resumes, negating the performance benefit.
- **Low:** If `_serialize_data` is hoisted but `endpoint_ctx` is not passed as a parameter (e.g., someone tries to capture it from `app()` scope via closure), Python would raise a `NameError` because `endpoint_ctx` is not in `get_request_handler()` scope.
- **Low:** If the 7 route-static closure cells are accidentally recreated (e.g., by creating `_serialize_data` inside a loop in `get_request_handler()`), memory usage would increase without performance benefit.

---

## 9. `_serialize_sse_item` Function Object (as State)

**Why it exists:** `_serialize_sse_item` serializes SSE items. In the current implementation, it is created inside `app()` on every SSE request.

**Classification:** In the current architecture, REQUEST-SPECIFIC. In the proposed architecture, ROUTE-STATIC.

**Current flow:** `_serialize_sse_item` is created inside `app()` inside `if is_sse_stream:`. It captures `_serialize_data` (newly created each request) and calls it.

**Proposed flow:** `_serialize_sse_item` is created once at `get_request_handler()` scope. It calls the pre-created `_serialize_data(data, endpoint_ctx)`.

**Behavioral regression if handled incorrectly:**
- **Low:** If `_serialize_sse_item` is hoisted but `_serialize_data` is not, `_serialize_sse_item` would capture `_serialize_data` from `get_request_handler()` scope (which is fine, because `_serialize_data` is also hoisted). But if `_serialize_data` is still inside `app()`, `_serialize_sse_item` would capture it from `app()` scope, which is route-static (same function object across requests) but adds closure chain overhead.
- **Moderate:** If `_serialize_sse_item` calls `_serialize_data(item)` without `endpoint_ctx`, and `_serialize_data` is now expecting `endpoint_ctx` as a parameter, the call would raise `TypeError`.

---

## 10. `sse_receive_stream` and `sse_stream_content` — SSE Streaming State

**Why they exist:** `sse_receive_stream` (routing.py:606) is the receive stream from the SSE producer context manager. `sse_stream_content` (routing.py:623) is the async iterator with checkpoints.

**Classification:** REQUEST-SPECIFIC. Created per-request.

**Current flow:** `sse_receive_stream` is created by entering `_sse_producer_cm()` on `async_exit_stack`. `sse_stream_content` is created by wrapping `sse_receive_stream` with `_sse_with_checkpoints()`.

**Proposed flow:** Both variables are **not used by any hoisted helper**. They are used only by `app()` directly. The flow is unchanged.

**Behavioral regression if handled incorrectly:**
- **None for the proposed architecture.**

---

## Summary Table: State Flow in Proposed Architecture

| State | Current Flow | Proposed Flow | Classification | Regression Risk |
|-------|-------------|---------------|----------------|-----------------|
| `endpoint_ctx` | Captured by `_serialize_data` from `app()` local scope | Explicit parameter to all hoisted helpers | Request-specific | **HIGH** — Missing context breaks error reporting |
| `gen` | Captured by `_async_stream_jsonl`/`_sync_stream_jsonl` from `app()` | Explicit parameter to hoisted stream helpers | Request-specific | **MODERATE** — Wrong generator streams wrong data |
| `sse_aiter` | Captured by `_sse_producer_cm` from `app()` | Unchanged (stays inside `app()`) | Request-specific | None |
| `body` | Used by `solve_dependencies()` and error reporting | Unchanged | Request-specific | None |
| `async_exit_stack` | Used by `solve_dependencies()` and `_sse_producer_cm` | Unchanged | Request-specific | None |
| `solved_result` | Used by `app()` directly | Unchanged | Request-specific | None |
| `response` | Constructed and returned by `app()` | Unchanged | Request-specific | None |
| `_serialize_data` (function) | Created per-request inside `app()` | Created once at `get_request_handler()` scope | Becomes route-static | Low — Revert negates benefit |
| `_serialize_sse_item` (function) | Created per-request inside `app()` | Created once at `get_request_handler()` scope | Becomes route-static | Low — Missing param causes TypeError |
| `sse_receive_stream` | Created by `_sse_producer_cm` entry | Unchanged | Request-specific | None |

---

## Critical Integration Points for Implementation

### Error Reporting (Highest Risk)

The most critical flow is `endpoint_ctx` through the SSE path:

```
app()
  → endpoint_ctx (local)
    → _sse_producer_cm (captures endpoint_ctx via closure)
      → _producer (captures endpoint_ctx via closure chain)
        → _serialize_sse_item(raw_item, endpoint_ctx)
          → _serialize_data(item, endpoint_ctx)
            → ResponseValidationError(errors, body, endpoint_ctx=ctx)
```

**Implementation requirement:** `_producer` (line 572) must capture `endpoint_ctx` from `app()`'s scope. Because `_producer` is nested inside `_sse_producer_cm` which is nested inside `app()`, Python's closure chain automatically propagates `endpoint_ctx` to `_producer` if `_producer` references it. The implementer must ensure `_producer` calls `_serialize_sse_item(raw_item, endpoint_ctx)` at line 575.

### Streaming Execution (Medium Risk)

For JSONL streaming:

```
app()
  → gen (local)
    → _async_stream_jsonl(gen, endpoint_ctx)  (hoisted, called with params)
      → _serialize_item(item, endpoint_ctx)
        → _serialize_data(item, endpoint_ctx)
```

**Implementation requirement:** `_async_stream_jsonl` must accept `gen` and `endpoint_ctx` as parameters. The generator `gen` must not be closed prematurely — `_async_stream_jsonl` must yield all items before the generator is garbage-collected.

### Request Lifecycle Management (Low Risk)

The proposed architecture does not change:
- Body reading order (before `solve_dependencies`)
- Dependency cleanup via `async_exit_stack`
- File cleanup via `file_stack`
- Response header merging from `solved_result.response.headers.raw`
- Background task attachment

The only lifecycle change is that `_serialize_data` now exists before `app()` is called, so it is available during the entire request. This is a superset of the current behavior (where it was created inside `app()` after body reading), so no regression.

---

## Verification Strategy

To ensure no regressions occur:

1. **Run `tests/test_stream_json_validation_error.py`** — Validates that `ResponseValidationError` contains `endpoint_ctx` with the correct path and method.
2. **Run `tests/test_stream_cancellation.py`** — Validates that generator lifecycle and cancellation checkpoints are preserved.
3. **Run `tests/test_stream_bare_type.py`** — Validates that bare type streaming (no `stream_item_field`) still works via the `jsonable_encoder` fallback.
4. **Run `tests/test_sse.py`** — Validates that SSE streaming with `ServerSentEvent` items works correctly.
5. **Add streaming benchmarks** — Measure before/after to confirm performance improvement and no latency regression.

**Test coverage gap:** There is no existing test that validates `endpoint_ctx` content specifically for SSE endpoints with `ResponseValidationError`. A new test should be added to cover this path after hoisting.