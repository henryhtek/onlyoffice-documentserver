# ONLYOFFICE Document Server – Self-Hosted Docker Deployment

Self-hosted **ONLYOFFICE Docs (Document Server)** designed to integrate with an existing **Nextcloud** instance and **Nginx Proxy Manager (NPM)**.

**Target environment**
- Ubuntu server on Raspberry Pi (or any Linux host)
- Nextcloud already running on `172.16.100.2` (ports 80/443) with local FQDN + self-signed certificate
- Nginx Proxy Manager already running, listening on `172.16.100.14:443`, using Docker network `outside`
- ONLYOFFICE placed behind NPM on the same Docker network

---

## Architecture Overview

<div align="center">

```
+--------------------------------------------------------------------------+
|                            Local Network                                 |
|                                                                          |
|  Clients / Browser                                                       |
|        |                                                                 |
|        |  https://onlyoffice.local                                       |
|        |  (or your chosen FQDN)                                          |
|        v                                                                 |
|  +------------------------------+                                        |
|  |  Nginx Proxy Manager (NPM)   |                                        |
|  |  172.16.100.14:443           |                                        |
|  |  Docker network: outside     |                                        |
|  |  - SSL termination           |                                        |
|  |  - WebSocket support         |                                        |
|  +--------------+---------------+                                        |
|                 |  HTTP (internal)                                       |
|                 |  Forward to container:80                               |
|                 v                                                        |
|  +------------------------------+    +------------------------------+    |
|  |  ONLYOFFICE Document Server  |    |  Nextcloud                   |    |
|  |  container name:             |<-->|  172.16.100.2:80/443         |    |
|  |  onlyoffice-documentserver   |JWT |  (existing, self-signed)     |    |
|  |  network: outside            |    |                              |    |
|  |  port 80 (HTTP only)         |    |                              |    |
|  +------------------------------+    +------------------------------+    |
|                                                                          |
|  Volumes (configurable via .env):                                        |
|    DATA_VOLUME, LOGS_VOLUME, LIB_VOLUME                                  |
+--------------------------------------------------------------------------+
```

</div>

**Key design decisions**
- ONLYOFFICE joins the existing external Docker network `outside` (same network used by NPM).
- No host ports 80/443 are published – NPM is the only entry point.
- SSL is terminated at NPM (recommended). The Document Server container runs plain HTTP on port 80.
- Local FQDN and all volume paths are defined in `.env` with defaults in `docker-compose.yml`.
- JWT protects the Document Server API.
- `USE_UNAUTHORIZED_STORAGE=true` allows Document Server to call back to Nextcloud when Nextcloud presents a self-signed certificate.

---

## Prerequisites

| Requirement | Notes |
|-------------|-------|
| Docker Engine | ≥ 20.10 (recommended latest) |
| Docker Compose | v2 (`docker compose`) |
| Host resources | ≥ 4 GB RAM, 2 CPU cores, ≥ 2 GB free disk |
| Existing Docker network | `outside` (already used by Nginx Proxy Manager) |
| Nginx Proxy Manager | Running and reachable on `172.16.100.14:443` |
| Nextcloud | Running on `172.16.100.2:80/443` with its own local FQDN |
| Local DNS / hosts | Entry for the ONLYOFFICE FQDN pointing to the NPM IP (`172.16.100.14`) |

### Verify the external network exists

```bash
docker network ls | grep outside
# should show the network used by NPM
```

If it does not exist (rare), create it:

```bash
docker network create outside
```

---

## Complete Installation

### 1. Place the project on the host

```bash
cd /opt          # or any preferred location
# copy / place the onlyoffice-documentserver folder here
cd onlyoffice-documentserver
```

### 2. Create and edit the environment file

```bash
cp .env.example .env
vim .env
```

**Important values**

```env
# Local FQDN that clients and Nextcloud will use
ONLYOFFICE_FQDN=onlyoffice.local

# Must match the network used by Nginx Proxy Manager
DOCKER_NETWORK=outside

# Volume locations (relative to the compose file or absolute paths)
DATA_VOLUME=./data
LOGS_VOLUME=./logs
LIB_VOLUME=./lib

# Strong random secret – MUST be identical in Nextcloud connector
JWT_SECRET=$(openssl rand -hex 32)   # generate once and paste the value

USE_UNAUTHORIZED_STORAGE=true
```

### 3. Create data directories

```bash
mkdir -p data logs lib
# certs/ is optional when SSL is terminated at NPM
```

### 4. Start the stack

```bash
docker compose up -d
```

Watch logs until the healthcheck becomes healthy:

```bash
docker compose logs -f documentserver
docker compose ps
```

### 5. Configure Nginx Proxy Manager

In the NPM web UI create a new **Proxy Host**:

| Field | Value |
|-------|-------|
| Domain Names | `onlyoffice.local` (must match `ONLYOFFICE_FQDN`) |
| Scheme | `http` |
| Forward Hostname / IP | `onlyoffice-documentserver` (container name) |
| Forward Port | `80` |
| Cache Assets | Off (recommended) |
| Block Common Exploits | On |
| Websockets Support | **On** (required for collaborative editing) |
| Access List | None (or restrict as needed) |

**SSL tab**
- Enable SSL
- Use an existing certificate or request/generate a self-signed certificate for `onlyoffice.local`
- Force SSL → On
- HTTP/2 → On

**Advanced tab** (usually not required, but useful if you encounter issues):

```nginx
# Optional extra headers
proxy_set_header Upgrade $http_upgrade;
proxy_set_header Connection $http_connection;
proxy_http_version 1.1;
```

Save the Proxy Host.

### 6. Local DNS / hosts entry

On every client machine (and on the Nextcloud host if needed) ensure the FQDN resolves to the NPM IP:

```bash
# /etc/hosts  or local DNS
172.16.100.14   onlyoffice.local
```

### 7. Verify Document Server through NPM

```bash
curl -k https://onlyoffice.local/healthcheck
# expected output: true
```

Open `https://onlyoffice.local` in a browser (accept the self-signed certificate warning if using a self-signed cert in NPM).

### 8. Integrate with Nextcloud (host install, not a container)

> **Important**  
> Nextcloud is installed directly on the host (`172.16.100.2`), **not** inside Docker.  
> Therefore the host **cannot** resolve Docker container names such as `onlyoffice-documentserver`.  
> All addresses used by Nextcloud must be reachable via the host network (NPM FQDN or host IP).

1. Install the **ONLYOFFICE** connector app in Nextcloud  
   (Apps → Office & text → ONLYOFFICE → Download and enable).

2. Configure the connector (Settings → ONLYOFFICE):

   | Setting | Value | Notes |
   |---------|-------|-------|
   | **Document Editing Service address** | `https://onlyoffice.local` | Public URL that browsers and Nextcloud use (via NPM) |
   | **Secret key** | Same value as `JWT_SECRET` in `.env` | Must match exactly |
   | **Disable certificate verification (insecure)** | Enabled | Required when using self-signed certificates |
   | **Advanced → Address for internal requests from Nextcloud to Document Server** | `https://onlyoffice.local` | **Do not** use the Docker container name. Host Nextcloud talks to Document Server through NPM. |
   | **Advanced → Address for internal requests from Document Server to Nextcloud** | Nextcloud public URL, e.g. `https://nextcloud.local` or `https://172.16.100.2` | Must be reachable **from inside** the ONLYOFFICE container |

3. Apply required system settings on the Nextcloud host:

   ```bash
   # Adjust the path to your Nextcloud installation if different
   sudo -u www-data php /var/www/nextcloud/occ config:system:set onlyoffice verify_peer_off --value=true --type=boolean
   sudo -u www-data php /var/www/nextcloud/occ config:system:set allow_local_remote_servers --value=true --type=boolean
   ```

   Alternatively edit `config/config.php` and add:

   ```php
   'onlyoffice' => array(
       'verify_peer_off' => true,
   ),
   'allow_local_remote_servers' => true,
   ```

4. Verify connectivity in both directions before saving:

   ```bash
   # From the Nextcloud host → Document Server (via NPM)
   curl -k https://onlyoffice.local/healthcheck
   # expected: true

   # From inside the ONLYOFFICE container → Nextcloud
   docker exec -it onlyoffice-documentserver curl -k https://nextcloud.local/status.php
   # or use the host IP if the FQDN is not resolvable inside the container
   docker exec -it onlyoffice-documentserver curl -k https://172.16.100.2/status.php
   ```

5. Save the connector settings and run the official check:

   ```bash
   sudo -u www-data php /var/www/nextcloud/occ onlyoffice:documentserver --check
   ```

6. Open any Office document in Nextcloud – it should open in the ONLYOFFICE editor.

#### Why these values?

| Direction | Why this address |
|-----------|------------------|
| Browser / Nextcloud → Document Server | `https://onlyoffice.local` goes through NPM (`172.16.100.14`). Works from the host and from clients. |
| Document Server → Nextcloud | Document Server runs in Docker. It must reach the **host** Nextcloud via its public FQDN or IP (`172.16.100.2`). Container names are not visible to the host, and the host is not on the Docker network by name. |

If the save still fails, check Document Server logs while clicking Save:

```bash
docker compose logs -f documentserver
```

Look for self-signed certificate errors or connection refused. `USE_UNAUTHORIZED_STORAGE=true` (already set in the compose file) allows Document Server to accept Nextcloud’s self-signed certificate.

---

## Useful Commands

```bash
# Start / stop / restart
docker compose up -d
docker compose stop
docker compose restart

# Logs
docker compose logs -f documentserver

# Enter the container
docker compose exec documentserver bash

# Update to the newest image
docker compose pull
docker compose up -d

# Health check (via NPM)
curl -k https://onlyoffice.local/healthcheck
```

---

## Clean-up / Uninstallation

### Soft stop (keeps data)

```bash
docker compose down
```

### Full removal of containers and network attachment (data kept)

```bash
docker compose down --remove-orphans
```

### Complete wipe (including data volumes)

```bash
docker compose down -v --remove-orphans
# optionally remove the project folder
sudo rm -rf /opt/onlyoffice-documentserver
```

### Remove the Docker image (optional)

```bash
docker rmi onlyoffice/documentserver:latest
```

---

## Troubleshooting

| Symptom | Likely cause | Fix |
|---------|--------------|-----|
| `docker compose up` fails with network error | Network `outside` does not exist | `docker network create outside` or set `DOCKER_NETWORK` to the correct name |
| NPM cannot reach the container | Container not on the same network | Verify `docker network inspect outside` contains `onlyoffice-documentserver` |
| Healthcheck fails | Container still starting | Wait 1–2 minutes; check logs |
| Nextcloud: error when saving connector settings | JWT mismatch, wrong internal URL, or self-signed cert | Use `https://onlyoffice.local` for both public and internal Document Server URLs (host cannot resolve container names). Enable `verify_peer_off` and `allow_local_remote_servers`. |
| Nextcloud: “ONLYOFFICE cannot be reached” | JWT mismatch or wrong public URL | Verify `JWT_SECRET` is identical; public URL must be the NPM FQDN |
| Document Server cannot download from Nextcloud | Self-signed cert rejected or Nextcloud unreachable from container | Ensure `USE_UNAUTHORIZED_STORAGE=true`. From the container run `curl -k https://172.16.100.2/status.php` |
| Collaborative editing / real-time fails | Websockets not enabled in NPM | Enable “Websockets Support” on the Proxy Host |
| Browser certificate warning | Expected with self-signed cert in NPM | Accept exception or import the CA |

---

## Security Notes

- Change the default `JWT_SECRET` immediately and keep it identical in Nextcloud.
- Prefer SSL termination at Nginx Proxy Manager; keep the Document Server container on plain HTTP.
- Restrict access to the NPM Proxy Host if the service should not be reachable by the whole LAN.
- Keep the Document Server image updated (`docker compose pull`).

---

## File Layout

```
onlyoffice-documentserver/
├── docker-compose.yml      # Main compose file (project name: onlyoffice)
├── .env.example            # Template – copy to .env
├── .env                    # Your local settings (not committed)
├── data/                   # Persistent data  (DATA_VOLUME)
├── logs/                   # Logs             (LOGS_VOLUME)
├── lib/                    # Cache / lib      (LIB_VOLUME)
└── README.md               # This file
```

---

**Ready for production internal use with Nextcloud + Nginx Proxy Manager.**  
After following the steps above, ONLYOFFICE will be reachable at your chosen local FQDN through NPM and fully integrated with the existing Nextcloud instance.
