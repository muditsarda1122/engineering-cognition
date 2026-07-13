model: kimi k2.5

## WebSocket vs HTTP Streaming: Architectural Approach Evaluation

### Similarities and Differences Between HTTP and WebSocket Execution Models

**HTTP Execution Model (`get_request_handler`)**
- Creates a per-request `app(request)` closure that handles the full request/response lifecycle
- Includes complex branching: body parsing, dependency solving, response serialization
- For streaming endpoints (SSE, JSONL), previously contained per-request helper closures (`_serialize_data`, `_serialize_sse_item`, `_serialize_item`, `_async_stream_jsonl`, `_sync_stream_jsonl`) that were recreated on every request
- These helpers were hoisted to module-level functions, saving ~160-320 bytes/request
- Still retains genuinely per-request closures for SSE context managers (`_sse_producer_cm`, `_producer`, `_keepalive_inserter`, `_sse_with_checkpoints`) because they manage per-request state (memory streams, task groups)

**WebSocket Execution Model (`get_websocket_app`)**
- Creates a per-request `app(websocket)` closure
- Extremely simple: extracts `endpoint_ctx`, retrieves `async_exit_stack`, solves dependencies, then calls `await dependant.call(**solved_result.values)`
- No response serialization, no body parsing, no streaming response path
- No helper closures inside `app(websocket)` at all
- The outer `websocket_session` wrapper has nested closures, but they are also genuinely per-request (creating AsyncExitStacks for the WebSocket session)

**Key Difference:** The HTTP path has a rich response serialization pipeline with route-static parameters that were being captured in per-request closures. The WebSocket path has no such pipeline — once dependencies are solved, the endpoint itself owns the entire WebSocket lifecycle.

### Do the Same Engineering Principles Apply?

**The principle:** Hoist route-static helper closures to module-level (or outer scope) to eliminate per-request function object allocation.

**Verdict: No applicable targets exist in the WebSocket path.**

In `get_websocket_app()`, there are no route-static helper closures to hoist. The only operations inside `app(websocket)` are:
1. `endpoint_ctx` extraction — genuinely per-request (path varies per request)
2. `async_exit_stack` retrieval — genuinely per-request (scope-bound)
3. `solve_dependencies()` — genuinely per-request
4. `dependant.call()` — genuinely per-request

The `websocket_session` wrapper does have closures, but they create per-request `AsyncExitStack` instances — this state cannot be hoisted because it must be scoped to the individual WebSocket connection.

### Expected Engineering Benefit

**Negligible to zero.**

The WebSocket path is already minimal. There are no per-request function object allocations from helper closures to eliminate. Any optimization here would be at the level of:
- Avoiding `endpoint_ctx` dict creation (micro-optimization, ~72 bytes)
- Avoiding the `app(websocket)` closure itself (would require structural changes to how Starlette routes work)

Neither of these is in the same category as the HTTP streaming optimization, which eliminated 5 helper closures per request.

### Estimated Implementation Complexity

**Low, but for the wrong reasons.**

There is nothing meaningful to implement. The WebSocket path does not have the closure-recreation problem that the HTTP streaming path had. A developer could spend time searching for analogous patterns, but they would find none.

If one were to force an optimization (e.g., eliminating the `app(websocket)` closure by restructuring `get_websocket_app` to return a non-closure callable), the complexity would be **medium-to-high** because it would require changing how Starlette's `WebSocketRoute` expects its endpoint — and the benefit would still be negligible.

### Recommendation

**Do not pursue this as a follow-up task.**

The streaming serialization optimization was successful because:
1. HTTP streaming had clear per-request closure recreation (5 helpers × ~160 bytes)
2. The helpers were route-static (same behavior for every request to that route)
3. Hoisting to module-level was a pure move with zero behavioral change
4. The pattern followed existing codebase precedent (`serialize_response`, `run_endpoint_function`)

The WebSocket path lacks all four of these conditions:
1. No per-request helper closures exist
2. No route-static state is being recreated per-request
3. Nothing to hoist
4. The path is already structurally minimal

**Better follow-up tasks would be:**
- Extend the benchmark suite to include WebSocket connection benchmarks (currently missing)
- Profile high-concurrency WebSocket scenarios to identify actual bottlenecks (likely in `solve_dependencies` or `anyio` task scheduling, not in the route handler scaffolding)
- Investigate whether dependency resolution caching could reduce per-request overhead in WebSocket endpoints that share heavy dependencies
