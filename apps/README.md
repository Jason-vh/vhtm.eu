# Apps on vhtm-eu

This is the inventory. Update it when you add, remove, or move an app.
The "next free port" is one above the highest number in the Port column.

| App     | Domain                                | VM port  | Repo                              | Runner service                      |
|---------|---------------------------------------|---------:|-----------------------------------|-------------------------------------|
| push    | `push.vhtm.eu`                        | `3001`   | `github.com/Jason-vh/push`        | `gh-actions-runner-push.service`    |
| recipes | `recipes.vhtm.eu`                     | `3002`   | `github.com/Jason-vh/recipes`     | `gh-actions-runner-recipes.service` |
| kyle    | `kyle.vhtm.eu`                        | `3003`   | `github.com/Jason-vh/kyle`        | `gh-actions-runner-kyle.service`    |
| ernest  | `ernest.vhtm.eu`                      | `3004`   | `github.com/Jason-vh/ernest`      | `gh-actions-runner-ernest.service`  |
| travels | `*.travels.vhtm.eu` (per-subdomain)   | `3006`   | `github.com/Jason-vh/travels`     | `gh-actions-runner-travels.service` |
| trevor  | `trevor.vhtm.eu`                      | `3007`   | `github.com/Jason-vh/trevor`      | `gh-actions-runner-trevor.service`  |

## Conventions

- One row per app. Sort ascending by port.
- Ports are `30NN`, allocated sequentially. Next free = highest + 1.
  Port `3005` is intentionally skipped (was reserved for `me`/jason.vhattum.com,
  which stayed on Railway).
- Domain is always `<app>.vhtm.eu CNAME vhtm-eu.exe.xyz` (explicit, **not**
  a wildcard — exe.dev's edge rejects any host whose parent zone has a
  wildcard CNAME), and registered with the edge via
  `ssh exe.dev domain add vhtm-eu <host>`.
- Apps that need multi-subdomain routing (like `travels`) use a Caddy
  wildcard matcher in their snippet but still need one explicit Porkbun
  CNAME + one `domain add` per subdomain.
- Runner service name is always `gh-actions-runner-<repo>.service`.

For the full add/remove workflow see the root [`README.md`](../README.md).
