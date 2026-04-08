---
description: Review the current git diff for bugs, security issues, and convention violations
allowed-tools: Bash(git diff:*), Bash(git status:*), Read, Glob, Grep
---

Review the current git diff (staged and unstaged) for:
- Bugs or logic errors
- Security issues (injection, XSS, etc.)
- Consistency with project conventions (see CLAUDE.md)
- Missing test coverage for new functionality

Be concise. Only flag real issues, not style preferences.
