Run the complete test suite and report results clearly.

## Steps
1. Run `python -m totalreclaw.tests.test_retrieval`
2. If pytest is available, also run `python -m pytest totalreclaw/tests/ -v --tb=short`
3. Run `python -m totalreclaw.examples.basic_agent` to verify the main demo still works
4. Report: total tests, passed, failed, any errors with full tracebacks
5. If any test fails, diagnose the root cause and suggest a fix (but don't apply it without confirmation)
