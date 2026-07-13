model: kimi k2.5

Done.
Audit results — nothing to delete
- Inner functions: All remaining closures inside app() are actively used:
- _sse_producer_cm (entered on async_exit_stack at line 657)
- _producer (started in task group at line 648)
- _keepalive_inserter (started in task group at line 649)
- Imports: Every import at the top of routing.py is referenced elsewhere in the file (e.g., copy at line 1689, errno at line 1944). Our refactor only moved existing logic; it did not remove any usages.
Docstrings added
Concise docstrings added to all seven factory-scope helpers inside get_request_handler, following the one-line style used by _extract_endpoint_context and other internal FastAPI utilities:
Function	Docstring summary
_serialize_data	Serialize a single stream item to JSON bytes, with validation fallback
_serialize_sse_item	Serialize a single SSE item to wire-format bytes, handling ServerSentEvent wrappers
_sse_with_checkpoints	Yield SSE bytes with async checkpoints for cancellation delivery
_serialize_jsonl_item	Serialize a single JSONL item with trailing newline
_async_stream_jsonl	Async generator wrapper for JSONL serialization with checkpoints
_sync_stream_jsonl	Sync generator wrapper for JSONL serialization
_async_stream_raw	Async generator wrapper for raw chunk streaming with checkpoints
Lint fixed
- ruff check fastapi/routing.py reported one pre-existing E501 (line too long) at line 143 in request_response. The error message string was re-wrapped to stay within 88 columns.
- ruff now passes cleanly.
Verification
291 tests pass (72 streaming + 219 non-streaming) in 14.34s.