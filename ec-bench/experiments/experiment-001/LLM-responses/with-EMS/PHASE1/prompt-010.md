model: kimi k2.5

## Architectural Approaches to Eliminate Repeated Helper Creation

### Context Summary

The investigation confirmed:
- `_serialize_data` (routing.py:487), `_serialize_sse_item` (routing.py:516), and `_sse_producer_cm` (routing.py:551) are created inside `app()` on every request
- Function objects are ~160 bytes each, closure cells ~40 bytes each
- Only `endpoint_ctx` is per-request; all other captured variables are route-static and already shared
- The non-streaming path already uses a module-level `serialize_response()` (routing.py:293) — proving FastAPI has precedent for extracting helpers
- The streaming helpers form a call graph: `_sse_producer_cm` → `_producer` → `_serialize_sse_item` → `_serialize_data`
- Helpers are conditionally created (`if not errors:`, `if is_sse_stream:`, `elif is_json_stream:`)
- `endpoint_ctx` is used for error reporting in `ResponseValidationError`

---

## Approach 1: Hoist Helpers to `get_request_handler()` Scope

**Design:** Move `_serialize_data` and `_serialize_sse_item` definitions from inside `app()` to inside `get_request_handler()`, at the same scope level as `app()`. Both functions would take `endpoint_ctx` as an explicit parameter rather than capturing it from `app()`'s local scope. `_sse_producer_cm` would remain inside `app()` because it captures `sse_aiter`, which is per-request.

**Implementation complexity:** Low. Only `_serialize_data` and `_serialize_sse_item` need to move. Their logic is unchanged; only the `endpoint_ctx` parameter is added. The call sites inside `app()` would pass `endpoint_ctx` explicitly.

**Maintainability:** High. The code remains functional and readable. The helpers are still close to their usage (inside the same `get_request_handler()` function) but no longer recreated per-request. The change is localized to `routing.py`.

**Compatibility:** Excellent. No public API changes. No behavioural changes. The function signatures inside `app()` are unchanged. Error reporting via `ResponseValidationError` with `endpoint_ctx` is preserved because `endpoint_ctx` is still passed to the helper.

**Performance implications:** Eliminates per-request allocation for `_serialize_data` (~160 bytes) and `_serialize_sse_item` (~160 bytes). The 7 route-static closure cells for `_serialize_data` are already shared, but the function object itself is recreated. This approach eliminates that recreation. For 10,000 req/s, this saves ~320 bytes/request = ~3.2 MB/s allocation.

**Error reporting:** Preserved. `endpoint_ctx` is passed explicitly to `_serialize_data`, which raises `ResponseValidationError(errors, body, endpoint_ctx=ctx)` exactly as today.

**Ease of testing:** Good. The helpers can still be tested indirectly via endpoint tests. They cannot be unit-tested independently (still nested in `get_request_handler()`), but FastAPI's existing test strategy already tests via endpoints.

---

## Approach 2: Module-Level Factory Functions

**Design:** Create module-level factory functions:

```python
def _make_serialize_data(
    stream_item_field, response_model_include, response_model_exclude,
    response_model_by_alias, response_model_exclude_unset,
    response_model_exclude_defaults, response_model_exclude_none
):
    def _serialize_data(data: Any, endpoint_ctx: EndpointContext) -> bytes:
        # ... same logic, but endpoint_ctx is parameter not closure var
    return _serialize_data

def _make_serialize_sse_item(_serialize_data):
    def _serialize_sse_item(item: Any) -> bytes:
        # ... same logic
    return _serialize_sse_item
```

`get_request_handler()` calls these factories once at setup time, capturing the returned functions in its local scope. `app()` then uses the pre-created functions.

**Implementation complexity:** Medium. Requires defining new module-level functions and wiring them through `get_request_handler()`. The factory pattern adds a layer of indirection.

**Maintainability:** Medium-High. Separates setup-time binding from runtime execution clearly. However, the factory functions are module-level, which slightly reduces locality — a developer reading `app()` would need to look elsewhere to understand the helper logic.

**Compatibility:** Excellent. No public API changes. The factory functions are internal.

**Performance implications:** Identical to Approach 1 in terms of eliminating per-request allocation. The factory functions are called once at setup time, not per-request.

**Error reporting:** Preserved. `endpoint_ctx` is an explicit parameter to the inner `_serialize_data`, just as in Approach 1.

**Ease of testing:** Better than Approach 1. The factory functions can be unit-tested independently by passing mock parameters. This is a significant improvement over the current nested approach.

---

## Approach 3: Small Helper Class

**Design:** Introduce a lightweight `StreamingSerializer` class:

```python
class StreamingSerializer:
    def __init__(self, stream_item_field, response_model_include, ...):
        self.stream_item_field = stream_item_field
        self.response_model_include = response_model_include
        # ... etc
    
    def serialize_data(self, data: Any, endpoint_ctx: EndpointContext) -> bytes:
        # ... same logic as _serialize_data, using self.* for route-static vars
    
    def serialize_sse_item(self, item: Any) -> bytes:
        # ... same logic as _serialize_sse_item
```

`get_request_handler()` instantiates this class once at setup time. `app()` calls `serializer.serialize_data(data, endpoint_ctx)` and `serializer.serialize_sse_item(item)`.

**Implementation complexity:** Medium. Requires introducing a new class and converting function-local variables to instance attributes. The logic is unchanged but the access pattern changes from closure variables to `self.*`.

**Maintainability:** Medium. Object-oriented programmers may find this more natural, but the existing codebase is heavily functional/closure-based. The class introduces a new abstraction that doesn't exist elsewhere in `routing.py`. It also makes the code slightly more verbose (`self.stream_item_field` vs `stream_item_field`).

**Compatibility:** Good. No public API changes. However, it introduces a new pattern (classes for helpers) that is inconsistent with the existing `serialize_response()` module-level function.

**Performance implications:** Similar to Approach 1 for eliminating per-request function allocation. But the class instance adds ~200-300 bytes of overhead (the instance object + attribute dict). For the typical use case (one serializer per route), this is negligible and paid once at setup. However, calling methods on the instance adds one `self` parameter pass per call, which is slightly more overhead than a plain function call.

**Error reporting:** Preserved. `endpoint_ctx` is still passed explicitly.

**Ease of testing:** Excellent. The class can be instantiated with mock parameters and unit-tested independently. This is the best testing story of all approaches.

---

## Approach 4: Module-Level Functions with `functools.partial`

**Design:** Define the helpers as module-level functions with all parameters explicit:

```python
def _serialize_data_raw(
    data: Any, endpoint_ctx: EndpointContext,
    stream_item_field, response_model_include, ...
) -> bytes:
    # ... same logic

def _serialize_sse_item_raw(
    item: Any, _serialize_data: Callable, ...
) -> bytes:
    # ... same logic
```

In `get_request_handler()`:

```python
from functools import partial

_serialize_data = partial(
    _serialize_data_raw,
    stream_item_field=stream_item_field,
    response_model_include=response_model_include,
    ...
)
```

The partial function captures route-static variables. `app()` calls `_serialize_data(data, endpoint_ctx)`.

**Implementation complexity:** Low. `functools.partial` is standard library. The module-level functions are straightforward. The partial application in `get_request_handler()` is a one-liner.

**Maintainability:** High. Purely functional approach with explicit parameter binding. No classes, no nested functions. The code is very explicit about what data is route-static vs per-request.

**Compatibility:** Excellent. `functools.partial` is standard Python. No new abstractions.

**Performance implications:** `functools.partial` creates a wrapper object (~72 bytes) that holds the bound arguments. This wrapper is created once at setup time. The per-request call adds one wrapper dispatch, which is negligible. This approach eliminates the per-request function object allocation for the inner function while keeping the code flat.

**Error reporting:** Preserved. `endpoint_ctx` is passed explicitly.

**Ease of testing:** Excellent. The raw functions `_serialize_data_raw` and `_serialize_sse_item_raw` can be unit-tested independently with full parameter control.

---

## Comparative Evaluation Matrix

| Dimension | Approach 1 (Hoist) | Approach 2 (Factory) | Approach 3 (Class) | Approach 4 (Partial) |
|-----------|-------------------|----------------------|--------------------|---------------------|
| **Implementation complexity** | Low | Medium | Medium | Low |
| **Maintainability** | High | Medium-High | Medium | High |
| **Compatibility** | Excellent | Excellent | Good | Excellent |
| **Performance gain** | High | High | High | High |
| **Error reporting** | Preserved | Preserved | Preserved | Preserved |
| **Testability** | Good | Better | Excellent | Excellent |
| **Consistency with codebase** | Excellent (follows `serialize_response` pattern) | Good (new pattern) | Poor (introduces class) | Good (functional) |
| **Risk of regression** | Very low | Low | Medium | Very low |

---

## Recommendation: Approach 1 — Hoist to `get_request_handler()` Scope

### Why Approach 1 is Best

1. **Lowest risk of regression.** The change is a pure move — no logic changes, no new abstractions, no new patterns. The code inside the helpers is identical. The only change is adding `endpoint_ctx` as an explicit parameter and passing it at call sites.

2. **Follows established precedent.** FastAPI already extracted `serialize_response()` (routing.py:293) and `run_endpoint_function()` (routing.py:336) from `get_request_handler()` for exactly the same reasons (profiling, avoiding nested function recreation). The codebase itself demonstrates that extracting helpers from `app()` is the preferred pattern.

3. **Highest consistency with existing architecture.** The codebase is functional/closure-based, not class-based. Approach 3 (class) would introduce an alien pattern. Approach 1 fits the existing style perfectly.

4. **Maintains locality.** The helpers remain inside `get_request_handler()`, so a developer reading the code can still see the full streaming pipeline in one function. They don't need to jump to module-level definitions to understand the logic.

5. **Sufficient performance gain.** Eliminating `_serialize_data` and `_serialize_sse_item` per-request allocation saves ~320 bytes/request. While `_sse_producer_cm` and its nested `_producer`/`_keepalive_inserter` would still be recreated (because they need per-request `sse_aiter`), the most frequently called helper (`_serialize_data`) is hoisted. The experiment showed `_serialize_data` is called for every yielded item — it's the hottest helper.

6. **Preserves error reporting exactly.** The `ResponseValidationError` raised in `_serialize_data` still receives `endpoint_ctx` correctly because it's passed as a parameter.

### Implementation Sketch

```python
def get_request_handler(...):
    # ... existing setup code ...
    
    # Extracted helper — created once at setup time, not per-request
    def _serialize_data(data: Any, endpoint_ctx: EndpointContext) -> bytes:
        if stream_item_field:
            value, errors_ = stream_item_field.validate(data, {}, loc=("response",))
            if errors_:
                ctx = endpoint_ctx or EndpointContext()
                raise ResponseValidationError(errors=errors_, body=data, endpoint_ctx=ctx)
            return stream_item_field.serialize_json(
                value,
                include=response_model_include,
                exclude=response_model_exclude,
                by_alias=response_model_by_alias,
                exclude_unset=response_model_exclude_unset,
                exclude_defaults=response_model_exclude_defaults,
                exclude_none=response_model_exclude_none,
            )
        else:
            data = jsonable_encoder(data)
            return json.dumps(data).encode("utf-8")
    
    def _serialize_sse_item(item: Any) -> bytes:
        # ... same logic, calls _serialize_data(item) ...
    
    async def app(request: Request) -> Response:
        # ... existing code ...
        if not errors:
            # Use pre-created _serialize_data, passing endpoint_ctx explicitly
            # _serialize_sse_item is also pre-created
            # _sse_producer_cm remains inside app() because it needs sse_aiter
            ...
```

### What Would NOT Change

- The conditional creation logic (`if not errors:`, `if is_sse_stream:`) remains inside `app()`
- `_sse_producer_cm` and its nested `_producer`/`_keepalive_inserter` remain inside `app()` because they capture per-request `sse_aiter`
- `_serialize_item`, `_async_stream_jsonl`, `_sync_stream_jsonl` can also be hoisted since they only need `_serialize_data` (which is now pre-created)
- `_async_stream_raw` remains inside `app()` because it captures per-request `gen`
- All error reporting via `ResponseValidationError` with `endpoint_ctx` is preserved

### Migration Path

1. Move `_serialize_data` and `_serialize_sse_item` to `get_request_handler()` scope
2. Add `endpoint_ctx` parameter to `_serialize_data`
3. Update call sites inside `app()` to pass `endpoint_ctx`
4. Optionally move `_serialize_item` and `_async_stream_jsonl`/`_sync_stream_jsonl` as well
5. Run the streaming test suite (tests/test_stream_*.py) to verify correctness
6. Add streaming benchmarks (as recommended in prior evaluation) to measure performance improvement

---

## Why Not the Other Approaches?

- **Approach 2 (Factory):** Adds unnecessary indirection. The factory pattern is elegant but overkill when a simple move achieves the same result. FastAPI's codebase prefers directness over abstraction layers.

- **Approach 3 (Class):** Inconsistent with the existing functional style. Introduces a new pattern (helper classes) that doesn't exist elsewhere in `routing.py`. The `self.*` access is more verbose and doesn't provide meaningful encapsulation beyond what closures already provide.

- **Approach 4 (Partial):** `functools.partial` is elegant but creates a wrapper object that adds dispatch overhead per call. The wrapper is also less transparent for debugging than a direct function. Approach 1 achieves the same performance gain with simpler code.

### Conclusion

Approach 1 (hoisting helpers to `get_request_handler()` scope) is the best balance of engineering simplicity, maintainability, and performance. It follows FastAPI's own precedent (`serialize_response`, `run_endpoint_function`), requires minimal code changes, introduces zero new abstractions, and eliminates the per-request allocation of the hottest helper (`_serialize_data`). The risk of regression is negligible because the logic is unchanged — only the scope of definition changes.