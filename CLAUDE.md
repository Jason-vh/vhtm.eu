# CLAUDE brief — vhtm-eu VM

You are on the `vhtm-eu` VM (or a workstation that manages it).

Read [`README.md`](README.md) in this directory for the full architecture,
conventions, and runbooks. That is the source of truth.

## Quick orientation

- This is a personal multi-app host. TLS is terminated by exe.dev's edge;
  the VM runs plain HTTP Caddy on `:8080` and host-routes to per-app
  containers on `127.0.0.1:30NN`.
- Apps live under `~/apps/<repo-name>/`, deployed by per-repo self-hosted
  GitHub Actions runners.
- Shared Postgres runs as a Docker Compose project at `~/infra/postgres/`,
  exposed on the `apps-net` Docker network and `127.0.0.1:5432` for ad-hoc
  access. Per-app role + DB; superuser password lives only at
  `~/infra-secrets/postgres.env` (mode 600).
- Caddy config is a thin shell at `~/infra/caddy/Caddyfile`
  (symlinked from `/etc/caddy/Caddyfile`). Each app ships its own routing
  fragment at `apps/<app>/deploy/caddy.snippet`, imported by the shell.

## Conventions to follow

Listed in [`README.md`](README.md#conventions-every-app-must-follow). The
short version: one dir per app under `~/apps/`, sequential ports allocated
in [`apps/README.md`](apps/README.md), one DB per app owned by a per-app
role, Caddy snippet per app in that app's own repo.

## Where things live

| Concern | Location |
|---|---|
| Architecture + conventions | [`README.md`](README.md) |
| App inventory (domain → port → repo) | [`apps/README.md`](apps/README.md) |
| Shared Postgres runbook | [`infra/postgres/README.md`](infra/postgres/README.md) |
| Caddy routing model | [`infra/caddy/README.md`](infra/caddy/README.md) |
| Runner-per-repo pattern | [`infra/README.md`](infra/README.md) |
| Per-app debugging | `~/apps/<app>/deploy/README.md` |

## Things not to do

- Do not put runtime secrets in this repo. It is public.
- Do not edit `/etc/caddy/Caddyfile` directly — it is a symlink into this
  repo. Edit `~/infra/caddy/Caddyfile`, commit, push, `git pull` on the VM,
  `sudo systemctl reload caddy`.
- Do not allocate ports out of order. Take the next number from
  [`apps/README.md`](apps/README.md).
- Do not use `docker compose down -v` on the shared Postgres unless you
  intend to destroy every app's data.
