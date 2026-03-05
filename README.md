# dmgerow/claude-plugins

A personal [Claude Code](https://claude.ai/claude-code) plugin marketplace.

## Install the marketplace

```
/plugin marketplace add dmgerow/claude-plugins
```

## Available plugins

### `homelab`

Docker infrastructure tools for homelab deployments via Portainer.

**Install:**
```
/plugin add dmgerow/claude-plugins/homelab
```

**Commands:**

| Command | Description |
|---------|-------------|
| `/homelab:docker-sidecar [sidecar-type]` | Set up a Docker sidecar service (e.g. Tailscale) in a repo deployed via Portainer Git stacks |

## Adding a new plugin

1. Create `plugins/<name>/`
2. Add `plugins/<name>/.claude-plugin/plugin.json`
3. Add commands to `plugins/<name>/commands/`
4. Register the plugin in `.claude-plugin/marketplace.json`
