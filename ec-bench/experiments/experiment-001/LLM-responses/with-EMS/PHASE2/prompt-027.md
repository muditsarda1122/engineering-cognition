model: kimi k2.5

All streaming-related tests pass. Final green status:

**Streaming tests (direct file execution):**
- `tests/test_stream_bare_type.py` + `tests/test_stream_json_validation_error.py` + `tests/test_stream_cancellation.py` → **8 passed** in 12.87s
- `tests/test_sse.py` → **24 passed** in 1.41s
- **Total: 32/32 streaming tests passed**

**Benchmark suite:**
- `tests/benchmarks/test_general_performance.py` → **1 skipped** (no `--codspeed`, as expected)
- `tests/benchmarks/test_streaming_performance.py` → **1 skipped** (no `--codspeed`, as expected)
- **No crashes, clean skip behavior**

**Pre-existing issue fixed:** `pyproject.toml` was missing the `xdist_group` marker registration required by `--strict-markers`. Added `markers = ["xdist_group: pytest-xdist group marker for workdir locking"]` under `[tool.pytest]`. Without this, `pytest -k` on the full test suite fails at collection time because pytest-xdist's `xdist_group` marker is used in `tests/utils.py:23` but was not registered. This is a repository configuration bug unrelated to our streaming changes.