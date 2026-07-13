model: kimi k2.5

All four test suites passed with zero failures.
Test file	Tests	Result	Time
tests/test_stream_cancellation.py	4	4 passed	12.75s
tests/test_stream_json_validation_error.py	2	2 passed	1.01s
tests/test_stream_bare_type.py	2	2 passed	1.01s
tests/test_sse.py	24	24 passed	1.81s
Total: 32 passed, 0 failed, 0 errors
The only output outside of passing tests is a repeated StarletteDeprecationWarning about httpx (unrelated to our changes). No tracebacks.