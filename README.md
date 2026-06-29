# Logto Docker Compose

Production-grade single-instance [Logto](https://logto.io/) deployment — an open-source identity platform (Auth0 alternative). Also supports multi-instance setups with a shared remote database.

## Architecture

```
                   ┌──────────────────────┐
                   │   Caddy (reverse)     │
                   │   :443 → :3001, :3002 │
                   └──────┬───────────────┘
                          │
              ┌───────────┴───────────┐
              ▼                       ▼
     auth.example.com        console.example.com
         :3001                      :3002
              │                       │
              └───────────┬───────────┘
                          ▼
              ┌──────────────────────┐
              │     logto-app        │
              │   (svhd/logto)       │
              └──────────┬───────────┘
                         │
                         ▼
              ┌──────────────────────┐
              │  logto-postgres      │
              │  (postgres:17-alpine) │
              └──────────────────────┘
```

## Services

| Service              | Image                  | Purpose                                    |
| -------------------- | ---------------------- | ------------------------------------------ |
| `logto-migration`    | `svhd/logto:${TAG}`    | One-shot DB seed — runs once, then exits   |
| `app`                | `svhd/logto:${TAG}`    | Logto core (auth API + admin console)      |
| `postgres`           | `postgres:17-alpine`   | PostgreSQL database with healthcheck       |

### Design notes

- **`logto-migration`** is a one-time container that seeds the database schema. It depends on `postgres` being healthy and the `app` service waits for it to complete before starting. On subsequent `docker compose up` runs it exits immediately if the DB is already seeded.
- Ports `3001` (auth API) and `3002` (admin console) are exposed — put a reverse proxy (Caddy, Nginx) in front with TLS.

## Quick Start (single instance)

```bash
# 1. Copy and fill in environment
cp .env.example .env
# Generate a DB password: openssl rand -hex 32

# 2. Start
docker compose up -d

# 3. Visit https://console.<YOUR_DOMAIN> to create your first admin account
```

## Multi-instance deployment

When running multiple Logto instances that share one PostgreSQL server:

1. **Remove the `postgres` service** from `docker-compose.yml` — keep only `logto-migration` and `app`.
2. **Remove `container_name`** from the `app` service so Docker Compose generates unique names per project.
3. **Point `DB_URL`** to your shared remote database (each instance should use its own database name).
4. **Remove the `volumes:` section** (the `logto-pg` volume is no longer needed).

Example diff for multi-instance:

```diff
  app:
    image: svhd/logto:${TAG}
-   container_name: logto-app
    command: ["start"]
    ...

- postgres:
-   image: postgres:17-alpine
-   ...

- volumes:
-   logto-pg:
```

Start each instance from its own directory with a dedicated `.env`:

```bash
# Instance A
cd /srv/www/logto-a && docker compose -p logto-a up -d

# Instance B
cd /srv/www/logto-b && docker compose -p logto-b up -d
```

Use `-p` (project name) to avoid container name collisions.

## Reverse Proxy (Caddy)

```caddy
auth.<YOUR_DOMAIN> {
    reverse_proxy 127.0.0.1:3001
    tls /etc/caddy/ssl/<YOUR_CERT>.pem /etc/caddy/ssl/<YOUR_CERT>.key
}

console.<YOUR_DOMAIN> {
    reverse_proxy 127.0.0.1:3002
    tls /etc/caddy/ssl/<YOUR_CERT>.pem /etc/caddy/ssl/<YOUR_CERT>.key
}
```

## Backup

```bash
# Single-instance (local postgres container)
docker exec logto-postgres pg_dump -U logto logto_prod > backup.sql

# Multi-instance (remote DB — adjust connection string)
pg_dump -h <DB_HOST> -U logto logto_prod > backup.sql
```
