model: kimi k2.5

## Context Assembly

### Repository State
The existing benchmark suite (`tests/benchmarks/test_general_performance.py`) contains 16 benchmarks for standard request/response endpoints. It gates behind `--codspeed`, uses module-scoped `TestClient`, and implements warmup-then-benchmark. Zero streaming coverage existed.

### Streaming Serialization Changes
The refactoring eliminated per-request serialization closures (`_serialize_data`, `_serialize_sse_item`, `_serialize_item`, `_async_stream_jsonl`, `_sync_stream_jsonl`) and replaced them with module-level functions (`serialize_stream_data`, `serialize_sse_item`, `serialize_stream_item`, `async_stream_jsonl`, `sync_stream_jsonl`).

---

## Benchmark Extension

### New File: `tests/benchmarks/test_streaming_performance.py`

Created 12 benchmark cases covering JSONL and SSE streaming with varying item counts and serialization paths.

### Why These Benchmarks Were Selected

1. **Short streams (5 items)** — These benefit most from per-request allocation reduction. With few items, the per-request overhead (closure creation, setup/teardown) dominates total cost. This matches the "high-throughput short-lived streams" case from the cost significance threshold analysis (10,000+ req/s, <10 items each).

2. **Medium streams (50 items)** — These verify that per-item serialization overhead does not regress. With many items, per-item cost dominates and per-request overhead is amortized.

3. **With model vs bare** — Separates the Pydantic v2 `serialize_json` fast path (Rust-based, most common in production) from the `jsonable_encoder` + `json.dumps` fallback (used for bare dicts/lists).

4. **JSONL vs SSE** — Covers both streaming formats. SSE has additional formatting overhead and context manager lifecycle.

### What Behaviour They Measure

**Throughput benchmarks (8 tests):**
- `test_jsonl_short_with_model` — 5 items, Pydantic v2 `serialize_json` fast path
- `test_jsonl_short_bare` — 5 items, `jsonable_encoder` fallback
- `test_jsonl_medium_with_model` — 50 items, Pydantic v2 fast path
- `test_jsonl_medium_bare` — 50 items, fallback
- `test_sse_short_with_model` — 5 items, SSE format with Pydantic
- `test_sse_short_bare` — 5 items, SSE format bare
- `test_sse_medium_with_model` — 50 items, SSE with Pydantic
- `test_sse_medium_bare` — 50 items, SSE bare

**Allocation diagnostics (4 tests):**
- Use `tracemalloc` to measure peak memory growth across 100 identical requests
- Diagnostic-only (no strict assertions) because TestClient overhead dominates

### Expected Improvement

- **Short streams:** Measurable per-request latency reduction. For SSE: ~320 bytes/request eliminated (was ~640 for all closures, now ~320 for remaining per-request closures). For JSONL: ~160 bytes/request eliminated.
- **Medium/long streams:** No significant improvement expected — per-item serialization dominates. Benchmarks verify no regression.

### Limitations

1. **TestClient overhead:** `TestClient` adds ~700-1200 KiB per 100 requests of HTTP/ASGI processing overhead. This dwarfs the closure elimination savings (~16-32 KiB per 100 requests), making allocation benchmarks less sensitive than throughput benchmarks.
2. **No CodSpeed integration verified:** The `--codspeed` gate follows the existing pattern but actual dashboard upload was not tested.
3. **Static data:** All endpoints return the same data every request. Dynamic/complex real-world data may have different serialization costs.
4. **No cancellation benchmark:** The `anyio.sleep(0)` cancellation checkpoint is not exercised under load.
5. **Sequential execution:** Each benchmark runs one request at a time. Concurrent streaming requests may expose different allocation patterns.

---

## Verification

- **Benchmark file created:** `tests/benchmarks/test_streaming_performance.py` (12 tests)
- **JSONL endpoints:** Return `application/jsonl` with correct line counts (5 or 50)
- **SSE endpoints:** Return `text/event-stream` with correct `data:` counts (5 or 50)
- **Allocation tests:** Pass with diagnostic measurements
- **Follows existing patterns:** `--codspeed` gate, module-scoped fixture, warmup-then-benchmark
