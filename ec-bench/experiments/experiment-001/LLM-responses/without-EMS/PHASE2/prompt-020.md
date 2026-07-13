model: kimi k2.5

Done. I created tests/benchmarks/test_streaming_performance.py with four streaming benchmark cases. ruff check passes on the new file. All existing tests pass.
Why these benchmarks were selected
The factory-scope abstraction eliminates per-request function-object allocation for streaming endpoints. The benefit is most visible when:
1. Many requests hit streaming endpoints — the per-request closure creation cost is paid on every request
2. Each stream yields many items — serialization helpers are called repeatedly per request, amplifying the benefit of avoiding closure recreation
3. Both async and sync generators are used — they exercise different code paths (_async_stream_jsonl vs _sync_stream_jsonl, iterate_in_threadpool for sync SSE)
The existing benchmark suite (test_general_performance.py) covers sync/async validated endpoints and large payload serialization, but it has no streaming coverage at all. The new benchmarks fill this gap and target the exact execution paths modified by the refactor.
What behaviour they measure
Benchmark	Endpoint	Items	Factory-scope helpers exercised
test_jsonl_async_stream	GET /jsonl/async	100	_serialize_data, _serialize_jsonl_item, _async_stream_jsonl
test_jsonl_sync_stream	GET /jsonl/sync	100	_serialize_data, _serialize_jsonl_item, _sync_stream_jsonl
test_sse_async_stream	GET /sse/async	100	_serialize_data, _serialize_sse_item, _sse_with_checkpoints, _sse_producer_cm
test_sse_sync_stream	GET /sse/sync	100	_serialize_data, _serialize_sse_item, _sse_with_checkpoints, _sse_producer_cm, iterate_in_threadpool
Each benchmark uses the CodSpeed benchmark fixture (same pattern as the existing suite) to measure the full request/response cycle via TestClient. The 100-item payload ensures the serialization path dominates the measurement, making closure allocation overhead detectable.
The response model (StreamItem Pydantic model) is included to exercise stream_item_field validation in _serialize_data, matching real-world usage where streaming endpoints validate items against a schema.
What improvement is expected
Before the refactor: every request to a streaming endpoint recreated 3–4 closure objects inside app():
- _serialize_data (or _serialize_item)
- _serialize_sse_item (SSE only)
- _async_stream_jsonl / _sync_stream_jsonl / _async_stream_raw
- _sse_with_checkpoints (SSE only)
After the refactor: these functions are created once at route registration time. The per-request cost is reduced to zero closure allocations, replacing them with simple function calls with explicit parameter passing.
For CodSpeed benchmarks measuring request latency, the expected improvement is modest as a percentage of total request time (HTTP parsing, routing, dependency resolution, and Pydantic serialization dwarf function allocation). However:
- The benchmark establishes a baseline for streaming endpoint performance
- It provides regression detection — if a future change accidentally reintroduces per-request closures, the benchmark will show a latency increase
- It makes the optimization objective measurable in CI, rather than relying on theoretical claims about closure overhead
The investigate_closure_creation.py script from earlier sessions already demonstrated that nested def creates ~2× overhead in a tight micro-benchmark. The endpoint-level benchmarks translate that finding into a real-world measurement.
Limitations
1. TestClient consumes the entire response — the benchmark measures server-side processing plus client-side buffer accumulation. It cannot isolate just the serialization phase. A more targeted micro-benchmark (calling _serialize_data directly in a loop) would show larger relative improvements, but the private helpers are inaccessible outside get_request_handler.
2. Benchmark skip without --codspeed — like the existing suite, these tests are skipped unless CodSpeed is enabled. This means the benchmarks are not validated in normal CI runs. The smoke-test script I ran independently verified endpoint correctness, but benchmark execution requires the CodSpeed plugin.
3. No memory-allocation benchmark — the core benefit is reduced object allocation, not just latency. CodSpeed measures wall-clock time. A companion benchmark using tracemalloc or pytest-memray would more directly measure the allocation reduction.
4. No keepalive/cancellation stress test — the SSE benchmarks use TestClient, which does not simulate slow consumers or connection drops. The _sse_producer_cm keepalive timer and task group cancellation paths are not stressed under load. Separate stress tests (using anyio timeout injection) would be needed to validate those paths.