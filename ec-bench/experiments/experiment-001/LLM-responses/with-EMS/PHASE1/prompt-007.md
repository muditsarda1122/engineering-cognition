models: kimi k2.5

Benchmark Structure Analysis

File: tests/benchmarks/test_general_performance.py

1. Module-Level Setup
- Skips unless --codspeed is passed (line 11-15): pytest.skip(..., allow_module_level=True)
- Defines LARGE_PAYLOAD with 300 items, each with nested dicts and lists (lines 17-38)
- Defines Pydantic models: ItemIn, ItemOut, LargeIn, LargeOut (lines 49-68)
- Defines dependency chain: dep_a → dep_b (lines 41-46)
- Creates a single FastAPI app with 20+ endpoints (lines 70-181)

2. Fixture (line 184-187)
- @pytest.fixture(scope="module")
- Returns a TestClient context manager
- Module scope means the same client is reused across all benchmarks in the file, amortizing app startup cost

3. Warmup Pattern (lines 190-211)
Two helper functions implement the benchmark protocol:
- _bench_get(benchmark, client, path): makes a warmup GET request, asserts 200, then benchmarks a second GET
- _bench_post_json(benchmark, client, path, json): makes a warmup POST request, asserts 200, then benchmarks a second POST

The warmup request serves two purposes:
- Warms up Python's JIT/optimizer (if any) and any connection pooling
- Validates the endpoint works before benchmarking

The benchmark function captures only the second request, excluding warmup overhead.

4. Assertions
Every benchmark test asserts:
- status_code == 200
- body == expected_bytes (exact byte comparison)

Expected bytes are hardcoded for small payloads (e.g., b'{"name":"foo","value":123,"dep":42}') and computed via json.dumps() for large payloads (lines 313-319).

5. Benchmark Coverage Matrix
The existing benchmarks cover:
- Sync/async × dict/model × with/without response_model (small payload)
- Sync/async × large dict/model × with/without response_model (300 items)
- Sync/async receiving large payload (300 items)
- All 16 tests use standard request/response pattern (no streaming)

Streaming Benchmark Coverage: NONE
- No JSONL streaming benchmarks
- No SSE streaming benchmarks
- No raw StreamingResponse benchmarks
- No generator endpoint benchmarks

Evaluation: Does the benchmark suite adequately measure the streaming serialization optimization?

NO. The existing benchmark suite is entirely inadequate for measuring the streaming serialization pipeline.

Why:
1. The optimization I'm investigating centers on _serialize_data (routing.py:487) and its per-request vs route-static variable capture. _serialize_data is only created inside the app(request) closure for streaming endpoints. The existing benchmarks never trigger streaming serialization.

2. The two serialization paths in _serialize_data are:
   a) stream_item_field.validate() + stream_item_field.serialize_json() (routing.py:488-507) — Pydantic v2 Rust-based fast path
   b) jsonable_encoder() + json.dumps() (routing.py:508-510) — pure-Python fallback
   Neither path is exercised by the existing benchmarks because all benchmark endpoints return complete responses, not generators.

3. The benchmark suite only measures:
   - Request parsing/validation overhead
   - Dependency resolution overhead
   - Response serialization for complete payloads (via serialize_response at routing.py:293)
   - Pydantic model conversion overhead
   It does NOT measure per-item serialization overhead for generator endpoints.

4. Streaming endpoints have fundamentally different performance characteristics:
   - Each yielded item triggers _serialize_data independently
   - Generator iteration overhead (async vs sync, threadpool vs native)
   - Memory stream creation and task group management (for SSE)
   - Checkpoint overhead (anyio.sleep(0) for cancellation safety)
   - Keepalive timer overhead (for SSE)

Recommended Additional Benchmarks

1. JSONL Streaming Serialization Benchmark
   Purpose: Measure per-item serialization overhead for generator endpoints.
   - Small items (dicts): baseline for jsonable_encoder vs serialize_json
   - Large items (nested Pydantic models): measure Pydantic v2 fast path benefit
   - Vary item count (10, 100, 1000): measure linear scaling
   - Compare typed (AsyncIterable[Item]) vs bare (AsyncIterable): measure validation overhead vs Any shortcut
   Why: This directly measures the _serialize_data paths discovered in the closure analysis.

2. SSE Streaming Serialization Benchmark
   Purpose: Measure SSE overhead vs raw JSONL.
   - Same item shapes as JSONL benchmark
   - Measure format_sse_event() wrapping cost
   - Measure _sse_producer_cm setup cost (memory streams, task group)
   - Measure keepalive insertion overhead (_keepalive_inserter)
   Why: SSE uses a more complex pipeline than JSONL (producer/consumer with anyio.create_memory_object_stream and task groups).

3. Raw StreamingResponse Benchmark
   Purpose: Measure StreamingResponse path without FastAPI serialization.
   - Return StreamingResponse with pre-serialized bytes
   - Measure pure ASGI throughput for streaming chunks
   - Vary chunk size and count
   Why: Establishes the baseline overhead of the ASGI streaming layer itself, isolating it from FastAPI's serialization logic.

4. Streaming Cancellation Benchmark
   Purpose: Measure the overhead of anyio.sleep(0) checkpoints.
   - Benchmark generator with checkpoints vs without
   - The checkpoint mechanism (_sse_with_checkpoints at routing.py:613, _async_stream_jsonl at routing.py:645) adds a yield point between every item
   Why: The cancellation safety fix (fastapi#14680) added mandatory checkpoints. A benchmark would measure their performance cost and ensure they don't dominate for high-throughput generators.

5. Stream Item Validation Error Benchmark
   Purpose: Measure ResponseValidationError handling in streams.
   - Benchmark endpoint that yields mostly valid items with occasional invalid items
   - Measure cost of catching and propagating ResponseValidationError mid-stream
   Why: The validation error test (test_stream_json_validation_error.py) proves correctness but doesn't measure performance impact.

Where to Add Them
- New file: tests/benchmarks/test_streaming_performance.py
- Follow the same structure: module-level app, module-scoped TestClient fixture, warmup pattern, exact byte assertions
- Use @pytest.mark.parametrize for item count and endpoint type (async/sync, typed/bare, JSONL/SSE/raw)
- Run with --codspeed to get CI-visible results

Summary
The existing benchmark suite measures standard request/response performance well but has zero coverage for streaming endpoints. The streaming serialization pipeline (_serialize_data, _serialize_sse_item, _sse_producer_cm) represents a significant and growing part of FastAPI's functionality (JSONL, SSE, raw streaming), yet none of it is benchmarked. Adding streaming benchmarks would:
- Establish baselines for the two serialization paths (Pydantic v2 fast path vs jsonable_encoder fallback)
- Measure the overhead of the streaming infrastructure (checkpoints, keepalive, task groups)
- Prevent performance regressions in streaming code paths
- Quantify the benefit of the Pydantic v2 Rust-based serialize_json() vs pure-Python json.dumps() for per-item serialization