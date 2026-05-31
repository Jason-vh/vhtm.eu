# Caddy edge router

Caddy on `vhtm-eu` runs as a systemd service installed by the system Caddy
package. It listens on plain HTTP `:8080` and host-routes by `Host:` header
to per-app containers on `127.0.0.1:30NN`. exe.dev's edge terminates TLS
upstream and forwards HTTP to `:8080`; that's why `auto_https off` is on.

## Where the config lives

```text
/etc/caddy/Caddyfile  -> /home/exedev/vhtm.eu/infra/caddy/Caddyfile  (symlink)
```

So editing the file in this repo *is* editing production. Push, `git pull`
on the VM, validate, reload.

## How an app gets a route

Each app ships its routing in its own repo, deployed by its CI to
`~/apps/<app>/deploy/caddy.snippet`. The shared Caddyfile imports them all:

```caddyfile
import /home/exedev/apps/*/deploy/caddy.snippet
```

A typical snippet looks like:

```caddyfile
@recipes host recipes.vhtm.eu
handle @recipes {
	reverse_proxy 127.0.0.1:3002
}
```

The deploy workflow for the app validates and reloads Caddy after `docker
compose up`:

```bash
sudo caddy validate --config /etc/caddy/Caddyfile
sudo systemctl reload caddy
```

Both commands are allowed to the `exedev` user without a password via
`/etc/sudoers.d/caddy-reload`:

```sudoers
exedev ALL=(root) NOPASSWD: /usr/sbin/systemctl reload caddy, /usr/bin/caddy validate --config /etc/caddy/Caddyfile
```

## See every route at a glance

```bash
grep -H "host " ~/apps/*/deploy/caddy.snippet /etc/caddy/Caddyfile
```

## After editing the shared Caddyfile

```bash
cd ~/vhtm.eu && git pull
sudo caddy fmt --overwrite /etc/caddy/Caddyfile
sudo caddy validate --config /etc/caddy/Caddyfile
sudo systemctl reload caddy
```

## Why `auto_https off`

The exe.dev "share" terminates TLS upstream and forwards HTTP to the VM.
If Caddy also tried to obtain certificates it would race with the edge
and fail (the VM has no public 80/443). Leave `auto_https off` in the
global block.
