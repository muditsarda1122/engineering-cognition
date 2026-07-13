model: kimi k2.5

## Context Assembly

### Repository State
The streaming serialization abstraction is implemented as 5 module-level functions in `fastapi/routing.py` (lines 349-524). The prior integration steps eliminated all per-request serialization closures.

---

## Cleanup Results

### Unused Inner Functions and Imports

**Result: No unused code found.**

- **Inner functions inside `get_request_handler`:** All per-request closures were already removed in the prior integration steps. The remaining closures (`_sse_producer_cm`, `_producer`, `_keepalive_inserter`, `_sse_with_checkpoints`, `_async_stream_raw`) all capture genuinely per-request state and are actively used.
- **Imports:** Verified by running `ruff check fastapi/routing.py` — passes on first run with no unused import warnings.

### `_StreamSerializers` Class

**Result: No such class exists.**

The abstraction was implemented as module-level functions (`serialize_stream_data`, `serialize_sse_item`, `serialize_stream_item`, `async_stream_jsonl`, `sync_stream_jsonl`), not a class. This is consistent with the prior architectural decision that rejected the "Small helper class" approach as "inconsistent with codebase style" and followed the precedent of `serialize_response` and `run_endpoint_function` (both module-level functions extracted from `get_request_handler`).

### Docstring Enhancements

Enhanced the docstrings of all 5 module-level streaming serialization functions:

1. **`serialize_stream_data`** — Expanded to mention the Pydantic v2 ``serialize_json`` fast path and the ``jsonable_encoder`` + ``json.dumps`` fallback.
2. **`serialize_sse_item`** — Expanded to describe both the `ServerSentEvent` serialization path and the plain-object path via `serialize_stream_data`.
3. **`serialize_stream_item`** — Added note about JSON Lines record format.
4. **`async_stream_jsonl`** — Added note about the `anyio.sleep(0)` cancellation checkpoint.
5. **`sync_stream_jsonl`** — Added note identifying it as the synchronous counterpart.

### Lint Check

- **ruff check:** ✅ All checks passed! (no fixes needed)

### Test Verification

- **32 streaming tests:** ✅ All pass (test_stream_cancellation.py, test_stream_json_validation_error.py, test_stream_bare_type.py, test_sse.py)
- **Import check:** ✅ `import fastapi.routing` succeeds

---

## Summary

No cleanup was required — the prior integration was already clean. Docstrings were enhanced for all 5 streaming serialization functions. Ruff passes with no issues.
