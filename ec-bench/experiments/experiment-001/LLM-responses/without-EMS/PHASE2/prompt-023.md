model: kimi k2.5

Here is the systematic investigation.
1. Symptom: streaming tests fail during collection
Running pytest tests/test_sse.py -v produces:
ERROR collecting tests/test_sse.py
miniconda3/.../starlette/testclient.py:36: in <module>
    import httpx2 as httpx
E   ModuleNotFoundError: No module named 'httpx2'

During handling of the above exception, another exception occurred:
tests/test_sse.py:10: in <module>
    from fastapi.testclient import TestClient
fastapi/testclient.py:1: in <module>
    from starlette.testclient import TestClient as TestClient
E   starlette.exceptions.StarletteDeprecationWarning:
    Using `httpx` with `starlette.testclient` is deprecated;
    install `httpx2` instead.
The traceback has three frames before the actual error:
1. starlette/testclient.py:36 — tries import httpx2 as httpx, fails
2. starlette/testclient.py:47 — falls back to import httpx, emits a StarletteDeprecationWarning
3. fastapi/testclient.py:1 — imports starlette.testclient, which triggers the warning
4. tests/test_sse.py:10 — imports fastapi.testclient.TestClient
Hypothesis 1: The error is in starlette.testclient, a third-party dependency, not in our modified fastapi/routing.py. This suggests an environment issue rather than a code regression.
2. Why the warning is fatal
The pyproject.toml contains:
[tool.pytest.ini_options]
filterwarnings = ["error"]
This converts every warning into a hard error. The StarletteDeprecationWarning emitted by starlette.testclient is therefore fatal.
Question: Why didn't this fail before?
Earlier in the session, streaming tests passed with pytest 7.4.3. After running pip install inline_snapshot, pytest was upgraded to 9.1.1. pytest 9.x enforces filterwarnings from pyproject.toml more strictly than pytest 7.x. The environment also gained configfile: pyproject.toml recognition.
Hypothesis 2: The regression is caused by the pytest upgrade exposing a pre-existing missing dependency (httpx2), not by our code changes.
3. Tracing the dependency chain
Checking pyproject.toml dependency groups:
[dependency-groups]
tests = [
    { include-group = "docs-tests" },   # ← includes httpx2
    "pytest >=9.0.0",
    ...
]

docs-tests = [
    "httpx >=0.23.0,<1.0.0",
    "httpx2>=2.0.0",                     # ← required for tests
    ...
]
Finding: httpx2>=2.0.0 is explicitly listed in the docs-tests group, which is included by the tests group. The project declares it as a required test dependency, but it was never installed in the current environment.
4. Secondary issue discovered during investigation
After installing httpx2, the collection errors disappear, but a new abort occurs:
ERROR: Unknown config option: timeout
collected 28 items
============================ no tests ran in 0.61s
pyproject.toml also contains:
timeout = "20"
This config option is provided by the pytest-timeout plugin. It was also missing from the environment. pytest 7.4.3 apparently silently ignored unknown options; pytest 9.1.1 treats them as fatal.
5. Why the regression occurred (timeline)
Time	Event	Effect
Start of session	Environment had pytest 7.4.3, missing httpx2 and pytest-timeout	Tests passed because pytest 7.4.3 didn't enforce filterwarnings or unknown config options
Earlier prompt	pip install inline_snapshot	Upgraded pytest to 9.1.1 (matching pytest >=9.0.0 in pyproject.toml); pytest 9.1.1 now enforces pyproject.toml strictly
Current state	pytest 9.1.1 + missing httpx2 + missing pytest-timeout	Two fatal config issues: (1) filterwarnings=error converts StarletteDeprecationWarning to fatal error; (2) timeout option is unknown → pytest aborts before running tests
6. Execution path summary
tests/test_sse.py:10
  → from fastapi.testclient import TestClient
    → fastapi/testclient.py:1
      → from starlette.testclient import TestClient
        → starlette/testclient.py:36
          → import httpx2 as httpx          [ModuleNotFoundError]
        → starlette/testclient.py:47
          → warnings.warn(StarletteDeprecationWarning(...))
            → pytest filterwarnings=["error"] converts this to Error
              → Test collection aborts
The path does not enter fastapi/routing.py (where our changes live) at all. This confirms the regression is purely environmental.
7. Smallest engineering change to restore behaviour
Install the two missing dependencies declared in the project's test dependency group:
pip install httpx2 pytest-timeout
Both are listed in pyproject.toml's tests dependency group. No code changes are required.
Why this is the smallest fix:
- httpx2 eliminates the warning at its source (starlette no longer needs to emit StarletteDeprecationWarning)
- pytest-timeout provides the timeout config option pytest expects
- No modifications to pyproject.toml, no code changes, no test rewrites
- The fix aligns the environment with the project's declared dependencies
8. Verification
After installing both packages:
python -m pytest tests/test_sse.py tests/test_stream_cancellation.py \
  tests/test_stream_json_validation_error.py tests/test_stream_bare_type.py \
  tests/test_dependency_after_yield_streaming.py -v
Result: 39 passed in 13.71s — all streaming tests pass.
python -m pytest tests/test_application.py tests/test_dependency_cache.py \
  tests/test_response_model_data_filter.py -v
Result: 15 passed in 1.20s — non-streaming tests also pass.
The fix restores the original test behavior across all execution paths.