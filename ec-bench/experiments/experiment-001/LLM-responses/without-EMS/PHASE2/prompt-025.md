model: kimi k2.5

The type-checking run found one new type error introduced by the factory-scope hoisting:
fastapi/routing.py:508: error: Unused "type: ignore" comment  [unused-ignore]
What happened: In the original per-request closure, the for item in gen: loop carried # ty: ignore[not-iterable] (with a typo — ty: instead of type:). When I hoisted the function to factory scope and gave gen an explicit Iterator[Any] parameter annotation, mypy now correctly understands that gen is iterable, so the suppression comment is unnecessary.
The fix (already applied):
# Before (line 508)
def _sync_stream_jsonl(
    gen: Iterator[Any],  # type: ignore[not-iterable]
    endpoint_ctx: EndpointContext | None,
) -> Iterator[bytes]:

# After (line 508)
def _sync_stream_jsonl(
    gen: Iterator[Any],
    endpoint_ctx: EndpointContext | None,
) -> Iterator[bytes]:
Why this improves type safety: Removing an unused # type: ignore comment prevents it from masking future legitimate type errors. The explicit Iterator[Any] parameter annotation already provides the type information mypy needs, making the suppression redundant. The change is purely in comments — zero runtime impact.
Verification:
- mypy fastapi/routing.py --strict — clean (no errors in routing.py)
- ruff check fastapi/routing.py — clean
- All 39 streaming tests pass in 13.58s