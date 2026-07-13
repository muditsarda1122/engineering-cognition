model: kimi k2.5

Here is the roadmap proposal for a broader architectural improvement to the request execution pipeline.
Roadmap Proposal: Extract a Response Type Dispatcher from get_request_handler
Engineering problem being addressed
The app function inside get_request_handler (fastapi/routing.py, lines 523–778) is a monolithic ~255-line closure that violates the Single Responsibility Principle. It currently handles:
- Body parsing and file lifecycle management (lines 542–590, ~49 lines)
- Dependency resolution (lines 592–606, ~15 lines)
- Response dispatch via a large inline if/elif/else block (lines 608–768, ~160 lines) containing four distinct response-type implementations:
- SSE streaming (lines 609–695, ~87 lines, including 3 nested per-request closures)
- JSONL streaming (lines 696–712, ~17 lines)
- Raw streaming (lines 713–722, ~10 lines)
- Non-streaming standard responses (lines 723–768, ~46 lines)
- Error handling (lines 769–773)
The recent optimization moved serialization helpers (_serialize_data, _serialize_sse_item, _serialize_jsonl_item, etc.) to factory scope, but the dispatch decision and response-building logic remain inline and per-request. Adding a new response type requires modifying the core app function, increasing regression risk. The SSE branch alone contains 3 nested per-request closures (_sse_producer_cm, _producer, _keepalive_inserter) that are still re-created on every SSE request.
Proposed architectural direction
Introduce a formal Response Type Dispatcher abstraction. At route registration time, get_request_handler selects and binds the appropriate handler based on route metadata (is_sse_stream, is_json_stream, dependant.is_async_gen_callable, response model configuration). Each handler is a factory-scope callable that receives the solved dependency result and endpoint context, and returns a Response.
get_request_handler (factory scope)
├── body parsing
├── dependency resolution
├── DISPATCHER (factory-scope, selected at registration)
│   ├── SSE handler      → StreamingResponse (event-stream)
│   ├── JSONL handler    → StreamingResponse (application/jsonl)
│   ├── Raw handler      → actual_response_class(content=gen, ...)
│   └── Standard handler → Response or actual_response_class(content, ...)
└── error handling / return
Each handler encapsulates its own response-building logic, serialization strategy, and stream-wrapping behavior. The SSE handler, for example, would absorb the task-group keepalive logic and the checkpointed receive stream, keeping those concerns out of the core pipeline.
Expected long-term engineering benefits
1. Reduced monolithic complexity — The app function shrinks from ~255 lines to ~60 lines (body parsing + dependency resolution + single dispatcher call + error handling), making the core pipeline easier to reason about and maintain.
2. Independent testability — Each response-type handler can be unit-tested in isolation with mocked solved results, rather than exercising the entire app function.
3. Extensibility — Adding a new response type (e.g., NDJSON, multipart streaming) requires only registering a new handler, not modifying the central if/elif/else block.
4. Further optimization becomes natural — The SSE branch’s remaining per-request closures (_sse_producer_cm, _producer, _keepalive_inserter) can be hoisted to factory scope within the SSE handler itself, without touching the core pipeline.
5. Composability — The dispatcher pattern sets the stage for eventually extracting body parsing and dependency resolution into reusable pipeline stages, similar to how middleware stacks work.
Migration complexity: Medium
The change is localized to fastapi/routing.py and requires no public API changes. A phased approach is recommended:
- Phase 1 (low risk) — Extract each response-type block into a standalone private async function (same pattern already used for _serialize_data, _async_stream_jsonl, etc.). Replace the if/elif/else branches with calls to these functions. This is purely structural refactoring.
- Phase 2 — Introduce a dispatcher registry and bind the correct handler at route registration time (inside get_request_handler), eliminating the runtime branching entirely.
- Phase 3 — Within each handler, apply the factory-scope principle to any remaining per-request closures (notably the SSE task-group functions).
Each phase can be merged independently, with the existing test suite providing regression coverage.
Estimated implementation effort: Medium
The extraction work is localized and well-scoped. The primary risk is ensuring the dispatcher correctly preserves behavior for all response types, especially the SSE keepalive and cancellation semantics. Because the change is internal to fastapi/routing.py and the existing test coverage is substantial, the effort is manageable for a focused engineering sprint.
Conclusion
This proposal directly extends the "factory-scope over per-request" principle established by the recent serialization optimization. It addresses the root cause of the pipeline's monolithic complexity — the inline response dispatch block — rather than optimizing individual closures within it.