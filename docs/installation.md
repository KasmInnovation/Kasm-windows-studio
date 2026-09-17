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

Create a dedicated API credential in the Kasm admin portal. Using a dedicated credential
rather than a personal admin account keeps Studio's changes distinguishable from a
human's in Kasm's own audit trail.

Grant these areas. The endpoint column is what Studio actually calls, so you can match
it against the permission names your Kasm version uses if the labels differ.

| Permission area | Access | Endpoints Studio calls |
| --- | --- | --- |
| **Server Pools** | Read, Write | `get/create/update/delete_server_pool` |
| **Servers** | Read, Write | `get_servers`, `get/create/update/delete_server` |
| **Server Templates** | Read, Write | `get/create/update/delete_server_template` — enrollment tokens |
| **Workspaces (Images)** | Read, Write | `get/create/update/delete_image`, `get/add/remove_images_group` |
| **Autoscale** | Read, Write | `get/create/update/delete_autoscale_config`, `update/delete_vm_provider_config` |
| **Groups** | Read, Write | `get_groups`, `create_group` — workspace visibility and Trust Profile assignment |
| **Users** | Read | `get_users` |
| **Sessions** | Read | `get_kasms` — checked before deleting a workspace in use |
| **Settings / Zones** | Read | `get_zones`, `get_settings`, `system_info`, `get_ldap_configs` |

### Additional permissions for Trust Profiles

Trust Profiles need storage, which is a separate set of permissions people routinely
miss — the feature then fails in ways that look like something else.

| Permission area | Access | Endpoints Studio calls |
| --- | --- | --- |
| **Storage Providers** | Read, Create | `get_storage_providers` |
| **Storage Mappings** | Read, Create, Update, Delete | `get/create/update/delete_storage_mapping` |
| **File Mappings** | Read, Create, Update | `get_file_mappings`, `create_file_map`, `update_file_map` — session scripts |

> [!IMPORTANT]
> **Storage mapping endpoints reject an API key.** Verified against a live deployment:
> `get_storage_mapping` and `update_storage_mapping` return **401 to an API key** no
> matter which permissions you grant it, while `get_storage_providers` and the file-map
> endpoints accept one. Those two require an authenticated **admin session**.
>
> So if you configure Studio with an API key **and no administrator username and
> password**, every storage mapping call fails and Trust Profiles cannot work. Supply
> the admin username and password in the Kasm API step as well as the key.

### If you skip these

| Missing | Symptom |
| --- | --- |
| Storage Providers / Mappings | Trust Profiles configure without error but no profile is ever saved or restored |
| Admin username + password | As above, with a 401 in the logs that reads like a credential problem |
| File Mappings | Session scripts never reach the VM; profiles silently do nothing |
| Groups | The workspace deploys but is visible to nobody |
| Sessions | Workspace deletion cannot check for active sessions first |

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
