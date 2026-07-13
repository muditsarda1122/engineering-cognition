model: kimi k2.5

## Roadmap Proposal: Decompose the Request Execution Pipeline into Phased Architecture

### Ticket: ARCH-001 — Phased Request Execution Pipeline

**Status:** Roadmap Proposal  
**Priority:** Medium  
**Related:** PERF-001 (Streaming Serialization Closure Hoisting), PERF-002 (`_async_stream_raw` Hoisting)

---

### The Engineering Problem

The request execution pipeline in `fastapi/routing.py` is a monolithic 300-line closure (`app(request)` inside `get_request_handler`) that handles the entire lifecycle from body parsing to response building. Responsibilities that should be distinct phases are instead interleaved in a single massive function. This creates four systemic problems that we have been addressing tactically:

**1. Duplication across HTTP and WebSocket paths**
Both `get_request_handler` and `get_websocket_app` perform the same three operations independently:
- Extract `endpoint_ctx` (routing.py:603-612 vs routing.py:910-918)
- Call `solve_dependencies` (routing.py:670-677 vs routing.py:923-929)
- Check `solved_result.errors` and raise validation errors (routing.py:891-895 vs routing.py:930-934)

The only meaningful difference between these paths is that the HTTP version reads a request body before solving dependencies, and the WebSocket version prepends "WS" to the path prefix. Otherwise, the preparation logic is identical.

**2. Closure proliferation as a symptom, not a root cause**
The streaming serialization optimization (PERF-001) hoisted 5 helper closures to module level because they were recreated on every request inside the massive `app(request)` closure. `_async_stream_raw` (PERF-002) is the last remaining route-static closure. But the root cause is unchanged: `app(request)` is a 300-line closure that accumulates helper functions because there is no clean boundary between pipeline phases.

**3. Response building is fragmented and hard to reason about**
The HTTP path contains four different response-building branches (SSE, JSONL, raw streaming, regular response), each with different logic for constructing the `Response` object. These branches are deeply nested inside the closure and interleaved with body parsing, dependency solving, and error handling. Adding a fifth response type would require modifying the same 300-line function.

**4. Testability is limited to integration-level testing**
Because all phases are fused into a single closure, individual pipeline phases cannot be unit-tested. You can only test the full integration (which is why the streaming optimization required 32 streaming tests + 223 non-streaming tests to validate).

---

### Proposed Architectural Direction

Decompose the request execution pipeline into three distinct phases with clear interfaces. Each phase becomes a module-level function that can be tested independently.

**Phase 1: Request Preparation** (shared between HTTP and WebSocket)

```python
async def _prepare_request_context(
    *,
    request: Request | WebSocket,
    dependant: Dependant,
    body: Any = None,
    dependency_overrides_provider: Any | None = None,
    embed_body_fields: bool = False,
) -> SolvedDependencies:
    """
    Extract endpoint context, solve dependencies, and handle validation errors.
    Shared by both HTTP and WebSocket execution paths.
    """
```

This function would:
1. Extract `endpoint_ctx` using `_extract_endpoint_context`
2. Set the path prefix ("GET /path" for HTTP, "WS /path" for WebSocket)
3. Call `solve_dependencies`
4. Raise `RequestValidationError` or `WebSocketRequestValidationError` if errors exist

**Phase 2: Response Building** (strategy pattern)

```python
class ResponseBuilder(Protocol):
    async def build(
        self,
        *,
        dependant: Dependant,
        solved_result: SolvedDependencies,
        endpoint_ctx: EndpointContext,
        status_code: int | None,
        response_class: type[Response],
        response_field: ModelField | None,
        response_model_include: IncEx | None,
        ...
    ) -> Response:
        ...
```

Four concrete implementations:
- `SSEResponseBuilder` — handles SSE streaming with keepalive
- `JSONLResponseBuilder` — handles JSONL streaming
- `RawStreamingResponseBuilder` — handles raw async generator streaming
- `RegularResponseBuilder` — handles standard response serialization

**Phase 3: Response Finalization** (shared)

```python
def _finalize_response(
    response: Response,
    *,
    solved_result: SolvedDependencies,
    status_code: int | None,
) -> Response:
    """
    Set background tasks, status code, and headers.
    """
```

**Result: `app(request)` becomes a thin orchestrator**

```python
async def app(request: Request) -> Response:
    # Phase 1: Preparation
    solved_result = await _prepare_request_context(
        request=request,
        dependant=dependant,
        body=await _read_body(request, body_field, is_body_form, strict_content_type),
        dependency_overrides_provider=dependency_overrides_provider,
        embed_body_fields=embed_body_fields,
    )
    
    # Phase 2: Response building
    response = await response_builder.build(
        dependant=dependant,
        solved_result=solved_result,
        ...
    )
    
    # Phase 3: Finalization
    return _finalize_response(response, solved_result=solved_result, status_code=status_code)
```

The per-request closure shrinks from ~300 lines to ~20 lines. All helper closures become module-level functions or strategy implementations.

---

### Expected Long-term Engineering Benefits

**1. Eliminates duplication between HTTP and WebSocket paths**
Both paths would use the same `_prepare_request_context` function. The WebSocket path would become:
```python
async def app(websocket: WebSocket) -> None:
    solved_result = await _prepare_request_context(
        request=websocket,
        dependant=dependant,
        dependency_overrides_provider=dependency_overrides_provider,
        embed_body_fields=embed_body_fields,
    )
    await dependant.call(**solved_result.values)
```

**2. Completes the closure-hoisting pattern**
All helper functions currently defined inside `app(request)` — including the four SSE closures and `_async_stream_raw` — would become module-level functions or strategy methods. No route-static closures would remain inside the per-request function.

**3. Improves testability**
Each phase can be unit-tested independently:
- `_prepare_request_context` can be tested with mock `Request`/`WebSocket` objects
- Each `ResponseBuilder` implementation can be tested with mock generators and solved dependencies
- `_finalize_response` can be tested with mock `Response` objects

**4. Makes future streaming work easier**
Adding a new streaming format (e.g., NDJSON, CSV streaming) would require only a new `ResponseBuilder` implementation, not modifying a 300-line closure.

**5. Aligns with existing codebase precedent**
`serialize_response` (routing.py:293) and `run_endpoint_function` (routing.py:336) are already extracted module-level functions that separate concerns. This proposal extends the same pattern to the entire pipeline.

---

### Migration Complexity

**Medium.**

This is not a pure move — it requires restructuring the control flow and introducing a strategy pattern. However:

- **No public API changes.** All changes are internal to `fastapi/routing.py`.
- **No behavioral changes.** The phased pipeline must produce identical results.
- **Incremental extraction possible.** Each phase can be extracted independently:
  1. Extract `_prepare_request_context` first (low risk, shared logic)
  2. Extract `_finalize_response` second (low risk, shared logic)
  3. Extract response builders last (higher risk, requires testing each branch)

**Risk mitigation:**
- The existing test suite (32 streaming + 223 non-streaming tests) provides comprehensive regression coverage.
- Each extracted phase can be validated by temporarily duplicating the logic (old path + new path) and asserting identical results.

---

### Implementation Effort

**Medium.**

Estimated: 2-3 engineering days.

Breakdown:
1. Extract `_prepare_request_context` and share between HTTP/WebSocket paths (0.5 day)
2. Extract `_finalize_response` (0.25 day)
3. Extract `RegularResponseBuilder` (0.5 day)
4. Extract `JSONLResponseBuilder` and `RawStreamingResponseBuilder` (0.5 day)
5. Extract `SSEResponseBuilder` (0.5 day — most complex due to task groups and keepalive)
6. Refactor `app(request)` into orchestrator (0.25 day)
7. Validation: all 255+ tests must pass, mypy strict, ruff (0.5 day)

---

### Relation to Completed Work

**This is the natural conclusion of the streaming serialization optimization.**

PERF-001 and PERF-002 were tactical fixes: they hoisted individual helper closures to module level without changing the pipeline structure. The root cause — a monolithic 300-line closure that accumulates helpers — remains.

This roadmap addresses the root cause by:
- Eliminating the monolithic closure entirely
- Replacing it with a thin orchestrator that delegates to module-level phases
- Making the closure-hoisting pattern obsolete (no closures to hoist)

**Dependency:** This work should wait until PERF-002 (`_async_stream_raw` hoisting) is complete, so the closure inventory is stable before restructuring.

---

### Recommendation

**Include in the next quarterly architecture roadmap.** The engineering value is long-term (maintainability, testability, future extensibility) rather than immediate performance. The risk is low because the change is internal, incremental, and well-covered by tests. The effort is moderate and can be spread across multiple sprints.
