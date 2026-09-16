# Configuration reference

All configuration is environment variables, supplied through the `.env` file next to
`docker-compose.yml`.

## Required

### `APPLIANCE_MASTER_KEY`

Encrypts every credential the appliance stores — Kasm API keys, hypervisor logins, cloud
secrets. The container exits immediately if this is unset.

```bash
openssl rand -base64 32
```

Three rules:

- **Back it up** separately from the database. Either alone is useless.
- **Never change it** after first run. Existing credentials become undecryptable.
- **Keep the `.env` readable only by its owner** (`chmod 600 .env`).

---

## Network

| Variable | Default | Notes |
| --- | --- | --- |
| `ALLOWED_ORIGINS` | `https://localhost:8443` | Comma-separated browser origins allowed to call the API. Must include the URL you actually open. A mismatch loads the UI but fails every request. |
| `APPLIANCE_BIND` | `0.0.0.0` | Host interface to publish on. Set `127.0.0.1` to expose only locally. |
| `APPLIANCE_PORT` | `8443` | Port inside the container. Change the compose port mapping instead unless you know why you need this. |
| `TLS_COMMON_NAME` | `localhost` | CN placed in the generated self-signed certificate. Ignored once you install your own. |

---

## Storage

| Variable | Default | Notes |
| --- | --- | --- |
| `DATABASE_URL` | `sqlite:////data/appliance.db` | SQLite is the only driver in the preview image. |
| `REDIS_URL` | `redis://redis:6379/0` | Must use the compose service name, not `localhost`. Pointed at `localhost` the app silently falls back to an in-memory stub and background tasks never run. |

---

## TLS

| Variable | Default | Notes |
| --- | --- | --- |
| `TLS_CERT_PATH` | `/etc/kasm-studio/ssl/cert.pem` | Path inside the container. |
| `TLS_KEY_PATH` | `/etc/kasm-studio/ssl/key.pem` | Must be readable by uid 1000. |

The appliance serves HTTPS directly — there is no bundled reverse proxy. See
[Installation](installation.md#tls-certificates) for installing your own certificate.

---

## Optional

| Variable | Default | Notes |
| --- | --- | --- |
| `IMAGE_TAG` | `latest` | Preview build to run. Pin it for reproducible evaluations. |
| `LOG_LEVEL` | `INFO` | `DEBUG`, `INFO`, `WARNING` or `ERROR`. |
| `REDIS_MAXMEMORY` | `256mb` | Redis memory ceiling. |
| `CELERY_WORKERS` | `2` | Background task worker processes. Raise only if you routinely run many concurrent deploys. |

---

## Volumes

| Volume | Contents | Back up? |
| --- | --- | --- |
| `kasm-data` | Appliance database — stacks, profiles, audit log, encrypted credentials | **Yes** |
| `ssl-data` | TLS certificate and key | If you installed your own |
| `redis-data` | Transient task queue state | No |

---

## A worked example

```bash
# Required
APPLIANCE_MASTER_KEY=<output of: openssl rand -base64 32>

# Network — administrators reach it at studio.example.com
ALLOWED_ORIGINS=https://studio.example.com:8443
APPLIANCE_BIND=0.0.0.0
TLS_COMMON_NAME=studio.example.com

# Container defaults — leave alone
DATABASE_URL=sqlite:////data/appliance.db
REDIS_URL=redis://redis:6379/0
APPLIANCE_PORT=8443
TLS_CERT_PATH=/etc/kasm-studio/ssl/cert.pem
TLS_KEY_PATH=/etc/kasm-studio/ssl/key.pem

# Optional
IMAGE_TAG=latest
LOG_LEVEL=INFO
```
