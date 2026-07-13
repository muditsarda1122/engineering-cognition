model: kimi k2.5

## Technical Proposal: Hoist `_async_stream_raw` to Module Level

### Ticket: PERF-002 — Complete Per-Request Closure Elimination in Streaming Paths

**Status:** Proposal  
**Priority:** Low-Medium  
**Related:** PERF-001 (Streaming Serialization Closure Hoisting)

---

### Suspected Bottleneck

Inside `get_request_handler()` in `fastapi/routing.py`, the raw async streaming path (line ~830) defines a helper closure `_async_stream_raw` inside the per-request `app(request)` function:

```python
async def _async_stream_raw(
    async_gen: AsyncIterator[Any],
) -> AsyncIterator[Any]:
    async for chunk in async_gen:
        yield chunk
        await anyio.sleep(0)
```

This closure is recreated on **every request** for endpoints where `dependant.is_async_gen_callable` is true and an explicit `response_class` (e.g. `StreamingResponse`) is used. Unlike the SSE/JSONL helpers that were hoisted to module level in PERF-001, this closure was left behind.

---

### Why It Deserves Attention

1. **Pattern Inconsistency.** PERF-001 eliminated per-request closures for all other streaming serialization helpers (`_serialize_data`, `_serialize_sse_item`, `_serialize_item`, `_async_stream_jsonl`, `_sync_stream_jsonl`). Leaving `_async_stream_raw` as a per-request closure leaves the streaming optimization incomplete and creates an inconsistency in the request handling pipeline.

2. **It Has No Captured Variables.** The closure captures **nothing** from the outer scope. Its behavior is entirely route-static. This makes it the simplest possible hoisting candidate — easier than any of the five helpers already moved to module level.

3. **Cancellation Safety Is Already Solved.** The `anyio.sleep(0)` checkpoint that `_async_stream_raw` inserts is the same cancellation mechanism used by `async_stream_jsonl`. The pattern is proven safe.

---

### Expected Engineering Benefit

- **Memory:** ~160 bytes/request saved for raw async streaming endpoints (one function object eliminated).
- **Consistency:** Completes the closure-hoisting pattern across all streaming branches in `get_request_handler()`.
- **Readability:** Removes a nested function definition from the already-complex `app(request)` body, making the control flow clearer.

The quantitative benefit is small (raw async streaming is less common than SSE/JSONL), but the architectural value of completing the pattern is real.

---

### Implementation Complexity

**Very Low.**

The closure has no free variables and no route-static parameters. It can be hoisted to module level with zero signature changes:

```python
async def _async_stream_raw(
    async_gen: AsyncIterator[Any],
) -> AsyncIterator[Any]:
    async for chunk in async_gen:
        yield chunk
        await anyio.sleep(0)
```

Inside `get_request_handler()`, the call site becomes:

```python
gen = _async_stream_raw(gen)
```

This is a pure move — no logic changes, no behavioral changes, no public API changes. Risk of regression is negligible.

---

### Relation to Completed Work

**Direct continuation.**

This proposal is a natural follow-up to the streaming serialization optimization (PERF-001) because:
- Same file (`fastapi/routing.py`)
- Same function (`get_request_handler`)
- Same principle (eliminate per-request closure recreation)
- Same codebase precedent (`serialize_response`, `run_endpoint_function`, and the five already-hoisted streaming helpers)

PERF-001 moved the serialization helpers. PERF-002 moves the last remaining streaming helper.

---

### Validation Plan

1. All existing streaming tests should pass (including raw streaming tests).
2. All non-streaming tests should pass.
3. `mypy strict mode` should pass.
4. `ruff check` and `ruff format --check` should pass.
5. Optional: benchmark raw async streaming throughput before/after to confirm the ~160 bytes/request saving.

---

### Rejected Alternatives (Why Not Something Bigger?)

We investigated the surrounding implementation for larger bottlenecks:
- **`solve_dependencies`:** Called on every request, but it performs genuinely per-request work (dependency resolution). Not a closure-hoisting candidate.
- **`endpoint_ctx` creation:** Per-request because the path varies per request (mounted sub-apps). Cannot be cached.
- **Body parsing:** Genuinely per-request work.
- **`websocket_session` closures:** Genuinely per-request (create per-connection AsyncExitStacks). Cannot be hoisted.

`_async_stream_raw` is the only remaining per-request closure with route-static behavior in the request handling pipeline.

---

### Recommendation

**Proceed.** This is a low-risk, low-effort completion of the streaming closure-hoisting pattern. It delivers architectural consistency and a small but clean performance gain. The work fits naturally into the next maintenance cycle for `fastapi/routing.py`.
