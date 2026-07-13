Phase: Investigation

Objective: Enumerate every inner function created inside the per-request app coroutine.

Prompt:
Inside the inner app function returned by `get_request_handler`, identify every nested helper defined using either `def` or `@asynccontextmanager`.

For each helper, report:

- its name
- its approximate location within the function
- its primary responsibility
- whether it is created on every request or only on specific execution paths
- the variables it appears to capture from the enclosing scope (if any)

Your inventory should include helpers such as `_serialize_data`, `_serialize_sse_item`, `_sse_producer_cm`, `_async_stream_jsonl`, `_sync_stream_jsonl`, `_async_stream_raw`, and `_sse_with_checkpoints`, but should not assume this list is exhaustive.

Expected engineering knowledge accumulated after completing the prompt:
A complete inventory of the helper functions created inside the per-request request handler, including an initial understanding of which ones may contribute to repeated per-request closure allocation.

Why this prompt naturally follows from previous work:
Prompt 3 established how `get_request_handler` is structured as a factory. Prompt 4 now decomposes the per-request execution path to identify where repeated helper creation occurs.