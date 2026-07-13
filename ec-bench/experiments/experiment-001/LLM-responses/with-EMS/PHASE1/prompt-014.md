model: kimi k2.5

## Validation Strategy: Streaming Helper Hoisting

The change is a pure structural move — five helpers lifted from `app()` (routing.py:398) to `get_request_handler()` (routing.py:367) scope with `endpoint_ctx` passed explicitly. No logic changes. Validation must therefore prove **behavioural preservation** and **allocation reduction**.

---

### Phase 1: Baseline Behaviour — Execute Existing Streaming Tests

| Test File | Tests | What It Validates |
|-----------|-------|-----------------|
| `tests/test_stream_cancellation.py` | 2 | Cancellation safety for raw `StreamingResponse` and JSONL paths. Validates the `anyio.sleep(0)` checkpoint mechanism in `_async_stream_jsonl` (routing.py:645) and `_sse_with_checkpoints` (routing.py:613). |
| `tests/test_stream_json_validation_error.py` | 2 | `ResponseValidationError` raised when streaming endpoints yield invalid items. Tests the `stream_item_field.validate()` path in `_serialize_data` (routing.py:488-498). |
| `tests/test_stream_bare_type.py` | 2 | Bare `AsyncIterable` / `Iterable` (no generic parameter) serialise to JSONL correctly. Tests the fallback `jsonable_encoder` + `json.dumps()` path (routing.py:508-510). |
| `tests/test_sse.py` | 19 | SSE path with async/sync generators, with/without annotations, dict items, POST method, SSE event fields, mixed plain/SSE events, keepalive pings. Validates `_serialize_sse_item` (routing.py:516) and `_sse_producer_cm` (routing.py:551). |

**Why Phase 1 is necessary:** These tests are the existing behavioural contract. Since the change is a pure move, every existing test must pass unmodified. Any failure indicates a scope-capture error or parameter-passing mistake.

**Execution:** Run first. Fastest feedback loop. If any test fails, stop.

---

### Phase 2: Targeted Functional Tests — Fill Coverage Gaps

Four new tests should be added:

1. **`test_sse_validation_error_endpoint_ctx`** — Verify that when an SSE endpoint yields an invalid item, the raised `ResponseValidationError` contains a correctly populated `endpoint_ctx` (route path, operation ID, etc.).
   *Rationale:* The existing `test_stream_json_validation_error.py` only checks that `ResponseValidationError` is raised; it does not inspect `endpoint_ctx`. Since `_serialize_data` is the function that constructs the error and it now receives `endpoint_ctx` as an explicit parameter, we must verify the parameter is correctly threaded through the SSE path (`_serialize_sse_item` → `_sse_producer_cm` → `_producer`).

2. **`test_jsonl_validation_error_endpoint_ctx`** — Same as above for the JSONL path (`_serialize_item` → `_async_stream_jsonl` / `_sync_stream_jsonl`).
   *Rationale:* The JSONL and SSE paths call `_serialize_data` through different helper chains. Both must be verified independently.

3. **`test_stream_response_model_filters`** — Verify that `response_model_include`, `response_model_exclude`, `response_model_by_alias`, `response_model_exclude_unset`, `response_model_exclude_defaults`, `response_model_exclude_none` are all respected in streaming endpoints (both JSONL and SSE).
   *Rationale:* These six variables are currently captured as route-static closure cells. When helpers are hoisted, they will access these variables from `get_request_handler()` scope instead of `app()` scope. A test must confirm the values are still correct at request time.

4. **`test_stream_dependency_cleanup`** — Verify that generator-based dependencies with `yield` are properly cleaned up after streaming response completion, using both `request_stack` and `function_stack` paths.
   *Rationale:* The `get_request_handler()` closure captures the `AsyncExitStack` instances created in `request_response()` (routing.py:113). Moving helpers out of `app()` must not change the lifecycle of these stacks or the timing of dependency cleanup.

**Why Phase 2 is necessary:** Phase 1 proves no regressions in tested paths. Phase 2 proves the changed code paths (explicit parameter passing, scope transitions) are correct for edge cases not currently tested.

**Execution:** After Phase 1 passes. These tests are written and added before the code change, then run against both the original and modified code. They should pass on the original (proving they are valid tests) and on the modified code (proving behaviour is preserved).

---

### Phase 3: Performance Benchmark — Prove Engineering Goal

Add `tests/benchmarks/test_streaming_performance.py`:

```python
# Measures per-request allocation and throughput for short-lived streams
# (the primary benefit case: 10,000+ req/s, <10 items each)
```

**Benchmark design:**

1. **Allocation benchmark** — Use `tracemalloc` to measure peak memory allocated during 1,000 requests to a JSONL endpoint yielding 5 items each. Compare baseline (helpers inside `app()`) vs. modified (helpers hoisted). Expect ~320 bytes reduction per request (two function objects × ~160 bytes each).
   *Rationale:* The engineering goal is reducing per-request allocation. This benchmark proves the goal is achieved.

2. **Throughput benchmark** — Use `pytest-benchmark` or CodSpeed to measure requests/second for the same endpoint. Expect measurable improvement for short-lived streams; expect no regression for long streams.
   *Rationale:* Proves the allocation reduction translates to throughput gain in the target use case.

**Why Phase 3 is necessary:** Without a benchmark, the PR has no quantitative evidence that the engineering goal is met. FastAPI maintainers require benchmark data for performance-related changes.

**Execution:** After Phase 2 passes. Run with `--codspeed` flag to integrate with CI.

---

### Phase 4: Full Regression Suite

Run the complete test suite (`pytest tests/`). The blast radius is small (1 file, 6 functions), but streaming code interacts with:
- Dependency injection (`solve_dependencies`, `request_stack`, `function_stack`)
- Exception handling (`wrap_app_handling_exceptions`, `ExceptionMiddleware`)
- Response construction (`StreamingResponse`, `EventSourceResponse`)

**Why Phase 4 is necessary:** Catches unexpected interactions in the broader codebase. The change is low-risk, but a full suite run is mandatory before merge.

**Execution:** Last. Only after Phases 1-3 pass.

---

### Execution Order

```
1. Phase 1  → Existing streaming tests (fastest feedback, must pass)
2. Phase 2  → Write and run targeted functional tests (edge case coverage)
3. Phase 3  → Write and run streaming benchmarks (engineering goal proof)
4. Phase 4  → Full regression suite (safety net)
```

---

### Sufficiency Justification

This strategy provides **sufficient engineering confidence** because:

- **Phase 1** establishes behavioural equivalence. If 25 existing streaming tests pass unmodified, the core logic is proven preserved.
- **Phase 2** addresses the specific risks introduced by the change: explicit parameter passing (`endpoint_ctx`), scope transition for route-static variables (response model filters), and request lifecycle preservation (dependency cleanup). These are the only behavioural dimensions that could change.
- **Phase 3** proves the engineering goal is achieved with quantitative data. A performance PR without benchmark evidence cannot be merged.
- **Phase 4** provides the standard FastAPI regression safety net.

The strategy is **proportionate to risk**: the change is a pure move with no logic changes, so extensive new test coverage is limited to the dimensions that actually change (parameter passing, scope). No unnecessary tests are added. The benchmark is the only new file; all other validation uses existing infrastructure.
