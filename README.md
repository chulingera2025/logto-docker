# Logto Docker Deployment

Docker Compose setup for [Logto](https://logto.io/) — an open-source Auth0 alternative for identity and access management.

## Services

| Service           | Image              | Port  |
| ----------------- | ------------------ | ----- |
| `logto-app`       | `svhd/logto:latest` | 3001 (auth), 3002 (admin console) |
| `logto-postgres`  | `postgres:17-alpine` | 5432 (internal) |

## URLs

- **Auth endpoint:** `https://auth.<YOUR_DOMAIN>`
- **Admin console:** `https://console.<YOUR_DOMAIN>`

## Quick Start

1. Copy the example env file and fill in your values:

   ```bash
   cp .env.example .env
   ```

2. Generate a secure database password:

   ```bash
   openssl rand -hex 32
   ```

3. Edit `.env` — set `POSTGRES_PASSWORD`, `DB_URL`, `ENDPOINT`, and `ADMIN_ENDPOINT`.

4. Start the stack:

   ```bash
   docker compose up -d
   ```

5. The admin console will be available at your configured `ADMIN_ENDPOINT`. Create your first admin account on first visit.

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

## Data Persistence

PostgreSQL data is stored in a named Docker volume (`logto-pg`). Back it up with:

```bash
docker exec logto-postgres pg_dump -U logto logto_prod > backup.sql
```
