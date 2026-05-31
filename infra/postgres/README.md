# Shared Postgres

One Postgres 17 instance, many databases. Every app gets its own database
and its own role.

## Layout

```text
~/vhtm.eu/infra/postgres/docker-compose.yml      this compose project
~/infra-secrets/postgres.env                     superuser password, mode 600
shared_pgdata                                    named Docker volume
apps-net                                         shared external Docker network
```

Container name: `postgres`. Compose project: `postgres`.

The container publishes `127.0.0.1:5432:5432` so you can run `psql`,
`pg_dump`, `pg_restore` from the VM shell. App containers connect over the
`apps-net` Docker network using the hostname `postgres`.

## Initial bring-up (one-time)

```bash
# 1. The shared Docker network must exist before any compose project that uses it.
docker network create apps-net   # idempotent: ok if it already exists

# 2. Superuser password lives outside the repo.
mkdir -p ~/infra-secrets
chmod 700 ~/infra-secrets
printf 'POSTGRES_PASSWORD=%s\n' "$(openssl rand -hex 32)" > ~/infra-secrets/postgres.env
chmod 600 ~/infra-secrets/postgres.env

# 3. Bring it up.
cd ~/infra/postgres
docker compose up -d
docker compose ps
docker compose logs postgres | tail
```

## Adding an app's database

Each app needs a role and a database it owns. Password is a one-time
generation; record it in that app's GitHub Actions secret (e.g.
`RECIPES_DB_PASSWORD`), never on the VM at rest beyond `.env.production`.

```bash
# Generate a password (record it somewhere safe immediately):
openssl rand -hex 32

# Create the role and DB. Replace <app> and <password>.
docker compose -f ~/infra/postgres/docker-compose.yml exec postgres \
  psql -U postgres -v ON_ERROR_STOP=1 <<SQL
CREATE ROLE <app> WITH LOGIN PASSWORD '<password>';
CREATE DATABASE <app> OWNER <app>;
SQL
```

The app's `.env.production` (written by its deploy workflow) uses:

```text
DATABASE_URL=postgresql://<app>:<password>@postgres:5432/<app>
```

## Removing an app's database

```bash
docker compose -f ~/infra/postgres/docker-compose.yml exec postgres \
  psql -U postgres -v ON_ERROR_STOP=1 <<SQL
DROP DATABASE IF EXISTS <app>;
DROP ROLE IF EXISTS <app>;
SQL
```

## Connecting from the VM shell

```bash
# As superuser:
docker compose -f ~/infra/postgres/docker-compose.yml exec postgres \
  psql -U postgres

# As an app role (uses the loopback publish):
PGPASSWORD=<password> psql -h 127.0.0.1 -p 5432 -U <app> -d <app>
```

## Dump and restore

```bash
# Dump one database to a custom-format file:
docker compose -f ~/infra/postgres/docker-compose.yml exec -T postgres \
  pg_dump -U postgres -Fc -d <app> > /tmp/<app>.dump

# Restore a custom-format dump into an existing (empty) database:
docker compose -f ~/infra/postgres/docker-compose.yml exec -T postgres \
  pg_restore -U postgres -d <app> --no-owner --role=<app> < /tmp/<app>.dump
```

`--no-owner --role=<app>` rewrites ownership to the per-app role on
restore, which is what you want when the dump came from a different
cluster (e.g., Railway, where the owner was `postgres`).

## Operations

```bash
docker compose -f ~/infra/postgres/docker-compose.yml ps
docker compose -f ~/infra/postgres/docker-compose.yml logs -f postgres
docker compose -f ~/infra/postgres/docker-compose.yml restart postgres
```

## Data destruction

The data volume is named `shared_pgdata`. `docker compose down` keeps it.
`docker compose down -v` deletes it, which **destroys every app's data**.
Do not run it.
