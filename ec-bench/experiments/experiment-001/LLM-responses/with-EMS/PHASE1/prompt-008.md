model: kimi k2.5

Closure Recreation Investigation

Investigation Approach

Method: Runtime instrumentation via temporary source-code modification of fastapi/routing.py.

Rationale: The helper functions (_serialize_data, _serialize_sse_item, _sse_producer_cm) are defined inside async def app(request) -> Response (routing.py:398). Because they are nested inside a function that runs once per request, Python semantics dictate that def statements inside app() create new function objects on every invocation. The key question is whether this theoretical behavior manifests in practice for the actual FastAPI request handler.

Instrumentation added (and subsequently reverted):
1. A module-level list _closure_recreation_log: list = [] was added to fastapi/routing.py
2. After def _serialize_data(...): inside app(), the newly created function object was appended to the log
3. Similar capture points were added for _serialize_sse_item and _sse_producer_cm

The instrumentation was minimal (3 lines of capture code) and did not alter control flow or logic.

Experiment Script

A standalone script (closure_experiment.py) was created that:
1. Cleared the log before the experiment
2. Created a FastAPI app with two streaming endpoints:
   - /jsonl — returns AsyncIterable[Item] (JSONL streaming)
   - /sse — returns AsyncIterable[Item] (SSE streaming via default response_class inference)
3. Made 4 requests via TestClient:
   - Request 1: GET /jsonl
   - Request 2: GET /jsonl
   - Request 3: GET /sse
   - Request 4: GET /sse
4. After all requests, analyzed the captured function objects

What Was Measured

For each captured function instance, the experiment measured:
- Function object identity (id(func)) — whether a new Python function object was created
- Code object identity (id(func.__code__)) — whether the compiled bytecode was shared
- Closure cell identities (id(cell) for each cell in func.__closure__) — whether closure cells were recreated
- Closure cell variable names (func.__code__.co_freevars) — which variables each cell corresponds to
- Cell contents — the actual value captured in each cell

Observations

Total log entries captured: 4
All 4 requests captured _serialize_data. No _serialize_sse_item or _sse_producer_cm were captured because the /sse endpoint in the experiment was inferred as JSONL rather than SSE (the response_class default behavior for AsyncIterable is JSONL, not EventSourceResponse).

Function: _serialize_data
  Captured instances: 4 (one per request)
  All share same __code__: True (1 unique code object)
  All different function objects: True (4 unique out of 4)
  Closure cells per instance: 8
    Cell[0] 'endpoint_ctx': all different = True
    Cell[1] 'response_model_by_alias': all different = False
    Cell[2] 'response_model_exclude': all different = False
    Cell[3] 'response_model_exclude_defaults': all different = False
    Cell[4] 'response_model_exclude_none': all different = False
    Cell[5] 'response_model_exclude_unset': all different = False
    Cell[6] 'response_model_include': all different = False
    Cell[7] 'stream_item_field': all different = False

Key Observations

1. Function objects are recreated: All 4 _serialize_data instances have different id() values. A new function object is created on every request.

2. Code objects are shared: All 4 instances share the exact same __code__ object (same id). The compiled function body is reused — only the function wrapper (with its closure) is recreated.

3. Closure cells have MIXED behavior:
   - Per-request cells (endpoint_ctx) are recreated: different cell object on every request
   - Route-static cells (stream_item_field, response_model_*) are SHARED: the same cell object is reused across all requests

This is because _serialize_data captures endpoint_ctx from app()'s local scope (co_cellvars — created per-request), but captures stream_item_field and response_model_* from get_request_handler()'s scope (co_freevars — shared across all app() invocations).

Conclusion

The observations STRONGLY SUPPORT the hypothesis that repeated closure creation exists.

On every request to a streaming endpoint:
- A new _serialize_data function object is allocated (~72 bytes for the function object + 8 closure cells)
- The same __code__ object is reused (no bytecode recompilation)
- Per-request closure cells are recreated (endpoint_ctx)
- Route-static closure cells are shared (stream_item_field, response_model_*)

The recreation of function objects is not merely a theoretical concern — it is confirmed by direct object identity inspection. For high-throughput streaming endpoints (e.g., SSE with thousands of items), the per-request allocation of _serialize_data and its closure cells represents a small but non-zero overhead that could be eliminated by moving the function definition outside app() (to the get_request_handler scope) and parameterizing it with a per-request endpoint_ctx argument.

Limitations of the Experiment

1. Only _serialize_data was captured. _serialize_sse_item and _sse_producer_cm were not triggered because the test endpoint was inferred as JSONL, not SSE. To fully measure the closure recreation overhead for all streaming paths, the experiment would need endpoints that explicitly use EventSourceResponse and StreamingResponse.

2. The experiment did not measure memory allocation volume or GC pressure. It only confirmed object recreation via identity comparison. A follow-up experiment using tracemalloc or sys.getallocatedblocks() would quantify the actual memory impact.

3. The experiment used TestClient, which runs requests synchronously through the ASGI app. The behavior may differ slightly from a production ASGI server (e.g., uvicorn) due to event loop differences, but the Python closure semantics are identical.

4. The instrumentation required temporary source-code modification. A non-invasive alternative (e.g., bytecode instrumentation via sys.settrace or a custom import hook) would be preferable for repeated experiments, but was not necessary for this one-off investigation.