# Apps on vhtm-eu

This is the inventory. Update it when you add, remove, or move an app.
The "next free port" is one above the highest number in the Port column.

| App     | Domain              | VM port  | Repo                              | Runner service                    |
|---------|---------------------|---------:|-----------------------------------|-----------------------------------|
| push    | `push.vhtm.eu`      | `3001`   | `github.com/Jason-vh/push`        | `gh-actions-runner-push.service`    |
| recipes | `recipes.vhtm.eu`   | `3002`   | `github.com/Jason-vh/recipes`     | `gh-actions-runner-recipes.service` |
| kyle    | `kyle.vhtm.eu`      | `3003`   | `github.com/Jason-vh/kyle`        | `gh-actions-runner-kyle.service`    |

## Conventions

- One row per app. Sort ascending by port.
- Ports are `30NN`, allocated sequentially. Next free = highest + 1.
- Domain is always `<app>.vhtm.eu CNAME vhtm-eu.exe.xyz` and registered
  with the exe.dev edge via `ssh exe.dev domain add vhtm-eu <host>`.
- Runner service name is always `gh-actions-runner-<repo>.service`.

For the full add/remove workflow see the root [`README.md`](../README.md).
