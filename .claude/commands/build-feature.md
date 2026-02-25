Plan and build a new feature for the TotalReclaw project.

## Input
$ARGUMENTS

## Process
1. **Scope Check:** Verify this feature is in scope for v1. Check CLAUDE.md "What NOT to Do" section. If it's v2, say so and stop.
2. **Plan:** Describe what you'll build, which files you'll modify/create, and edge cases. Present plan for review before executing.
3. **Build:** Implement with type hints, docstrings, error handling, parameterized SQL.
4. **Test:** Write or update tests. Run full test suite.
5. **Verify:** Run `python -m totalreclaw.examples.basic_agent` to ensure nothing broke.
6. **Report:** Summarize what was built, tested, and any follow-up needed.

## Rules
- Do not modify existing working code unless the feature requires it
- Do not add external dependencies
- If unsure about a design decision, ask rather than guessing
