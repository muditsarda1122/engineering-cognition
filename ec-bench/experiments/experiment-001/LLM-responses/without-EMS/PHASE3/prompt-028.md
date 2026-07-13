model: kimi k2.5

Here is the architectural evaluation comparing the WebSocket and HTTP execution paths.
1. The Architectural Approach Introduced
The recent change to get_request_handler() in fastapi/routing.py applied a factory-scope closure extraction pattern. Specifically, helper functions for response serialization (_serialize_data, _serialize_sse_item, _serialize_jsonl_item, etc.) and async stream wrapping (_async_stream_jsonl, _sse_with_checkpoints) were moved from inside the per-request async def app(request) closure to the outer get_request_handler() factory scope.
Engineering principle: Create function objects once at route registration time instead of re-creating them on every incoming request, eliminating per-request allocation overhead and reducing GC pressure.
2. HTTP vs. WebSocket Execution Models
Dimension	HTTP (get_request_handler)	WebSocket (get_websocket_app)
Lifecycle	Short-lived request/response	Long-lived persistent connection
Response handling	Framework serializes and sends Response objects	Endpoint directly calls websocket.send_text()/send_json()
Streaming	Heavy framework-managed streaming (SSE, JSONL)	Endpoint-managed iteration
Serialization	Complex: model validation, jsonable_encoder, Pydantic dump_json, field filtering	None at the framework level
Factory function	get_request_handler() → app(request)	get_websocket_app() → app(websocket)
ASGI wrapper	request_response() creates nested app per-request	websocket_session() creates nested app per-connection
3. Do the Same Engineering Principles Apply?
Partially, but with a crucial difference in impact.
Both request_response and websocket_session define a nested inner app closure on every invocation:
# HTTP (request_response) – per request
async def app(scope, receive, send):
    request = Request(...)
    async def app(scope, receive, send):  # <-- re-created per request
        ...

# WebSocket (websocket_session) – per connection
async def app(scope, receive, send):
    session = WebSocket(...)
    async def app(scope, receive, send):  # <-- re-created per connection
        async with AsyncExitStack() as request_stack:
            ...
            await func(session)
So the structural pattern (nested closure creation) exists in both paths. However, the motivating factor for the recent HTTP optimization was the complex, high-frequency serialization closures inside get_request_handler. The WebSocket path has no equivalent serialization pipeline because WebSocket endpoints do not return Response objects to be serialized by the framework.
The WebSocket inner closure is very lightweight: two AsyncExitStack context managers and a single await func(session) call.
4. Expected Engineering Benefit
Low to negligible.
- Request volume: HTTP endpoints typically handle thousands of short requests per second. WebSocket connections are long-lived and far fewer in number. A per-connection allocation is amortized over the entire connection lifetime.
- Closure complexity: The WebSocket inner closure does no serialization, no model validation, and no JSON encoding. It is ~4 lines of context manager bookkeeping.
- GC pressure: The objects allocated per WebSocket connection (WebSocket session, inner app function, two AsyncExitStack frames) are tiny compared to the HTTP streaming serialization path, which was creating multiple validation/serialization closures on every request, including non-streaming ones.
5. Estimated Implementation Complexity
Low, but not zero.
If one wanted to apply the principle consistently, websocket_session could be refactored to avoid the nested app closure. For example, the inner ASGI app could become a module-level helper that receives func and session as explicit parameters. However, this would:
1. Require careful handling of wrap_app_handling_exceptions, which expects an ASGI app signature.
2. Need test coverage to ensure AsyncExitStack ordering and exception handling remain correct.
3. Touch vendored code derived from Starlette, which adds maintenance friction when upstream changes.
6. Recommendation
I would not recommend pursuing this as a follow-up task.
Reasoning:
1. No equivalent hotspot: The HTTP optimization targeted a clear performance hotspot — per-request serialization closures for SSE/JSONL streaming. The WebSocket path has no such hotspot.
2. Negligible payoff: WebSocket connections are orders of magnitude fewer than HTTP requests, and the per-connection allocation is already minimal.
3. Risk/reward ratio: Changing websocket_session (which is vendored from Starlette and handles critical exception wrapping) for a theoretical micro-optimization is not justified.
4. Better follow-ups: If the team wants to extend the factory-scope pattern, a more valuable target would be the nested inner app in request_response (the HTTP ASGI wrapper), which shares the same structural pattern but is also hit on every request. However, even there, the closure is lightweight compared to the serialization helpers that were already fixed.
Conclusion: The factory-scope optimization is highly specific to the HTTP response serialization pipeline. The WebSocket execution model is architecturally different — it delegates communication to the endpoint and lacks the framework-level serialization stage where the optimization delivered its benefit. The cost of change outweighs the expected gain.