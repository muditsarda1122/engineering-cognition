Phase: Debugging  

Objective: Final integration test run.  

Prompt:  
Run pytest tests/ -k "stream or sse or jsonl or cancellation" -x to execute every streaming-related test with fail-fast. Confirm all pass. Then run pytest tests/benchmarks/test_general_performance.py locally (it should skip because --codspeed is missing, but verify it does not crash). Report the final green status.

Expected engineering knowledge accumulated after completing the prompt:  
Absolute confidence that the feature is production-ready.

Why this prompt naturally follows from previous work:  
Prompts 23–26 fixed individual issues; Prompt 27 validates the whole package.