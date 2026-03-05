# Claude Code Plugin Marketplace

This is a Claude Code plugin marketplace repo (`dmgerow/claude-plugins`).

## Structure

```
claude-plugins/
├── .claude-plugin/
│   └── marketplace.json        # Marketplace catalog
├── plugins/
│   └── <plugin-name>/
│       ├── .claude-plugin/
│       │   └── plugin.json     # Plugin manifest
│       └── commands/
│           └── <command>.md    # Command files
├── CLAUDE.md
└── README.md
```

## Conventions

- Plugin names and command file names: **kebab-case**
- Command files use YAML frontmatter with `description`, `argument-hint`, `allowed-tools`
- Each plugin has `.claude-plugin/plugin.json` with `name`, `description`, `version`, `author`

## Adding a new plugin

1. Create `plugins/<name>/`
2. Add `plugins/<name>/.claude-plugin/plugin.json`
3. Add command `.md` files to `plugins/<name>/commands/`
4. Register in `.claude-plugin/marketplace.json` under `plugins[]`

## Adding a command to an existing plugin

Add a `.md` file to `plugins/<plugin-name>/commands/` with frontmatter:

```markdown
---
description: What this command does
argument-hint: [optional-arg]
allowed-tools: Read, Write, Edit, Bash, Glob
---

Command body here...
```

## Validation

```bash
claude plugin validate .
```

## References

- Plugin docs: https://code.claude.com/docs/en/plugins
- Marketplace docs: https://code.claude.com/docs/en/plugin-marketplaces
- Official plugins: https://github.com/anthropics/claude-plugins-official
