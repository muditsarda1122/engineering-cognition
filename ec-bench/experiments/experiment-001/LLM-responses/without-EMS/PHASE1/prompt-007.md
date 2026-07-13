model: kimi k2.5

Fixture
@pytest.fixture(scope="module")
def client() -> Iterator[TestClient]:
    with TestClient(app) as client:
        yield client
Module-scoped TestClient fixture (line 184). The app and client are created once per module, not per test. This avoids connection-setup overhead so that benchmarks measure the endpoint logic, not HTTP stack initialization.
TestClient
Benchmarks use fastapi.testclient.TestClient (line 8). The app under test is a single FastAPI instance (line 70) wired with ~20 routes covering sync/async, dict/model return, with/without response_model, and large/small payload variants. All routes are pre-wired at import time, so the benchmark measures the runtime request path (FastAPI.__call__ → router → get_request_handler → solve_dependencies → endpoint → serialize_response), not route registration.
Warmup pattern
Two helper functions enforce a mandatory warmup:
def _bench_get(benchmark, client: TestClient, path: str) -> tuple[int, bytes]:
    warmup = client.get(path)          # ← one real request before benchmarking
    assert warmup.status_code == 200   # ← assertion on warmup
    def do_request() -> tuple[int, bytes]:
        response = client.get(path)
        return response.status_code, response.content
    return benchmark(do_request)        # ← pytest-benchmark measures only do_request
The same pattern is used in _bench_post_json (lines 201-211). The warmup request:
- triggers lazy middleware-stack construction (build_middleware_stack)
- warms the endpoint's code path through CPU caches
- verifies the route is wired correctly before timing it
Assertions
Every benchmark test (lines 214-399) asserts two things:
1. status_code == 200 — the request didn't error
2. body == expected_bytes — the response was correctly serialized
The expected payload for large responses is pre-computed via _expected_large_payload_json_bytes() (line 313), using json.dumps with ensure_ascii=False, separators=(",", ":") to match Starlette's JSON encoding.
Streaming benchmark coverage today
There is none. The only benchmark file in the repository is tests/benchmarks/test_general_performance.py, and it contains zero streaming endpoints. All 24 test functions exercise standard request/response paths:
Pattern
Sync GET/POST, dict/model, with/without response_model
Async GET/POST, dict/model, with/without response_model
Large payload receive
Large payload return
No StreamingResponse, no AsyncIterable return types, no SSE, no JSONL.
Does the existing suite adequately measure the optimization we're investigating?
No. The optimization path we've been tracing — the factory pattern in get_request_handler, the closure capture mechanism, and especially the streaming branches (_serialize_data, _serialize_sse_item, the anyio.sleep(0) cancellation checkpoints) — is entirely unmeasured.
Specifically, the current benchmarks miss:
1. Closure capture overhead — The benchmarks use response_model and Depends, so they do exercise get_request_handler and solve_dependencies. But they do not isolate the difference between:
- A route with 7 response-model serialization parameters (high closure capture) vs. a route with 0
- The _serialize_data fast path (stream_item_field set → Pydantic serialize_json) vs. the fallback (jsonable_encoder + json.dumps)
2. Streaming serialization paths — The large-payload benchmarks return a pre-built LargeOut model or a dict. They do not exercise:
- The _serialize_data inner function that validates each item individually against stream_item_field
- The _serialize_sse_item branch that handles ServerSentEvent special-casing
- The _serialize_item + json.dumps JSONL path
3. Cancellation checkpoint overhead — The anyio.sleep(0) injected at routing.py:650 and :681 adds a per-item event-loop yield. For fast generators (the cancellation bug scenario), this is the difference between hanging and being cancellable. The benchmarks do not measure the latency of this checkpoint for high-throughput streaming.
Recommended additional benchmark coverage
I would add a new file, tests/benchmarks/test_streaming_performance.py, with at least these 6 benchmark functions:
Benchmark	What it measures
test_jsonl_stream_small_items	AsyncIterable[Item] returning 100 small Pydantic items, with response_model
test_jsonl_stream_large_items	AsyncIterable[LargeItem] returning 100 large Pydantic items
test_jsonl_stream_bare_type	AsyncIterable (no generic), returning 100 dicts
test_sse_stream_small_items	AsyncIterable[Item] with response_class=EventSourceResponse
test_raw_stream_no_serialization	StreamingResponse yielding raw bytes, no validation
test_sync_vs_async_jsonl_stream	Compare sync Iterable[Item] vs. async AsyncIterable[Item]
Each should follow the same structure as test_general_performance.py:
- Module-scoped TestClient fixture
- _bench_stream helper with single-request warmup
- assert status_code == 200 and body content assertions
- Run under --codspeed (the existing skip logic at lines 11-15)
This would give the team a baseline to measure the exact cost of the streaming serialization layer and to catch regressions in the factory/closure patterns we've been analyzing.