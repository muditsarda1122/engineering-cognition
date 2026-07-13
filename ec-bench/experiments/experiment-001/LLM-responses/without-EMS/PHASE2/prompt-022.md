model: kimi k2.5

All 24 test files errored during collection, not during test execution. None of the tests actually ran.
Root cause: StarletteDeprecationWarning from starlette.testclient is being raised as a hard error by pytest's warning filter configuration (likely -W error in pyproject.toml). The warning states:
Using `httpx` with `starlette.testclient` is deprecated; install `httpx2` instead.
First few lines of the traceback pattern (repeated for all 24 files):
miniconda3/lib/python3.12/site-packages/starlette/testclient.py:36: in <module>
    import httpx2 as httpx
E   ModuleNotFoundError: No module named 'httpx2'

During handling of the above exception, another exception occurred:
tests/test_*.py:2: in <module>
    from fastapi.testclient import TestClient
fastapi/testclient.py:1: in <module>
    from starlette.testclient import TestClient as TestClient  # noqa
E   starlette.exceptions.StarletteDeprecationWarning: Using `httpx` with `starlette.testclient` is deprecated; install `httpx2` instead.
Files affected (24 collection errors):
Category	Files
Application	test_application.py
Dependencies (17 files)	test_dependency_after_yield_raise.py, test_dependency_after_yield_streaming.py, test_dependency_after_yield_websockets.py, test_dependency_cache.py, test_dependency_class.py, test_dependency_contextmanager.py, test_dependency_contextvars.py, test_dependency_duplicates.py, test_dependency_overrides.py, test_dependency_paramless.py, test_dependency_partial.py, test_dependency_pep695.py, test_dependency_security_overrides.py, test_dependency_wrapped.py, test_dependency_yield_except_httpexception.py, test_dependency_yield_scope.py, test_dependency_yield_scope_websockets.py
Response model (6 files)	test_response_model_as_return_annotation.py, test_response_model_data_filter.py, test_response_model_data_filter_no_inheritance.py, test_response_model_default_factory.py, test_response_model_include_exclude.py, test_response_model_sub_types.py
Assessment: This is an environment issue, not related to our changes. The test environment lacks httpx2, and the pytest configuration treats StarletteDeprecationWarning as a fatal error. Our earlier test runs (streaming tests, benchmark smoke-tests) succeeded because they were run with a different pytest version/configuration that didn't enforce this warning-as-error behavior. None of the non-streaming paths in get_request_handler were actually exercised by these tests.