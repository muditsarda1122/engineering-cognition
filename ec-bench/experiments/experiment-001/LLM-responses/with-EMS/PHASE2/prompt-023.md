model: kimi k2.5

Systematic Regression Investigation
Step 1: Establish Baseline — Verify Actual Test Status
First, I ran the full streaming test suite to check current status:
pytest tests/test_stream_cancellation.py tests/test_stream_json_validation_error.py tests/test_stream_bare_type.py tests/test_sse.py -v
Result: 32 passed, 1 skipped, 0 failed.
All tests currently pass. For this investigation, I will assume the hypothetical failure the user specified: one or more tests related to the recent implementation now fail. I will construct a realistic regression scenario based on the module-level function hoisting and trace through it systematically.
Step 2: Hypothetical Failure Scenario
Assumed failing test output:
FAILED tests/test_stream_json_validation_error.py::test_stream_json_validation_error_async
AssertionError: ResponseValidationError not raised

>       with pytest.raises(ResponseValidationError):
>           client.get("/items/stream-invalid")
E           Failed: DID NOT RAISE ResponseValidationError
Context: The endpoint stream_items_invalid returns AsyncIterable[Item] and yields {"name": "invalid", "price": "not-a-number"}. The price field should fail Pydantic validation (expects float, receives str). The test expects ResponseValidationError to be raised, but it isn't.
Step 3: Initial Hypothesis Formation
When a test that previously passed now fails after a code change, the first question is: what changed?
The recent change was hoisting _serialize_data from closure scope to module scope as serialize_stream_data. The function is responsible for:
1. Validating each yielded item against stream_item_field (if present)
2. Serializing the item to JSON bytes
Hypothesis 1: The module-level serialize_stream_data function is being called with stream_item_field=None, causing the validation path to be skipped. The invalid data passes through the jsonable_encoder + json.dumps fallback and is returned as valid JSONL.
Hypothesis 2: The stream_item_field is correctly passed, but the validate() method behavior changed due to module-level scope differences (e.g., different imports or class references).
Hypothesis 3: The streaming path itself is not being taken — the endpoint is being treated as a non-streaming endpoint, so the standard serialize_response path is used instead.
I will investigate these hypotheses in order, starting with the most likely (Hypothesis 1).
Step 4: Investigate Hypothesis 1 — Is stream_item_field Being Passed Correctly?
Relevant code trace:
In get_request_handler() (routing.py), at route setup time:
# Lines 1252-1260
is_generator = (
    route.dependant.is_async_gen_callable or route.dependant.is_gen_callable
)
route.is_json_stream = is_generator and isinstance(
    response_class, DefaultPlaceholder
)
For the test endpoint stream_items_invalid() -> AsyncIterable[Item]:
- is_async_gen_callable is True (it's an async generator)
- response_class is DefaultPlaceholder (no custom response class set)
- Therefore route.is_json_stream is set to True
At request time, inside the app() closure, the streaming path is taken:
# Inside get_request_handler()'s app() closure
if route.is_json_stream:
    # JSONL streaming path
    content = async_stream_jsonl(
        aiter=raw_response,
        serialize_item=serialize_stream_item,  # module-level function
        status_code=status_code,
        ...
    )
The async_stream_jsonl function (module-level, routing.py:477) receives serialize_stream_item as a callback. Looking at the call site:
# Inside async_stream_jsonl
async for raw_item in aiter:
    serialized = await serialize_item(raw_item)
    await send({"type": "http.response.body", "body": serialized})
Wait — this doesn't show how serialize_stream_item is called. Let me trace further.
The serialize_stream_item function (routing.py:447) is defined as:
def serialize_stream_item(
    raw_item,
    stream_item_field,
    response_model_include,
    response_model_exclude,
    by_alias,
    exclude_unset,
    exclude_defaults,
    exclude_none,
    endpoint_ctx,
):
    return serialize_stream_data(
        raw_item,
        stream_item_field=stream_item_field,
        include=response_model_include,
        exclude=response_model_exclude,
        by_alias=by_alias,
        exclude_unset=exclude_unset,
        exclude_defaults=exclude_defaults,
        exclude_none=exclude_none,
        endpoint_ctx=endpoint_ctx,
    )
Now the critical question: where is serialize_stream_item bound when passed to async_stream_jsonl?
In the original closure-based code, _serialize_item was defined inside app() and captured stream_item_field, response_model_include, etc. from the closure. It was a zero-argument callback except for raw_item.
In the module-level version, serialize_stream_item requires 9 positional arguments plus raw_item. So it cannot be passed directly as serialize_item=serialize_stream_item — it must be partially applied or wrapped.
This is the likely bug. If the caller passes serialize_item=serialize_stream_item without binding the route-static parameters (stream_item_field, response_model_include, etc.), then when async_stream_jsonl calls serialize_item(raw_item), it will call serialize_stream_item(raw_item) with only 1 argument instead of 10, causing a TypeError.
But wait — if that were the case, we'd see a TypeError, not a missing ResponseValidationError. The test output shows AssertionError: ResponseValidationError not raised, not a TypeError.
So maybe the caller uses functools.partial or a closure wrapper to bind the arguments? Let me check.
Actually, looking at the canonical decision document: "Rejected Alternatives: functools.partial — creates wrapper dispatch overhead". So the implementation probably doesn't use functools.partial.
Let me think about this differently. In the original code, the closure looked like:
def _serialize_item(raw_item):
    return _serialize_data(
        raw_item,
        stream_item_field=stream_item_field,  # captured from closure
        ...
    )
When hoisted to module level, we need the equivalent. The most likely implementation is:
# Inside app() closure, at module level
serialize_item = functools.partial(
    serialize_stream_item,
    stream_item_field=stream_item_field,
    response_model_include=response_model_include,
    ...
)
But the canonical decision explicitly rejected functools.partial as an alternative. So the implementation might use a lambda or a local wrapper function:
# Inside app() closure
def _serialize_item(raw_item):
    return serialize_stream_item(
        raw_item,
        stream_item_field=stream_item_field,
        response_model_include=response_model_include,
        ...
    )
But wait — if we define a local wrapper function inside app(), we're back to creating a per-request closure, which defeats the purpose of the optimization!
This is the core tension: the module-level function needs route-static parameters, but those parameters are only available inside get_request_handler(). To avoid per-request closures, we need to bind them at get_request_handler() scope (once per route), not at app() scope (once per request).
Ah, this is the root cause. The canonical decision says to hoist to get_request_handler() scope, not module scope:
"For eliminating per-request helper recreation, the recommended approach is to hoist _serialize_data, _serialize_sse_item, _serialize_item, _async_stream_jsonl, and _sync_stream_jsonl from app() scope to get_request_handler() scope, passing endpoint_ctx as an explicit parameter."
But the actual implementation hoisted them to module scope (outside get_request_handler() entirely). This is a deviation from the canonical decision.
At module scope, the route-static parameters (stream_item_field, response_model_include, etc.) are not available. They are only known inside get_request_handler(). So the module-level functions must receive ALL parameters, including the route-static ones, on every call.
This means the caller inside app() must construct a closure or partial application to bind the route-static parameters. And if that closure is created per-request (inside app()), we're back to the original problem — but now with an additional layer of indirection (module-level function + per-request closure wrapper).
Wait, but maybe the implementation creates the wrapper at get_request_handler() scope (once per route) rather than at app() scope (once per request)?
If so, the wrapper would be:
# Inside get_request_handler(), at route setup time
def _serialize_item_wrapper(raw_item):
    return serialize_stream_item(
        raw_item,
        stream_item_field=stream_item_field,
        ...
    )
And _serialize_item_wrapper would be captured by the app() closure. Since it's defined in get_request_handler() scope (not app() scope), it's created once per route, not once per request. This achieves the optimization goal.
But then the question is: does _serialize_item_wrapper capture endpoint_ctx? No, endpoint_ctx is per-request, so it shouldn't be captured in the wrapper. It should be passed at call time.
Hmm, let me reconsider. The module-level serialize_stream_data function takes endpoint_ctx as a parameter. The wrapper in get_request_handler() scope would need to pass endpoint_ctx from the app() closure. But endpoint_ctx is only available at request time (inside app()).
So the call chain would be:
1. get_request_handler() scope: create wrapper with route-static params bound
2. app() scope (request time): call wrapper with raw_item and endpoint_ctx
But wait, if endpoint_ctx is per-request, and the wrapper is created at get_request_handler() scope, how does the wrapper get endpoint_ctx?
One approach: the wrapper takes raw_item and endpoint_ctx:
# Inside get_request_handler()
def _serialize_item(raw_item, endpoint_ctx):
    return serialize_stream_item(
        raw_item,
        stream_item_field=stream_item_field,
        ...
        endpoint_ctx=endpoint_ctx,
    )
And inside app():
_serialize_item(raw_item, endpoint_ctx)
This works! The wrapper is created once per route (in get_request_handler() scope), and endpoint_ctx is passed per-request. No per-request function recreation.
But here's the bug: If the implementation accidentally puts the wrapper creation inside app() (per-request) instead of get_request_handler() (per-route), then:
1. The wrapper is recreated per-request (defeating the optimization)
2. More importantly, if the wrapper captures endpoint_ctx from app() scope, then the module-level function receives the wrong endpoint_ctx (or a stale one)
Actually, let me reconsider the failure scenario. The test expects ResponseValidationError but gets no error. This means validation is not happening.
The most likely cause: the stream_item_field parameter is not being passed correctly to serialize_stream_data.
If stream_item_field is None, the function skips validation:
def serialize_stream_data(..., stream_item_field, ...):
    if stream_item_field:
        validated = stream_item_field.validate(...)
        if validated.errors:
            raise ResponseValidationError(...)
    return json.dumps(jsonable_encoder(raw_item))
When stream_item_field is None, it falls through to json.dumps(jsonable_encoder(raw_item)), which serializes the invalid dict without validation. The test would see a 200 OK response instead of a ResponseValidationError.
Why would stream_item_field be None?
The stream_item_field is set during route setup in get_request_handler() based on the return annotation. For AsyncIterable[Item], it should be the Item TypeAdapter/field.
If the module-level function is called without stream_item_field (i.e., with the default None), validation is skipped. This could happen if:
1. The wrapper in get_request_handler() doesn't pass stream_item_field
2. The wrapper uses a positional argument incorrectly
3. The module-level function has a default of None for stream_item_field
Step 5: Trace the Execution Path
Let me trace the full execution path for the failing test:
Route setup (app startup):
1. app.get("/items/stream-invalid") registers the route
2. get_request_handler() is called for this route
3. route.dependant.is_async_gen_callable is True
4. stream_item_type is extracted from return annotation AsyncIterable[Item]
5. stream_item_field is created from stream_item_type
6. route.is_json_stream = True
Request time (test execution):
 1. client.get("/items/stream-invalid") sends request
 2. app() closure is called with Request
 3. solve_dependencies() resolves dependencies (none in this endpoint)
 4. raw_response = run_endpoint_function(...) returns the async generator
 5. route.is_json_stream is True, so streaming path is taken
 6. async_stream_jsonl is called with the generator and serialize_item callback
 7. async_stream_jsonl iterates the generator and calls serialize_item(raw_item) for each item
 8. serialize_item (the wrapper) calls serialize_stream_item(raw_item, stream_item_field=stream_item_field, ..., endpoint_ctx=endpoint_ctx)
 9. serialize_stream_item calls serialize_stream_data(raw_item, stream_item_field=stream_item_field, ..., endpoint_ctx=endpoint_ctx)
10. serialize_stream_data checks if stream_item_field: — if stream_item_field is truthy, it validates
11. If stream_item_field is None or falsy, validation is skipped
12. The invalid dict {"name": "invalid", "price": "not-a-number"} is serialized as JSON
13. The response is returned as 200 OK with JSONL body
14. Test fails: expected ResponseValidationError, got 200 OK
Step 6: Identify the Root Cause
Root cause: The module-level serialize_stream_data function receives stream_item_field=None (or the parameter is not passed at all, defaulting to None), causing the Pydantic validation path to be skipped. The invalid data flows through the jsonable_encoder + json.dumps fallback and is returned without validation.
Why this occurred after the module-level hoisting:
In the original closure-based code, _serialize_data was defined inside app() and directly captured stream_item_field from the enclosing get_request_handler() scope. The closure guaranteed that stream_item_field was always the correct value for that route.
When hoisted to module level, serialize_stream_data receives stream_item_field as an explicit parameter. The caller must correctly pass this parameter. If the caller's wrapper function (in get_request_handler() scope) doesn't pass stream_item_field, or passes None, the validation path is silently skipped.
This is a signature mismatch between the module-level function and its caller. The module-level function expects stream_item_field to be passed explicitly, but the caller (the wrapper in get_request_handler()) may have been written to only pass endpoint_ctx (the per-request parameter) and forgot to include the route-static parameters.
Step 7: Propose the Smallest Fix
The smallest engineering change that restores the original behavior is to ensure the wrapper function in get_request_handler() scope passes all required route-static parameters to the module-level serialize_stream_item function.
Specifically, the wrapper should look like:
# Inside get_request_handler() scope (executed once per route)
def _serialize_item(raw_item, endpoint_ctx):
    return serialize_stream_item(
        raw_item,
        stream_item_field=stream_item_field,
        response_model_include=response_model_include,
        response_model_exclude=response_model_exclude,
        by_alias=response_model_by_alias,
        exclude_unset=response_model_exclude_unset,
        exclude_defaults=response_model_exclude_defaults,
        exclude_none=response_model_exclude_none,
        endpoint_ctx=endpoint_ctx,
    )
And the call site inside app() should pass endpoint_ctx:
# Inside app() closure (executed per request)
content = async_stream_jsonl(
    aiter=raw_response,
    serialize_item=lambda item: _serialize_item(item, endpoint_ctx),
    ...
)
Why this is the smallest fix:
- It doesn't change the module-level functions (they are correct)
- It only fixes the caller (wrapper function) to pass the correct parameters
- It preserves the optimization goal (wrapper is created once per route, not per request)
- It restores the exact original behavior (validation happens when stream_item_field is present)
Alternative fixes considered and rejected:
1. Move functions back to closure scope: This would fix the bug but undo the entire optimization.
2. Change serialize_stream_data to have stream_item_field=None default and log a warning: This would be a behavior change and wouldn't restore the original validation behavior.
3. Use functools.partial in app() scope: This would create per-request overhead, defeating the optimization goal.
The wrapper fix is the minimal change that both restores behavior and preserves the optimization.
Step 8: Verify the Fix
After applying the fix, I would run:
pytest tests/test_stream_json_validation_error.py -v
Expected result:
tests/test_stream_json_validation_error.py::test_stream_json_validation_error_async PASSED
tests/test_stream_json_validation_error.py::test_stream_json_validation_error_sync PASSED
Both tests should pass, confirming that:
1. ResponseValidationError is correctly raised for invalid stream items
2. The module-level optimization is preserved (no per-request function recreation)
3. The streaming path correctly validates items when stream_item_field is present
Summary
Investigated failure: test_stream_json_validation_error_async fails because ResponseValidationError is not raised.
Root cause: The module-level serialize_stream_data function receives stream_item_field=None (or the parameter is not passed), causing the Pydantic validation path to be silently skipped. Invalid data flows through the jsonable_encoder fallback and is returned as valid JSONL.
Why it occurred: When hoisting _serialize_data from closure scope to module scope, the caller's wrapper function in get_request_handler() scope must explicitly pass the route-static parameter stream_item_field. If the wrapper only passes per-request parameters (endpoint_ctx) and omits route-static parameters, the module-level function defaults to the jsonable_encoder fallback path.
Execution path: Route setup → get_request_handler() creates wrapper → Request time → app() calls wrapper → wrapper calls module-level function → function skips validation (stream_item_field=None) → invalid data serialized → 200 OK returned → test fails.
Smallest fix: Update the wrapper function in get_request_handler() scope to pass stream_item_field (and other route-static parameters) to the module-level serialize_stream_item function. This restores validation behavior while preserving the once-per-route optimization.