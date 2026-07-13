model: kimi k2.5

Test Execution Results
All three streaming test suites pass with 100% success rate.

1. tests/test_stream_cancellation.py — PASS (4/4)
   - test_raw_stream_cancellation[asyncio] — PASSED
   - test_jsonl_stream_cancellation[asyncio] — PASSED
   - test_raw_stream_cancellation[trio] — PASSED
   - test_jsonl_stream_cancellation[trio] — PASSED
   Runtime: 12.74s

2. tests/test_stream_json_validation_error.py — PASS (2/2)
   - test_stream_json_validation_error_async — PASSED
   - test_stream_json_validation_error_sync — PASSED
   Runtime: 1.05s

3. tests/test_stream_bare_type.py — PASS (2/2)
   - test_stream_bare_async_iterable — PASSED
   - test_stream_bare_sync_iterable — PASSED
   Runtime: 0.52s

What Each Test Guards
1. test_stream_cancellation.py — Cancellation Safety
   Guards against the issue fixed in https://github.com/fastapi/fastapi/issues/14680: async streaming endpoints that yield items without any internal await points (e.g., tight while-True loops) could hang indefinitely and never respond to cancellation signals.
   - The test creates endpoints that yield from infinite loops with no internal await
   - Uses anyio.move_on_after(timeout) to cancel the ASGI call after 3 seconds
   - Verifies that the cancellation is actually delivered (cancel_scope.cancelled_caught is True)
   - Tests both raw StreamingResponse and JSONL streaming paths
   - Runs against both asyncio and trio event loop backends (parametrized via pytestmark)

2. test_stream_json_validation_error.py — Validation Error Propagation
   Guards that ResponseValidationError is properly raised (not silently swallowed or incorrectly surfaced as a 500) when a streaming endpoint yields an item that fails Pydantic validation against the declared return type.
   - Async endpoint returning AsyncIterable[Item] yields a dict with an invalid price field ("not-a-number")
   - Sync endpoint returning Iterable[Item] yields the same invalid dict
   - Both tests assert that ResponseValidationError is raised (via pytest.raises)
   - This guards the _serialize_data path where stream_item_field.validate() is called (routing.py:488-498)

3. test_stream_bare_type.py — Bare Type Streaming
   Guards that streaming endpoints can declare a bare AsyncIterable or Iterable return type (without a generic parameter like AsyncIterable[Item]) and still correctly serialize each yielded dict to JSONL format with application/jsonl content-type.
   - Endpoints yield plain dicts with no declared item type
   - Tests verify the response has status 200, content-type application/jsonl, and each line parses as valid JSON
   - This guards the path where stream_item_field is None and _serialize_data falls back to jsonable_encoder + json.dumps (routing.py:508-510)
   - Tests both async and sync generator paths

All three test suites ran against the actual FastAPI codebase and completed without failures.