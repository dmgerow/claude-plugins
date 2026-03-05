---
description: Set up a Docker sidecar service (e.g. Tailscale) in a repo deployed via Portainer Git stacks
argument-hint: [sidecar-type]
allowed-tools: Read, Write, Edit, Bash, Glob
---

## Context

You are helping set up a Docker sidecar for a repo that will be deployed via Portainer using a Git URL stack.

**Critical Portainer Git stack constraint:** Portainer pulls only the `docker-compose.yml` (or stack file) from the Git repo — it does NOT reliably make other files in the repo available as bind-mount sources. If you bind-mount a host file (e.g. `./config.json:/container/config.json`) and that file doesn't exist on the Portainer host at deploy time, Docker creates a **directory** at that path instead. Once created, subsequent git pulls cannot overwrite the directory with a file, so the broken state persists across redeployments.

**The rule:** Never use file bind mounts (`./some-file:/container/path`) in a Portainer Git stack. Use only:
1. **Named volumes** for persistent data
2. **Directory bind mounts** (`./some-dir:/container/dir`) — directories are safe because Docker creates them correctly when missing
3. **Inline config** — write config files inside the container at startup via an entrypoint override

## Your task

The user wants to add a sidecar service to their `docker-compose.yml`. Argument: `$ARGUMENTS`

1. Read the existing `docker-compose.yml` if present.
2. Identify what sidecar is needed (from the argument or conversation). Common sidecars:
   - **Tailscale** — exposes the app on a Tailscale network with HTTPS via `TS_SERVE_CONFIG`
   - Others as described by the user
3. Design the sidecar configuration following these rules:
   - **No file bind mounts** — if the sidecar needs a config file, write it at container startup using an entrypoint override like:
     ```yaml
     entrypoint:
       - /bin/sh
       - -c
       - |
         printf '<json-content>' > /tmp/config.json
         exec /original/entrypoint
     ```
   - Use `$VAR` in docker-compose YAML when you need a literal `$VAR` in the shell (double `$` escapes docker-compose interpolation)
   - Use single quotes in `printf` to prevent shell from expanding variables that the sidecar process itself needs to substitute (e.g. Tailscale's `${TS_CERT_DOMAIN}`)
   - Named volumes only for persistent state
4. Wire the app service to the sidecar using `network_mode: service:<sidecar>` if the sidecar owns the network interface (e.g. Tailscale)
5. Write the updated `docker-compose.yml`
6. Commit and push

## Tailscale best practices and troubleshooting

Refer to the official Tailscale Docker guide for best practices and troubleshooting:
https://tailscale.com/blog/docker-tailscale-guide

## Tailscale sidecar reference

When the sidecar is Tailscale, use this proven pattern:

```yaml
tailscale-<appname>:
  image: tailscale/tailscale:latest
  container_name: <appname>-tailnet
  hostname: <appname>
  environment:
    TS_AUTHKEY: ${TS_OAUTH_SECRET}
    TS_EXTRA_ARGS: --advertise-tags=tag:docker
    TS_STATE_DIR: /var/lib/tailscale
    TS_SERVE_CONFIG: /tmp/ts-serve.json
  entrypoint:
    - /bin/sh
    - -c
    - |
      printf '{"TCP":{"443":{"HTTPS":true}},"Web":{"${TS_CERT_DOMAIN}:443":{"Handlers":{"/":{"Proxy":"http://127.0.0.1:<app-port>"}}}}}' > /tmp/ts-serve.json
      exec /usr/local/bin/containerboot
  volumes:
    - ts-state:/var/lib/tailscale
    - /dev/net/tun:/dev/net/tun
  cap_add:
    - net_admin
    - sys_module
  restart: unless-stopped
```

The app service uses `network_mode: service:tailscale-<appname>` so it shares the Tailscale container's network interface.

The user must set `TS_OAUTH_SECRET` as an environment variable in Portainer's stack config.
