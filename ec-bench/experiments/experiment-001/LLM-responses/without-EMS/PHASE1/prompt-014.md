model: kimi k2.5

Validation Strategy
Phase 1: Execute Existing Tests (Baseline)
Fastest, must pass first. Run all streaming-related test files to establish a green baseline before the change:
1. tests/test_stream_cancellation.py (2 tests)
- test_raw_stream_cancellation, test_jsonl_stream_cancellation
- Why: Validates anyio.sleep(0) checkpoints at lines 621, 650, 681 remain functional. If the refactor accidentally removes or reorders a checkpoint, the infinite generator test will hang.
2. tests/test_stream_json_validation_error.py (2 tests)
- test_stream_json_validation_error_async, test_stream_json_validation_error_sync
- Why: This is the only existing test that exercises ResponseValidationError in the streaming path. It validates that _serialize_data still raises with the correct endpoint_ctx when stream_item_field.validate fails.
3. tests/test_stream_bare_type.py (2 tests)
- test_stream_bare_async_iterable, test_stream_bare_sync_iterable
- Why: Exercises JSONL serialization without a stream_item_field (the else branch of _serialize_data). Ensures jsonable_encoder path still works.
4. tests/test_sse.py (~20 tests)
- Covers async/sync SSE, ServerSentEvent payloads, mixed items, raw data, router prefixes, keepalive.
- Why: The most comprehensive streaming test suite. It exercises _serialize_sse_item and _serialize_data through the SSE path, including ServerSentEvent handling (lines 522–537) and plain object serialization (line 542). The router-prefix test (/api/events) verifies endpoint_ctx path construction from request.scope["root_path"].
5. tests/test_dependency_after_yield_streaming.py (5 tests)
- test_stream_simple, test_stream_session, test_broken_session_stream_raise, test_broken_session_stream_no_raise
- Why: Validates that async_exit_stack lifecycle (lines 606, 611) and dependency cleanup still work with streaming responses. The broken-session tests verify structured teardown survives errors.
6. tests/test_validation_error_context.py (5 tests)
- test_request_validation_error_includes_endpoint_context, test_subapp_request_validation_error_includes_endpoint_context
- Why: Validates endpoint_ctx path construction for regular and subapp routes. The subapp test is critical because endpoint_ctx["path"] includes request.scope["root_path"] (line 414) — if endpoint_ctx were cached at factory scope, subapp paths would be incorrect.
7. Full test suite (pytest tests/)
- Why: The change touches get_request_handler, which is on every route's hot path. Non-streaming routes must be unaffected.
Phase 2: Add Targeted Tests (Gaps)
Three gaps exist in current coverage that the refactor could expose:
A. SSE ResponseValidationError with correct endpoint_ctx
- Gap: test_stream_json_validation_error.py covers JSONL but not SSE.
- Add: A test yielding invalid data through an EventSourceResponse endpoint with a Pydantic model annotation, asserting ResponseValidationError is raised and the error message contains the correct endpoint path.
- Why: Verifies that the explicit endpoint_ctx parameter is correctly forwarded from app through _producer to factory-scope _serialize_sse_item → _serialize_data.
B. endpoint_ctx in streaming ResponseValidationError for mounted sub-apps
- Gap: test_validation_error_context.py tests subapp RequestValidationError but not streaming ResponseValidationError.
- Add: A mounted sub-app with a streaming endpoint that yields invalid data, asserting the ResponseValidationError message includes the full mounted path.
- Why: This is the highest-risk behavioral regression. If endpoint_ctx is cached at factory scope or incorrectly forwarded, the first request's path would appear in all subsequent errors from the same route, and subapp mount prefixes would be missing or stale.
C. Allocation micro-benchmark
- Gap: No benchmark covers streaming endpoint setup.
- Add: A benchmark test (using the existing CodSpeed infrastructure) that creates 10,000 streaming requests and measures per-request object allocations.
- Why: The entire engineering motivation is reducing allocation overhead. The benchmark should show a measurable reduction in function-object and closure-cell creation. If the refactor doesn't move the needle, the change should be reconsidered.
Phase 3: Justify Confidence
This strategy provides three layers of confidence:
1. Behavioral: All 11+ streaming tests (existing + 2 new) cover every combination of:
- SSE / JSONL / raw
- Async / sync generators
- Valid / invalid data (validation errors)
- Plain / annotated / bare return types
- Router prefix / sub-app mount paths
- Cancellation checkpoints
- Dependency lifecycle
2. Structural: The SSE keepalive test and the cancellation tests exercise the task group and async_exit_stack paths that must remain per-request. If async_exit_stack or gen were accidentally hoisted, these tests would hang or leak.
3. Quantitative: The allocation benchmark provides objective evidence that the engineering goal (−3 function objects, −7 closure cells per request) is achieved. Without it, the change is an unmeasured optimization.
Execution Order
1. Run existing streaming tests → must all pass (baseline)
2. Implement the change
3. Run existing streaming tests again → must all pass (regression check)
4. Run full test suite → must all pass
5. Add new tests (A, B) → must all pass
6. Run allocation benchmark → must show measurable reduction
7. Merge
If any existing test fails at step 3, the change violates a behavioral guarantee and must be revised before adding new tests.