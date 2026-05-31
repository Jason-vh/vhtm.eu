# Shared infra on vhtm-eu

Shared services live here. Each subdirectory is one concern.

| Concern                                  | Where                                |
|------------------------------------------|--------------------------------------|
| Shared Postgres (one instance, many DBs) | [`postgres/`](postgres/README.md)    |
| Caddy edge router (host-routes by name)  | [`caddy/`](caddy/README.md)          |
| Self-hosted GitHub Actions runners       | this file, [below](#self-hosted-runners) |

The runtime state of every shared piece is documented in its own README.
This file only covers things that don't deserve a directory of their own.

## Self-hosted runners

Each app repo has its own runner on the VM. The pattern is:

```text
~/actions-runner-<repo>/             working dir for the runner
gh-actions-runner-<repo>.service     systemd unit, runs the runner as user `exedev`
```

Runners are labeled `<repo>-prod` so workflows can target them with:

```yaml
runs-on: [self-hosted, linux, <repo>-prod]
```

### Adding a new runner

GitHub's "actions-runner" tarball changes URL frequently — check
<https://github.com/actions/runner/releases> for the current `linux-x64`
download URL.

```bash
# On the VM as `exedev`:
cd ~
mkdir actions-runner-<repo> && cd actions-runner-<repo>
curl -O -L <runner-tarball-url>
tar xzf actions-runner-linux-x64-<ver>.tar.gz

# Get a one-shot registration token (run this on your workstation):
gh api -X POST repos/Jason-vh/<repo>/actions/runners/registration-token

# Back on the VM, register:
./config.sh \
  --url https://github.com/Jason-vh/<repo> \
  --token <token-from-above> \
  --name vhtm-eu-<repo> \
  --labels <repo>-prod \
  --unattended \
  --replace

# Install + start as a systemd service:
sudo ./svc.sh install exedev
sudo ./svc.sh start

# Rename the unit to the project convention:
sudo systemctl stop  actions.runner.Jason-vh-<repo>.vhtm-eu-<repo>.service
sudo systemctl disable actions.runner.Jason-vh-<repo>.vhtm-eu-<repo>.service
sudo mv /etc/systemd/system/actions.runner.Jason-vh-<repo>.vhtm-eu-<repo>.service \
        /etc/systemd/system/gh-actions-runner-<repo>.service
sudo systemctl daemon-reload
sudo systemctl enable --now gh-actions-runner-<repo>.service
```

The `svc.sh` rename step is cosmetic but keeps the service-name pattern
consistent and matches the [inventory](../apps/README.md).

### Removing a runner

```bash
sudo systemctl disable --now gh-actions-runner-<repo>.service
sudo rm /etc/systemd/system/gh-actions-runner-<repo>.service
sudo systemctl daemon-reload
cd ~/actions-runner-<repo>
# Remove from GitHub side:
./config.sh remove --token "$(gh api -X POST repos/Jason-vh/<repo>/actions/runners/remove-token --jq .token)"
cd ~ && rm -rf actions-runner-<repo>
```
