model: kimi k2.5

Both checks pass. Here's the summary:
ruff check: Clean — no lint errors (E501 line-too-long, unused variables, import order, etc.)
ruff format --check: One reformat needed — the stream_item_field.validate() call was split across three lines unnecessarily.
Fix applied: ruff format collapsed the call to a single line since it fits within the 88-character limit.
# Before
value, errors_ = stream_item_field.validate(
    data, {}, loc=("response",)
)

# After
value, errors_ = stream_item_field.validate(data, {}, loc=("response",))
Why this is correct: The single-line version is 79 characters, well under the 88-column project limit. The multi-line split was carried over from the per-request closure but is unnecessary at factory scope where the line is shorter (no closure-cell overhead in the expression). No runtime behavior change.
Verification:
- ruff check fastapi/routing.py — clean
- ruff format --check fastapi/routing.py — already formatted
- mypy fastapi/routing.py --strict — clean
- 39 streaming tests pass in 13.67s