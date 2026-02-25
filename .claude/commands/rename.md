Rename the product throughout the entire codebase.

## Input
$ARGUMENTS = the new product name

## Process
1. Determine display name and module name (lowercase) forms
2. Rename directories, update all imports, docstrings, comments, config.py, CLAUDE.md
3. Update database default path in config.py
4. Run full test suite to verify
5. Commit the rename
