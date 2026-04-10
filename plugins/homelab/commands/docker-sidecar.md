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
4. Wire the sidecar to the app's network namespace using `network_mode: service:<app>` on the **sidecar** container (not the other way around) — see "Critical: network_mode direction" below
5. Write the updated `docker-compose.yml`
6. Commit and push

## Tailscale best practices and troubleshooting

Refer to the official Tailscale Docker guide for best practices and troubleshooting:
https://tailscale.com/blog/docker-tailscale-guide

## Tailscale sidecar reference

When the sidecar is Tailscale, use this proven pattern:

```yaml
services:
  <appname>:
    image: <app-image>
    hostname: <appname>          # Tailscale reads this for the node name
    environment:
      HOSTNAME: 0.0.0.0          # Bind app to all interfaces so Tailscale can reach it
    # ... other app config ...
    restart: unless-stopped

  tailscale-<appname>:
    image: tailscale/tailscale:latest
    container_name: <appname>-tailnet
    network_mode: service:<appname>   # Tailscale joins the app's namespace
    depends_on:
      - <appname>
    environment:
      TS_AUTHKEY: ${TS_OAUTH_SECRET}
      TS_EXTRA_ARGS: --advertise-tags=tag:docker
      TS_STATE_DIR: /var/lib/tailscale
      TS_SERVE_CONFIG: /tmp/ts-serve.json
      TS_EPHEMERAL: false
    entrypoint:
      - /bin/sh
      - -c
      - |
        printf '{"TCP":{"443":{"HTTPS":true}},"Web":{"$${TS_CERT_DOMAIN}:443":{"Handlers":{"/":{"Proxy":"http://127.0.0.1:<app-port>"}}}}}' > /tmp/ts-serve.json
        exec /usr/local/bin/containerboot
    volumes:
      - ts-state:/var/lib/tailscale
      - /dev/net/tun:/dev/net/tun
    cap_add:
      - net_admin
      - sys_module
    restart: unless-stopped

volumes:
  ts-state:
```

The **app** service owns the network namespace; Tailscale joins it via `network_mode: service:<appname>`.

The user must set `TS_OAUTH_SECRET` as an environment variable in Portainer's stack config.

## Critical: network_mode direction

**Tailscale must join the app's namespace — not the other way around.**

`containerboot` (the Tailscale Docker entrypoint) changes its own network namespace after startup to attach WireGuard and set up routing. If the app uses `network_mode: service:tailscale-<appname>`, the app lands in Tailscale's *original* namespace — which `containerboot` has already abandoned. The app gets no Tailscale interface and can't be reached.

The correct direction:
- App service: no `network_mode` — owns the namespace
- Tailscale service: `network_mode: service:<appname>` + `depends_on: <appname>`

This way Tailscale joins the stable, app-owned namespace and WireGuard is set up there.

## TLS cert on first deploy

On the **first** deploy, Tailscale has not yet provisioned a TLS cert for the hostname. HTTPS requests will hang indefinitely until the cert exists.

After the first deploy, run once:

```bash
docker exec <appname>-tailnet tailscale cert <hostname>.<tailnet>.ts.net
```

Subsequent container restarts auto-renew the cert from the persistent `ts-state` volume — no manual step needed.

## Debugging checklist

When something isn't working, verify each layer in order:

**1. Confirm namespace sharing**
```bash
# Both containers must show the same inode number
docker exec <appname> cat /proc/self/ns/net
docker exec <appname>-tailnet cat /proc/self/ns/net
```

**2. Test app reachability from Tailscale container**
```bash
# Should return the app's response; if it hangs, the app isn't binding to 0.0.0.0
docker exec <appname>-tailnet wget -qO- http://127.0.0.1:<app-port>
```

**3. Check serve config was applied**
```bash
docker exec <appname>-tailnet tailscale serve status
```

**4. Provision cert if HTTPS hangs**
```bash
docker exec <appname>-tailnet tailscale cert <hostname>.<tailnet>.ts.net
```
