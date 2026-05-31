# vhtm.eu

Infrastructure and documentation for the `vhtm-eu` VM on exe.dev.

This repo is the source of truth for everything *shared* on the box: the docs,
the conventions, the Caddy edge router, the shared Postgres compose file.
Per-app source code lives in each app's own repo.

If you (or an agent) just SSHed into the VM and have no context, read this
file end-to-end first. It is short on purpose.

## Architecture

```text
client
  -> https://<app>.vhtm.eu
  -> exe.dev edge (TLS termination, "share")
  -> vhtm-eu VM :8080
  -> Caddy (host-routes by Host header)
  -> 127.0.0.1:30NN
  -> Docker container for the app
  -> shared Postgres on the apps-net Docker network
```

TLS is terminated by exe.dev itself, not on the VM. Caddy on the VM runs with
`auto_https off` and listens on plain HTTP `:8080`. The exe.dev "share" maps
that port to `<app>.vhtm.eu` for every hostname registered via
`ssh exe.dev domain add vhtm-eu <host>`.

## Inventory

See [`apps/README.md`](apps/README.md) for the canonical per-app table
(domain, port, repo, runner service).

## Layout on the VM

```text
/home/exedev/
├── README.md              -> vhtm.eu/README.md          (symlink, this file)
├── CLAUDE.md              -> vhtm.eu/CLAUDE.md          (symlink, agent brief)
├── vhtm.eu/               (git clone of Jason-vh/vhtm.eu)
├── infra/                 -> vhtm.eu/infra/             (symlink)
├── infra-secrets/
│   └── postgres.env       (mode 600, NOT in any repo — superuser password)
├── apps/
│   ├── README.md          (symlink into vhtm.eu/apps/README.md)
│   ├── recipes/           (app checkout, populated by the recipes runner)
│   └── push/              (app checkout, populated by the push runner)
└── actions-runner-<repo>/ (self-hosted runner working dirs)

/etc/caddy/Caddyfile       -> /home/exedev/vhtm.eu/infra/caddy/Caddyfile  (symlink)
```

Editing infra = editing this repo. Push to `main`, `git pull` on the VM,
reload the affected service. There is no CI for the infra repo itself
(single editor, low change rate — automation is not worth its cost yet).

## Conventions every app must follow

1. **One app = one directory under `~/apps/<repo-name>/`**, deployed by a
   self-hosted GitHub Actions runner registered to that repo, labeled
   `<repo>-prod`, installed as `gh-actions-runner-<repo>.service`.
2. **Each app binds to `127.0.0.1:30NN`.** Ports are allocated sequentially,
   recorded in [`apps/README.md`](apps/README.md). Next free port =
   highest in the table + 1.
3. **Each app ships its own Caddy routing** at `deploy/caddy.snippet` in its
   own repo. The shared Caddyfile imports `apps/*/deploy/caddy.snippet`.
   When an app is removed, its routing disappears with it.
4. **Each app owns one Postgres database** in the shared instance, owned by
   a per-app role. The role's password lives in that repo's GitHub Actions
   secrets — never on the VM at rest. The shared Postgres superuser password
   is the only DB secret stored on the box, at `~/infra-secrets/postgres.env`,
   mode 600.
5. **DNS pattern**: `<app>.vhtm.eu CNAME vhtm-eu.exe.xyz` at Porkbun,
   plus `ssh exe.dev domain add vhtm-eu <app>.vhtm.eu` to register the host
   with the exe.dev edge. Both steps documented in every app's
   `deploy/README.md`.

## Adding a new app

A short, mechanical checklist. Each step links to where it is documented.

1. Pick the next port from [`apps/README.md`](apps/README.md) and add the row.
2. Create the app's DB and role on the shared Postgres
   (see [`infra/postgres/README.md`](infra/postgres/README.md)).
3. In the app's repo, add `Dockerfile`, `docker-compose.yml` (app service only,
   joins external `apps-net`), `deploy/caddy.snippet`, `deploy/README.md`,
   `.github/workflows/deploy.yml`. Use `apps/recipes` as a template.
4. Add GitHub Actions secrets: `APP_SECRET`, `<APP>_DB_PASSWORD`, plus
   any app-specific secrets.
5. Register a self-hosted runner on the VM
   (see [`infra/README.md`](infra/README.md#self-hosted-runners)).
6. DNS at Porkbun + `ssh exe.dev domain add vhtm-eu <host>`.
7. Push to `main` to deploy.
8. Smoke test, then commit the inventory update to this repo.

## Where to look next

| If you are here to… | Read this |
|---|---|
| Understand the architecture | this file (above) |
| Find an app's port / domain / repo | [`apps/README.md`](apps/README.md) |
| Add a new database for an app | [`infra/postgres/README.md`](infra/postgres/README.md) |
| Add or change a route | [`infra/caddy/README.md`](infra/caddy/README.md) |
| Add a self-hosted runner | [`infra/README.md`](infra/README.md) |
| Debug a running app | `~/apps/<app>/deploy/README.md` (per-app runbook) |

## What this repo does *not* contain

- App source code (each app has its own repo).
- Runtime secrets (those live in GitHub Actions secrets per app, plus
  `~/infra-secrets/postgres.env` on the VM).
- The runner working directories (`actions-runner-<repo>/`) — these are
  installed in place and not under version control.
