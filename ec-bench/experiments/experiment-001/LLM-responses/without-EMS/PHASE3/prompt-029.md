model: kimi k2.5

Based on my review of the surrounding implementation, here is the technical proposal for the next optimization:
Ticket: Eliminate per-request inner app closure allocation in request_response
Suspected bottleneck
fastapi/routing.request_response (lines 126–155) defines a nested inner async def app inside the outer ASGI app on every HTTP request. This creates a new function object + closure tuple per request, capturing f (the wrapped endpoint) and request. The same structural anti-pattern exists in websocket_session (line 171), but the HTTP path is the high-volume one.
Why it deserves attention
1. Universal impact — unlike the streaming serialization closures fixed in the recent work, this closure is created on every HTTP request (streaming and non-streaming).
2. Same anti-pattern — it is structurally identical to the per-request closures that were just hoisted to factory scope in get_request_handler (_serialize_data, _serialize_sse_item, etc.).
3. Cumulative GC pressure — at high throughput (10k+ req/s), a ~200-byte function object + closure tuple per request adds up to several GB/hour of short-lived allocations, creating generational GC churn.
4. Engineering consistency — the codebase now applies the “factory-scope over per-request” principle to streaming serialization; the request/response wrapper is the remaining high-frequency allocation site.
Expected engineering benefit
- Modest but measurable reduction in per-request allocation overhead and GC pressure.
- Removes the last significant per-request function-object allocation in the standard HTTP hot path.
- Brings the request_response wrapper in line with the engineering principle established by the recent streaming optimization.
Implementation complexity: Low to Medium
The inner app logic can be extracted to a module-level helper (e.g., _handle_request_response) that accepts f, request, scope, receive, send as explicit parameters. wrap_app_handling_exceptions expects an ASGI-app signature; the helper can be wrapped with functools.partial (a lightweight C-level object) to satisfy that interface without creating a Python closure.
Key care points:
- Preserve exact AsyncExitStack ordering and scope key injection (fastapi_inner_astack, fastapi_function_astack).
- Preserve the response_awaited safety check and FastAPIError raise path.
- Verify exception handling, dependency teardown, and response streaming behavior remain correct.
- websocket_session shares the same pattern; the helper can likely be reused or mirrored, but the WebSocket path is lower priority per the previous evaluation.
Builds upon this task?
Yes — directly. This applies the exact same factory-scope principle (create function objects at route registration time, not per request) to the HTTP request/response wrapper, which is the next most frequent per-request allocation site after the streaming serialization closures that were just fixed.
Recommendation
Pursue this as a small follow-up PR. The change is localized to fastapi/routing.py, requires no public API changes, and is a natural continuation of the recent closure-extraction work.