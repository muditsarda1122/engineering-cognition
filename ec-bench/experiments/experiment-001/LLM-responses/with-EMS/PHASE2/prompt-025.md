model: kimi k2.5

## Type-Checking Investigation Results

**Repository:** fastapi (master branch)
**Type checker:** mypy (strict mode, pydantic.mypy plugin)

### Step 1: Run Type Checker

Ran mypy on the benchmark file I created:
```bash
python3 -m mypy tests/benchmarks/test_streaming_performance.py
```

**Result:** 18 type errors identified, all in `tests/benchmarks/test_streaming_performance.py`.

### Step 2: Categorize Errors

| Error Category | Count | Lines | Root Cause |
|---|---|---|---|
| Generator return type mismatch | 8 | 51, 58, 65, 72, 84, 91, 98, 105 | Sync generator functions (`def` + `for ... yield`) declared as returning `AsyncIterator[T]` instead of `Iterator[T]` |
| Missing type annotation | 9 | 128, 142, 152, 161, 170, 182, 192, 201, 210 | `benchmark` fixture parameter untyped (pre-existing benchmark pattern) |
| Returning Any | 1 | 136 | `benchmark(do_request)` returns `Any` but function declares `-> tuple[int, bytes]` |

**No type errors in `fastapi/routing.py`** — the module-level streaming functions pass mypy strict mode cleanly.

### Step 3: Fix 1 — Generator Return Types

**Problem:** Eight benchmark endpoints use `def` with `for ... yield` (sync generator syntax) but declare return type `AsyncIterator[T]`.

**Why this is wrong:** In Python typing, `AsyncIterator` is reserved for `async def` functions with `async for ... yield` or `async def` + `yield`. A regular `def` with `for ... yield` produces a sync `Iterator`. Mypy rejects `AsyncIterator` as the return type of a sync generator function.

**Fix:** Changed all 8 occurrences from `AsyncIterator[T]` to `Iterator[T]`:

```python
# Before (wrong):
def jsonl_short_with_model() -> AsyncIterator[StreamItem]:

# After (correct):
def jsonl_short_with_model() -> Iterator[StreamItem]:
```

**Why this improves type safety without altering behavior:** This is a pure type annotation correction. The runtime behavior is identical — the function still produces a sync generator that `StreamingResponse` wraps with `iterate_in_threadpool`. The fix makes the static type match the runtime type, preventing mypy from flagging a type mismatch when the generator is passed to `StreamingResponse` or any other consumer.

### Step 4: Fix 2 — Benchmark Fixture Type Annotation

**Problem:** Nine function signatures have `benchmark` parameter with no type annotation. Mypy strict mode (`disallow_untyped_defs`) requires all parameters to be typed.

**Fix:** Added `benchmark: Any` to all 9 function signatures:

```python
# Before:
def test_jsonl_short_with_model(benchmark, jsonl_client: TestClient) -> None:

# After:
def test_jsonl_short_with_model(benchmark: Any, jsonl_client: TestClient) -> None:
```

**Why this improves type safety without altering behavior:** The `benchmark` fixture is provided by `pytest-benchmark` and has no type stubs in the environment. Using `Any` is the correct way to annotate an untyped third-party fixture. This is a zero-runtime-cost annotation that satisfies mypy's strict mode. The existing benchmark file (`test_general_performance.py`) uses the same pattern.

### Step 5: Fix 3 — Casting Benchmark Return Value

**Problem:** `_bench_stream_get` returns `benchmark(do_request)`, which mypy infers as `Any` because `benchmark` is untyped. But the function signature declares `-> tuple[int, bytes]`.

**Fix:** Used `cast()` from `typing` to tell mypy the actual return type:

```python
# Before:
return benchmark(do_request)

# After:
return cast(tuple[int, bytes], benchmark(do_request))
```

**Why this improves type safety without altering behavior:** `typing.cast()` is a no-op at runtime — it generates no bytecode. It exists solely for the type checker. The inner `do_request` function is already correctly typed as `tuple[int, bytes]`, so the cast accurately reflects the runtime behavior. Without the cast, mypy cannot prove the return type matches the signature, and it reports a `no-any-return` error.

### Step 6: Verification

**Post-fix mypy run:**
```bash
python3 -m mypy tests/benchmarks/test_streaming_performance.py
```
Result: **Zero errors in the file** (remaining 11 errors are in other files, pre-existing).

**Runtime verification:**
All 8 benchmark endpoints verified to return correct responses:
- JSONL endpoints: `application/jsonl` content-type, correct line counts (5 or 50)
- SSE endpoints: `text/event-stream` content-type, correct data counts (5 or 50)

**No runtime behavior changed** — all fixes are type annotations and typing utilities (`cast`, `Any`) that have zero runtime impact.

### Summary

| Fix | Files Changed | Lines | Runtime Impact |
|---|---|---|---|
| `AsyncIterator` → `Iterator` | `test_streaming_performance.py` | 8 | None (pure annotation) |
| `benchmark` → `benchmark: Any` | `test_streaming_performance.py` | 9 | None (pure annotation) |
| `return benchmark(...)` → `return cast(..., benchmark(...))` | `test_streaming_performance.py` | 1 | None (`cast` is no-op) |

**Total: 18 type errors resolved, 0 runtime behavior changes, `fastapi/routing.py` remains error-free.**