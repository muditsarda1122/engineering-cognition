model: kimi k2.5

Approach 1 — Lazy branch-guarded creation
Move def _serialize_data from the top of if not errors: into each streaming branch that actually calls it.
What it looks like:
# Current (routing.py:483-510)
if not errors:
    def _serialize_data(data): ...        # created for EVERY request
    
    if is_sse_stream:
        ...
    elif is_json_stream:
        ...

# Proposed
if not errors:
    if is_sse_stream:
        def _serialize_data(data): ...    # only when SSE
        def _serialize_sse_item(item): ...
    elif is_json_stream:
        def _serialize_data(data): ...    # only when JSONL
        def _serialize_item(item): ...
Why it preserves behavior: The branches are mutually exclusive (guarded by setup-time booleans is_sse_stream and is_json_stream). At most one _serialize_data executes per request. The body is identical, so serialization and error-reporting are unchanged.
Evaluation:
Dimension	Assessment
Implementation complexity	Very low (~10-line diff). No new abstractions, no signature changes.
Maintainability	Slightly reduced — _serialize_data body is duplicated in two places. If the validation/serialization logic changes, both copies must be updated. The duplication is small (~20 lines) and localized.
Compatibility	Perfect. No API changes, no new imports, no changes to get_request_handler signature.
Performance	Eliminates dead allocation for non-streaming requests (the common case). The investigation showed _serialize_data is created for all successful requests but only invoked by streaming branches. Streaming requests still pay the per-request allocation cost.
Error-reporting	Preserved identically. endpoint_ctx is still captured from app scope inside each branch, and ResponseValidationError at line 493 raises with the same context.
Ease of testing	Unchanged. The code is not extracted from get_request_handler, so it remains reachable only through integration tests.
Approach 2 — Factory-scope serialization helpers
Move _serialize_data, _serialize_sse_item, and _serialize_item from inside app to factory scope (inside get_request_handler but outside async def app). They naturally capture the 7 route-static variables via closure (created once). Only the per-request endpoint_ctx is passed explicitly.
What it looks like:
def get_request_handler(...):
    # --- Factory scope: created once per route ---
    
    def _serialize_data(data: Any, endpoint_ctx: EndpointContext | None = None) -> bytes:
        if stream_item_field:
            value, errors_ = stream_item_field.validate(data, {}, loc=("response",))
            if errors_:
                ctx = endpoint_ctx or EndpointContext()
                raise ResponseValidationError(
                    errors=errors_, body=data, endpoint_ctx=ctx,
                )
            return stream_item_field.serialize_json(
                value,
                include=response_model_include,
                exclude=response_model_exclude,
                by_alias=response_model_by_alias,
                exclude_unset=response_model_exclude_unset,
                exclude_defaults=response_model_exclude_defaults,
                exclude_none=response_model_exclude_none,
            )
        else:
            return json.dumps(jsonable_encoder(data)).encode("utf-8")
    
    def _serialize_sse_item(item: Any, endpoint_ctx: EndpointContext | None = None) -> bytes:
        if isinstance(item, ServerSentEvent):
            # ... SSE logic ...
            return format_sse_event(...)
        else:
            return format_sse_event(
                data_str=_serialize_data(item, endpoint_ctx=endpoint_ctx).decode("utf-8")
            )
    
    def _serialize_item(item: Any, endpoint_ctx: EndpointContext | None = None) -> bytes:
        return _serialize_data(item, endpoint_ctx=endpoint_ctx) + b"\n"
    
    # --- Per-request app ---
    async def app(request: Request) -> Response:
        # ...
        if not errors:
            if is_sse_stream:
                # _serialize_sse_item is reused from factory scope
                ...
            elif is_json_stream:
                # _serialize_item is reused from factory scope
                ...
Why it preserves behavior: The route-static variables (stream_item_field, response_model_include, etc.) are still captured by closure — but now at factory time instead of per-request. The only difference is endpoint_ctx, which is passed explicitly. The default endpoint_ctx=None preserves the ctx = endpoint_ctx or EndpointContext() logic at line 493.
Evaluation:
Dimension	Assessment
Implementation complexity	Medium (~40-50 lines changed). Requires moving three function definitions and threading endpoint_ctx through the call chain. No new abstractions.
Maintainability	Improved. Explicit parameter passing makes the data flow visible. A reader can see exactly which variables are per-request (endpoint_ctx) vs. route-static (captured by closure). The transitive capture chain is broken into explicit parameter plumbing.
Compatibility	Excellent. Preserves the functional closure style of FastAPI. The app function remains an ASGI-compatible coroutine. No changes to public API.
Performance	Eliminates per-request creation of the three most expensive helpers (_serialize_data with 8 closure cells, _serialize_sse_item, _serialize_item). The micro-benchmark showed a ~2.4× overhead for nested-def creation; this removes that overhead entirely for the serialization path. The streaming infrastructure helpers (_sse_producer_cm, _async_stream_jsonl) remain inside app because they need per-request variables like gen and sse_aiter that cannot be hoisted.
Error-reporting	Preserved. endpoint_ctx is passed explicitly to _serialize_data, which still executes endpoint_ctx or EndpointContext() at the validation-error path.
Ease of testing	Improved. _serialize_data is now a standalone function at factory scope. In tests, you can instantiate it by calling get_request_handler with mock parameters and then call _serialize_data directly with fake data and endpoint_ctx.
Approach 3 — Class-based serializer with factory-time instantiation
Encapsulate the streaming serialization logic in a class whose instance is created once at factory time. Methods accept per-request state as arguments.
What it looks like:
class _StreamingSerializer:
    def __init__(
        self,
        stream_item_field: ModelField | None,
        response_model_include: IncEx | None,
        # ... other route-static params ...
    ):
        self.stream_item_field = stream_item_field
        self.response_model_include = response_model_include
        # ...
    
    def serialize(self, data: Any, endpoint_ctx: EndpointContext | None = None) -> bytes:
        if self.stream_item_field:
            value, errors_ = self.stream_item_field.validate(data, {}, loc=("response",))
            if errors_:
                ctx = endpoint_ctx or EndpointContext()
                raise ResponseValidationError(errors=errors_, body=data, endpoint_ctx=ctx)
            return self.stream_item_field.serialize_json(
                value,
                include=self.response_model_include,
                # ...
            )
        else:
            return json.dumps(jsonable_encoder(data)).encode("utf-8")
    
    def serialize_sse(self, item: Any, endpoint_ctx: EndpointContext | None = None) -> bytes:
        # ...
    
    def serialize_jsonl(self, item: Any, endpoint_ctx: EndpointContext | None = None) -> bytes:
        return self.serialize(item, endpoint_ctx) + b"\n"
Factory usage:
def get_request_handler(...):
    serializer = _StreamingSerializer(
        stream_item_field=stream_item_field,
        response_model_include=response_model_include,
        # ...
    )
    
    async def app(request):
        # ...
        if is_sse_stream:
            # use serializer.serialize_sse(...)
        elif is_json_stream:
            # use serializer.serialize_jsonl(...)
Why it preserves behavior: Same as Approach 2 — route-static variables are bound once at factory time via __init__, and endpoint_ctx is passed per-request. The error path is identical.
Evaluation:
Dimension	Assessment
Implementation complexity	Medium-high (~60-70 lines). New class with 3+ methods. Requires updating all streaming branches to call serializer.serialize_* instead of helper functions.
Maintainability	High for extensibility. Adding a new streaming format (e.g., NDJSON, MessagePack) means adding a method to the class. All serialization logic is centralized. However, it introduces an OOP pattern into a codebase that is predominantly functional-closure style.
Compatibility	Moderate. The class is a local implementation detail (private, prefixed with _), so it doesn't affect the public API. But it shifts the internal coding style from closures to OOP, which may be unfamiliar to contributors working on routing.py.
Performance	Same allocation reduction as Approach 2 (one object created at factory time). Method dispatch adds a tiny overhead (~1-2 ns per call in CPython) that is negligible compared to serialization.
Error-reporting	Preserved. endpoint_ctx is passed as a method argument.
Ease of testing	Excellent. The class is independently instantiable. You can create a _StreamingSerializer with mock ModelField objects and test each method in isolation.
Comparison
Approach	Lines changed	Dead alloc eliminated	Streaming alloc eliminated	Style shift	Testability
1. Lazy branch-guarded	~10	Yes (non-streaming)	No	None	Unchanged
2. Factory-scope helpers	~50	Yes (non-streaming)	Yes (serialization helpers)	Minimal	Improved
3. Class-based serializer	~70	Yes (non-streaming)	Yes (serialization helpers)	Moderate (OOP)	Excellent
Recommendation
I recommend Approach 2: Factory-scope serialization helpers.
Justification:
1. It hits the sweet spot of the trade-off triangle. It eliminates the most expensive per-request allocations (the three serialization helpers with 8, 1, and 1 closure captures respectively) with a moderate, reviewable diff (~50 lines). Approach 1 is too small a win (streaming requests still pay the cost). Approach 3 is over-engineered for the current codebase's functional style.
2. It preserves FastAPI's architectural style. FastAPI is built on closures — get_request_handler is a factory, request_response is a decorator, solve_dependencies returns a SolvedDependency dataclass. Approach 2 keeps the functional closure pattern; it simply moves the closure creation from per-request to factory-time. Approach 3 introduces an OOP pattern that would be stylistically inconsistent.
3. It improves maintainability without harming readability. Making endpoint_ctx an explicit parameter (instead of an implicit closure capture) makes the data flow visible. A reader can now see exactly which variables cross the factory→request boundary. The transitive capture chain is broken into explicit parameter plumbing, which reduces the "where did this variable come from?" debugging problem.
4. It has zero risk to error-reporting. The ResponseValidationError path with endpoint_ctx or EndpointContext() is preserved because endpoint_ctx is still supplied at the call site inside app(request).
5. It improves testability. _serialize_data becomes a standalone factory-scope function. You can call get_request_handler with mock ModelField and Dependant objects, get back the app closure, and directly invoke _serialize_data with fake data and a controlled endpoint_ctx. This is impossible today because _serialize_data is a local variable inside app.
Implementation path: If the team wants an immediate zero-risk win first, Approach 1 (lazy branch-guarded) can be merged as a ~10-line hotfix. Then Approach 2 can follow as the structural improvement.