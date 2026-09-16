# Installation

Production-shaped installation of the Kasm Windows Studio Tech Preview. For a fast
evaluation, use the [Quick start](quick-start.md) instead.

## Contents

- [Sizing](#sizing)
- [Network requirements](#network-requirements)
- [Kasm API permissions](#kasm-api-permissions)
- [Install](#install)
- [TLS certificates](#tls-certificates)
- [Windows guest templates](#windows-guest-templates)
- [Backup and restore](#backup-and-restore)
- [Upgrading](#upgrading)
- [Uninstalling](#uninstalling)

---

## Sizing

The control plane is not in the session data path, so it sizes against the number of
concurrent deploy operations, not the number of user sessions.

| | vCPU | RAM | Disk |
| --- | --- | --- | --- |
| Minimum | 2 | 4 GB | 20 GB |
| Recommended | 4 | 8 GB | 50 GB |

Host requirements: Linux with Docker Engine 24+ and the Compose plugin. The image
bundles the API, task worker and web UI; Redis runs as a second container.

---

## Network requirements

| From | To | Port | Why |
| --- | --- | --- | --- |
| Administrators | Studio | 8443/tcp | Web UI and API |
| Studio | Kasm Workspaces API | 443/tcp | All Kasm operations |
| Studio | VM provider endpoints | provider-specific | Creating and querying VMs |
| Windows VMs | Kasm Workspaces | 443/tcp | Desktop Service enrollment and check-in |
| Kasm Workspaces | Windows VMs | 4902/tcp | Required for dynamic user accounts and session scripts |

Two of these are easy to miss:

- **Studio must reach Kasm by a URL that resolves inside the container.** A hostname that
  only resolves on the Docker host will fail. Test with
  `docker compose exec appliance curl -kfsS https://kasm.example.com/api/__healthcheck`.
- **Kasm must reach the Windows VMs on 4902/tcp.** Without it, dynamic user accounts and
  session scripts do not work, even though enrollment appears to succeed.

---

## Kasm API permissions

Create a dedicated API credential in the Kasm admin portal. Grant the minimum set:

| Permission area | Access | Used for |
| --- | --- | --- |
| **Server Pools** | Read, Write | Create and synchronise pools |
| **Servers** | Read, Write | Register and manage server records |
| **Workspaces** | Read, Write | Create pool-backed workspaces and control routing |
| **Autoscale** | Read, Write | Manage autoscale configs and VM providers |
| **Settings** | Read | Read zones and deployment settings |

Use a dedicated credential rather than a personal admin account — the audit trail in
Kasm will then distinguish Studio's changes from a human's.

> [!NOTE]
> Some Kasm endpoints accept an API key while others require an administrator session.
> If you supply only a key, features that depend on session-authenticated endpoints
> (notably storage mapping management) will not be available. Supply the administrator
> username and password as well for full functionality.

---

## Install

```bash
mkdir -p /opt/kasm-windows-studio && cd /opt/kasm-windows-studio

curl -fsSLO https://raw.githubusercontent.com/KasmInnovation/Kasm-windows-studio/main/docker/docker-compose.yml
curl -fsSL  https://raw.githubusercontent.com/KasmInnovation/Kasm-windows-studio/main/docker/.env.example -o .env
```

Edit `.env` and set at minimum:

```bash
APPLIANCE_MASTER_KEY=$(openssl rand -base64 32)   # generate, then paste
ALLOWED_ORIGINS=https://studio.example.com:8443
TLS_COMMON_NAME=studio.example.com
IMAGE_TAG=latest                                   # pin to a specific preview tag
```

Protect the file — it holds the key that decrypts every stored credential:

```bash
chmod 600 .env
```

Start:

```bash
docker compose up -d
docker compose ps
```

Pin `IMAGE_TAG` to an explicit preview tag rather than `latest` for anything you intend
to reproduce. Preview builds change.

### Verify the install

```bash
# Health endpoint
curl -kfsS https://localhost:8443/api/health

# Studio can reach Kasm from inside the container
docker compose exec appliance curl -kfsS https://kasm.example.com/api/__healthcheck
```

---

## TLS certificates

The appliance terminates HTTPS itself. On first start an init container generates a
self-signed certificate into the `ssl-data` volume.

**Replace it for anything beyond evaluation.** Browsers will refuse or warn on a
self-signed certificate, and administrators clicking through that warning is a habit
worth not building.

To install your own certificate and key:

```bash
docker compose down

VOL=$(docker volume ls -q | grep ssl-data)
docker run --rm -v "$VOL":/ssl -v "$PWD/certs":/in alpine:3.20 sh -c '
  cp /in/fullchain.pem /ssl/cert.pem &&
  cp /in/privkey.pem  /ssl/key.pem  &&
  chown 1000 /ssl/cert.pem /ssl/key.pem &&
  chmod 644 /ssl/cert.pem && chmod 600 /ssl/key.pem'

docker compose up -d
```

The key must be readable by uid 1000 — the account the appliance runs as — and should
not be readable by anyone else. The init container leaves an existing certificate
untouched, so this survives restarts.

Alternatively, run Studio behind your own reverse proxy and terminate TLS there.

---

## Windows guest templates

For templates used by the hypervisor and cloud pool workflows:

- Windows Server 2019/2022, or Windows 10/11 Enterprise
- Guest tools installed and running (VMware Tools, Nutanix Guest Tools, or the
  provider's equivalent) — required for guest script execution
- PowerShell 5.1 or later, execution policy `RemoteSigned` or `Bypass`
- Sysprepped/generalised, so each clone gets its own identity
- Sizing: 2 vCPU / 4 GB / 50 GB minimum; 4 vCPU / 8 GB / 80 GB recommended

If you intend to use storage mappings for user data, install **WinFsp** in the template.
Kasm's storage mapping uses rclone, which depends on the WinFsp driver. Without it, a
server enrols and looks healthy while every mapping silently fails to mount.

---

## Backup and restore

Two things matter, and you need both — one is useless without the other.

| What | Where | Why |
| --- | --- | --- |
| `.env` | Your install directory | Holds `APPLIANCE_MASTER_KEY`. Without it the database's credentials cannot be decrypted. |
| `kasm-data` volume | Docker volume | The appliance database — stacks, profiles, audit log, encrypted credentials. |

**Back up:**

```bash
cp .env /secure/backup/kasm-studio.env

docker compose stop appliance
docker run --rm \
  -v "$(docker volume ls -q | grep kasm-data)":/data \
  -v /secure/backup:/backup \
  alpine:3.20 tar czf /backup/kasm-studio-$(date +%F).tar.gz -C /data .
docker compose start appliance
```

**Restore** onto a fresh host by putting the original `.env` back, then extracting the
archive into the `kasm-data` volume before the first start. The schema is brought up to
date automatically on start, so a backup from an older preview build restores onto a
newer one.

The `redis-data` volume holds transient queue state and does not need backing up.

---

## Upgrading

```bash
cd /opt/kasm-windows-studio
cp .env /secure/backup/               # and take a volume backup, as above
docker compose pull
docker compose up -d
```

Use `up -d`, not `restart` — `restart` reuses the old image.

The schema migrates automatically on start. A failed migration stops the container
rather than starting it in a broken state, so check `docker compose logs appliance` if
it does not come back.

---

## Uninstalling

```bash
docker compose down          # stop, keep data
docker compose down -v       # stop and delete all volumes — irreversible
```

Neither removes pools, workspaces or servers already created in Kasm. Remove those in
the Kasm admin UI if you want a clean environment.
