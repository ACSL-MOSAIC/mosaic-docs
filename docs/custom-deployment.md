---
title: Custom Deployment
nav_order: 8
---

# Custom Deployment

This page describes how to deploy **MOSAIC-Server** using Docker Compose.

MOSAIC-Server consists of three services:

| Service | Image | Port | Role |
|:---|:---|:---|:---|
| `mosaic-be` | `mosaic-be:latest` | `9001` | Spring WebFlux backend |
| `mosaic-fe` | `mosaic-fe:latest` | `8080` | React frontend (Nginx) |
| `mosaic-db` | `postgres:17` | internal | PostgreSQL database |

---

## Prerequisites

- Docker Engine 24+
- Docker Compose v2

---

## 1. Clone the Repository

```bash
git clone https://github.com/ACSL-MOSAIC/MOSAIC-Server.git
cd MOSAIC-Server
```

---

## 2. Build the Images

**Backend** (Spring WebFlux — multi-stage build with Azul Zulu JDK 21):

```bash
docker build -t mosaic-be:latest ./backend
```

**Frontend** (React → Nginx):

```bash
docker build -t mosaic-fe:latest ./frontend
```

---

## 3. Configure Environment Variables

```bash
cp deploy/prod/.env-template deploy/prod/.env
```

Edit `deploy/prod/.env`:

```bash
POSTGRES_USER=mosaic
POSTGRES_PASSWORD=your-strong-password
POSTGRES_DB=mosaic
MOSAIC_SECURITY_ENCRYPTION_KEY=your-base64-encoded-aes256-key
```

| Variable | Description |
|:---|:---|
| `POSTGRES_USER` | PostgreSQL username |
| `POSTGRES_PASSWORD` | PostgreSQL password |
| `POSTGRES_DB` | Database name (default: `mosaic`) |
| `MOSAIC_SECURITY_ENCRYPTION_KEY` | AES-256 key for encrypting robot auth tokens (Base64-encoded) |

---

## 4. Run

```bash
docker compose -f deploy/prod/docker-compose.yaml up -d
```

Database schema migrations (Flyway) run automatically on backend startup. No manual migration step is required.

### Verify

```bash
docker compose -f deploy/prod/docker-compose.yaml ps
```

```
NAME         STATUS    PORTS
mosaic-be    running   0.0.0.0:9001->9001/tcp
mosaic-fe    running   0.0.0.0:8080->80/tcp
mosaic-db    running
```

---

## 5. Nginx Reverse Proxy (Production)

In production, place an Nginx reverse proxy in front to handle TLS and route traffic:

- `api-mosaic.your-domain.com` → backend (`localhost:9001`)
- `mosaic.your-domain.com` → frontend (`localhost:8080`)

WebSocket routes (`/ws/*`) must forward the `Upgrade` header:

```nginx
location /ws {
    proxy_pass http://backend;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
    proxy_set_header Host $host;
    proxy_read_timeout 86400;
    proxy_send_timeout 86400;
}
```

See `deploy/prod/nginx.conf` for the full reference configuration.

---

## Useful Commands

```bash
# Follow logs
docker compose -f deploy/prod/docker-compose.yaml logs -f

# Restart a specific service
docker compose -f deploy/prod/docker-compose.yaml restart mosaic-be

# Stop all services
docker compose -f deploy/prod/docker-compose.yaml down

# Stop and wipe the database volume
docker compose -f deploy/prod/docker-compose.yaml down -v
```