---
description: Run lint and type check, auto-fix what's fixable, report remaining issues
allowed-tools: Bash(npm run:*), Bash(npx tsc:*), Bash(npx eslint:*)
---

Run `npm run lint` and `npx tsc --noEmit`.
- If there are auto-fixable lint errors, run `npm run lint -- --fix`
- Report any remaining errors with file paths and descriptions
- If clean, confirm with a one-line summary
