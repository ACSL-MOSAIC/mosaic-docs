---
title: Config on Server
nav_order: 5
---

# Config on Server

MOSAIC-Server is configured through two mechanisms:

1. **`.env` file** — secrets and database credentials, read by Docker Compose at startup
2. **Dashboard UI** — ICE/TURN server entries and robot registrations, stored in the database at runtime

---

## `.env` File

Copy the template before first launch:

```bash
cp deploy/prod/.env-template deploy/prod/.env
```

Then edit `deploy/prod/.env`:

```bash
POSTGRES_USER=mosaic
POSTGRES_PASSWORD=your-strong-password
POSTGRES_DB=mosaic
MOSAIC_SECURITY_ENCRYPTION_KEY=your-base64-aes256-key
```

| Variable | Required | Description |
|:---|:---|:---|
| `POSTGRES_USER` | Yes | PostgreSQL username |
| `POSTGRES_PASSWORD` | Yes | PostgreSQL password |
| `POSTGRES_DB` | Yes | Database name (default: `mosaic`) |
| `MOSAIC_SECURITY_ENCRYPTION_KEY` | Yes | AES-256 key used to encrypt robot auth tokens before storing them in the database (Base64-encoded, 32 bytes) |

### Generating the Encryption Key

The encryption key must be a Base64-encoded 256-bit (32-byte) AES key. Generate one with OpenSSL:

```bash
openssl rand -base64 32
```

Paste the output as the value of `MOSAIC_SECURITY_ENCRYPTION_KEY`.

{: .note }
> Never reuse the local development key (`1tcmum10T0aSvjd01iZ/9Vr04nB2GoJIo9gXCLHJKaE=`) in production. Always generate a fresh key.

---

## ICE / TURN Server Configuration

TURN and STUN server entries are stored in the database, not in the `.env` file. They are managed through the MOSAIC-Server dashboard after deployment:

1. Log in to the dashboard as an organization admin.
2. Navigate to **Settings → ICE Servers**.
3. Add one or more entries:

| Field | Example | Description |
|:---|:---|:---|
| URL | `turn:turn.your-domain.com:3478` | TURN or STUN server URI |
| Username | `your-username` | TURN credential username (leave blank for STUN-only) |
| Credential | `your-password` | TURN credential password (leave blank for STUN-only) |

The backend exposes these entries via `GET /api/v1/webrtc/ice-servers`. The dashboard fetches them automatically when initiating a WebRTC session.

{: .note }
> STUN-only works on open networks. For robots behind NAT (e.g. a mobile robot on a cellular network), a TURN server is required for reliable connections.

---

## Robot Registration

Robots are registered through the dashboard:

1. Log in as an organization admin.
2. Navigate to **Robots → Add Robot**.
3. Give the robot a name. The server generates:
   - A **Robot UUID** — used as `robot_id` in the robot's `mosaic_config.yaml`
   - An **Auth Token** — used as `params.token` in the robot's `mosaic_config.yaml`

Copy both values into the robot's configuration file. See [Config on Robot](./config-on-robot) for the full YAML format.

---

## Advanced: Application Settings

These settings live in `backend/src/main/resources/application.yml` and can be overridden via environment variables. Most deployments do not need to change them.

| Setting | Default | Environment variable | Description |
|:---|:---|:---|:---|
| `server.port` | `9001` | `SERVER_PORT` | Backend HTTP/WebSocket port |
| `jwt.access-token.expiration-time` | `15552000000` | `JWT_ACCESS_TOKEN_EXPIRATION_TIME` | Access token TTL in milliseconds |
| `jwt.refresh-token.expiration-time` | `15552000000` | `JWT_REFRESH_TOKEN_EXPIRATION_TIME` | Refresh token TTL in milliseconds |
| `file.storage.base-path` | `/mosaic-server/storage` | `FILE_STORAGE_BASE_PATH` | Base directory for uploaded files |
| `file.storage.occupancy-map-path` | `/mosaic-server/storage/occupancy-maps` | `FILE_STORAGE_OCCUPANCY_MAP_PATH` | Directory for occupancy map files |

To override a setting without modifying the source, add the environment variable to the `environment` section of `deploy/prod/docker-compose.yaml`:

```yaml
services:
  mosaic-be:
    environment:
      SERVER_PORT: "9002"
      JWT_ACCESS_TOKEN_EXPIRATION_TIME: "3600000"
```

---

## Services Overview

| Service | Image | Port | Role |
|:---|:---|:---|:---|
| `mosaic-be` | `mosaic-be:latest` | `9001` | Spring WebFlux backend (API + WebSocket signaling) |
| `mosaic-fe` | `mosaic-fe:latest` | `8080` | React frontend served by Nginx |
| `mosaic-db` | `postgres:17` | internal | PostgreSQL database |

Database schema migrations (Flyway) run automatically when the backend starts. No manual migration step is required.

See [Custom Deployment](./custom-deployment) for the full Docker Compose setup and Nginx reverse proxy instructions.