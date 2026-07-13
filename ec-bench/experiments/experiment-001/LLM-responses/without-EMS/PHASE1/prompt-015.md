model: kimi k2.5

Done. I've implemented the foundational component for the streaming serialization refactor.
What was added:
A factory-scope _serialize_data helper at fastapi/routing.py:402-428, defined inside get_request_handler but outside the per-request app closure.
Key properties:
- Production-quality: Full type annotations (endpoint_ctx: EndpointContext | None, data: Any) → bytes, plus a docstring explaining its role in eliminating per-request allocation
- Explicit endpoint_ctx parameter: Replaces the implicit closure capture from the old inner helper, preserving request-specific ResponseValidationError paths
- Route-static closures only: Captures stream_item_field and the 6 response-model filter parameters at registration time
- Behavioral parity: Identical logic to the existing inner _serialize_data — same validation, same ctx = endpoint_ctx or EndpointContext() fallback, same serialization branches
What was NOT changed:
- The existing inner _serialize_data at line 519 remains untouched and is still the active code path
- No streaming generators (_producer, _async_stream_jsonl, _sync_stream_jsonl) were modified
- No call sites were updated
- All existing tests pass (46 verified across 5 streaming/validation test files)
This helper is ready for future integration: the next step is moving _serialize_sse_item and _serialize_item to factory scope, then updating the streaming generators to pass endpoint_ctx explicitly and removing the old inner helpers.