# Tuulikartta Server

An Ansible role for deploying and managing the Tuulikartta weather observation application server. It provisions a CentOS 10 host with Docker, deploys the application container, installs Nginx as an HTTPS reverse proxy, obtains a Let's Encrypt TLS certificate via Certbot, and sets up automated wind-data processing via cron.

## Requirements

- Ansible 2.10 or higher
- Target system: CentOS 10 (or compatible RHEL-based distro with `dnf`)
- Python 3 on the target host
- SSH access as `root` (or a user with `become` privileges)
- The domain name(s) in `roles/tuulikartta_server/vars/certbot.yml` must resolve to the server's public IP **before** running the play (required for Let's Encrypt certificate issuance)
- Ansible collections:

```bash
ansible-galaxy collection install community.docker
```

## Architecture

```
Internet
  │
  ├── :80  ──► Nginx (host)  ──► redirect to HTTPS
  └── :443 ──► Nginx (host)  ──► TLS termination
                                   │
                              proxy_pass http://127.0.0.1:8080
                                   │
                         tuulikartta-server container
                         (binds 127.0.0.1:8080, serves /var/www/html/data)
```

- **Nginx** runs on the host, owns ports 80 and 443, terminates TLS, and reverse-proxies to the app container.
- **tuulikartta-server** Docker container serves the wind-data web application on `127.0.0.1:8080`.
- **tuulikartta-processor** Docker container runs every minute via cron (as the `runner` user), fetches and processes weather observation data, and writes results to `/data/processed`.
- **Certbot** (`python3-certbot-nginx` plugin) obtains and auto-renews a Let's Encrypt certificate. Renewal runs via cron at `03:21` daily.

## What This Role Does

1. **User & group management** — creates the `runner` user (UID 4200 / GID 3600) used for cron jobs
2. **Directory setup** — creates `/data/processed` for processed wind data
3. **Docker installation** — installs Docker CE, the Compose plugin, and the Python Docker SDK
4. **Container deployment** — pulls and starts `tuulikartta-server` (bound to `127.0.0.1:8080`) with `/data/processed` and `/etc/letsencrypt` mounted
5. **Cron job** — runs `tuulikartta-processor` every minute as `runner`
6. **System hardening** — disables and removes unnecessary services (`rpcbind`, `nfs-server`)
7. **Nginx + TLS** — installs Nginx, deploys the reverse-proxy vhost, obtains a Let's Encrypt cert via `certbot --nginx`, and installs a daily renewal cron job

## Role Variables

### `roles/tuulikartta_server/vars/certbot.yml` — domain and TLS settings

| Variable               | Default                                   | Description                                              |
| ---------------------- | ----------------------------------------- | -------------------------------------------------------- |
| `domain_name`          | `test-nakkim.dy.fi`                       | Primary domain for the certificate and Nginx vhost       |
| `domain_aliases`       | `[www.test-nakkim.dy.fi]`                 | Additional SANs added to the certificate                 |
| `all_domains`          | derived                                   | Combined list of `domain_name` + `domain_aliases`        |
| `certbot_email`        | `you@example.com`                         | Email address for Let's Encrypt account / expiry notices |
| `letsencrypt_dir`      | `/etc/letsencrypt`                        | Let's Encrypt base directory                             |
| `letsencrypt_live_dir` | `/etc/letsencrypt/live/{{ domain_name }}` | Path to the live certificate files                       |
| `renew_cron_minute`    | `21`                                      | Minute field for the renewal cron job                    |
| `renew_cron_hour`      | `3`                                       | Hour field for the renewal cron job                      |

### `roles/tuulikartta_server/vars/docker-images.yml` — container image versions

| Variable                      | Value                                    |
| ----------------------------- | ---------------------------------------- |
| `tuulikartta_server.image`    | `docker.io/nakkim/tuulikartta-server`    |
| `tuulikartta_server.tag`      | `v1.0.2`                                 |
| `tuulikartta_processor.image` | `docker.io/nakkim/tuulikartta-processor` |
| `tuulikartta_processor.tag`   | `v1.0.0`                                 |

### `roles/tuulikartta_server/vars/users.yml` / `roles/tuulikartta_server/vars/groups.yml`

- **runner** user: UID 4200, primary group `runner` (GID 3600), secondary group `wheel`

## Pre-flight Checklist

1. Point your domain's A record to the server IP.
2. Edit `roles/tuulikartta_server/vars/certbot.yml` — set `domain_name`, `domain_aliases`, and `certbot_email`.
3. Edit `roles/tuulikartta_server/vars/docker-images.yml` — pin the image tags you want to deploy.
4. Edit `inventories/production/hosts.yml` — add your server's IP or hostname.

## Inventory Setup

```yaml
# inventories/production/hosts.yml
---
all:
  children:
    tuulikartta_servers:
      children:
        tuulikartta_main_server: # runs shared services (GeoServer, SmartMet Server, etc.)
          hosts:
            203.0.113.10: # replace with your server IP
        tuulikartta_aux_servers: # use shared services from the main server
          hosts:
            # 203.0.113.11:
```

## Running the Playbook

```bash
ansible-playbook \
  --private-key ~/.ssh/your_key \
  --inventory inventories/production/hosts.yml \
  --user root \
  --ask-vault-pass \
  tuulikartta-server.yml
```

To run only a specific part of the role use tags or limit to specific tasks. To do a dry-run first:

```bash
ansible-playbook \
  --private-key ~/.ssh/your_key \
  --inventory inventories/production/hosts.yml \
  --user root \
  --check \
  tuulikartta-server.yml
```

## GeoServer (main server)

The `tuulikartta_geoserver` role deploys [tuulikartta-geoserver](https://github.com/nakkim/tuulikartta-geoserver) on the main server: `geoserver`, `radar-poller`, `geoserver-proxy` and a one-shot `geoserver-init` container, all on the `tuulikartta` docker network.

- The granule index uses the shared `postgres` container (PostGIS image), in its own `radar` database owned by the `radar` user.
- The radar-poller and geoserver-init images come from ghcr.io, pinned to `geoserver_version`. The role also checks out the repo to `/opt/tuulikartta-geoserver` at `geoserver_repo_version`, for `proxy/nginx.conf` and the initial GeoServer data dir.
- `/data/geoserver/data_dir` is seeded from the repo on first deploy only. After that, GeoServer owns it.
- GeoServer admin (web UI, REST): `127.0.0.1:8081`, reach it with `ssh -L 8081:127.0.0.1:8081 <server>`.
- WMS/WMTS proxy: `127.0.0.1:8088`. Host nginx routes `https://tuulikartta.info/geoserver/` there (TLS ends at the Hetzner load balancer), e.g. `https://tuulikartta.info/geoserver/wms?service=WMS&version=1.3.0&request=GetCapabilities`.

Set `geoserver_admin_user`, `geoserver_admin_password` and `geoserver_db_password` in `group_vars/tuulikartta_main_server.yml` (encrypt the passwords with vault). Other settings are in `roles/tuulikartta_geoserver/defaults/main.yml`.

## Managing Notifications

`notifications.yml` inserts rows into the app's `notifications` table on the main server. Copy `notification-data/data.yml`, edit the rows, then run:

```bash
ansible-playbook \
  --private-key ~/.ssh/your_key \
  --inventory inventories/production/hosts.yml \
  --user root \
  -e @notification-data/your-file.yml \
  notifications.yml
```

To remove all rows, add `-e notifications_clear=true`. The playbook asks for confirmation; add `-e notifications_clear_confirmed=true` to skip it. Combined with a data file, clearing runs first, so the file replaces all rows.

## Files Structure

```
.
├── ansible.cfg
├── tuulikartta-server.yml              # playbook: role on all servers, postgres on main server
├── notifications.yml                   # playbook: insert or remove notification rows
├── notification-data/
│   └── data.yml                     # example notification rows
├── group_vars/
│   └── tuulikartta_main_server.yml     # postgres credentials (password vault-encrypted)
├── inventories/
│   └── production/
│       └── hosts.yml                   # main and aux servers
└── roles/
    ├── tuulikartta_geoserver/          # GeoServer, radar poller and WMS/WMTS proxy (main server only)
    │   ├── defaults/main.yml           # repo version, paths, ports, database name
    │   ├── meta/main.yml
    │   ├── tasks/                      # main, database, source (checkout + directories), containers
    │   └── vars/main.yml               # image names and tags, layer names
    └── tuulikartta_server/
        ├── defaults/
        │   └── main.yml                # (empty — all config lives in vars/)
        ├── files/
        │   └── docker-daemon.json      # Docker daemon settings
        ├── handlers/
        │   └── main.yml                # reload nginx, reload systemd, restart docker
        ├── meta/
        │   └── main.yml
        ├── tasks/
        │   ├── main.yml                # top-level task orchestration
        │   ├── certbot/
        │   │   └── main.yml            # nginx install, cert issuance, renewal cron (deprecated)
        │   ├── cron/
        │   │   └── main.yml            # tuulikartta-processor cron job
        │   ├── directories/
        │   │   └── main.yml            # /data/processed creation
        │   ├── docker/
        │   │   ├── main.yml
        │   │   ├── install-docker.yml  # Docker CE + SDK install
        │   │   ├── daemon-settings.yml # Docker daemon config
        │   │   └── docker-containers.yml # docker network, pull & run tuulikartta-server
        │   ├── packages/
        │   │   └── main.yml            # system packages
        │   ├── postgres/
        │   │   └── main.yml            # postgres container + prisma migrations (main server only)
        │   ├── system/
        │   │   └── main.yml            # disable unused services
        │   └── users/
        │       ├── main.yml
        │       ├── add-required-groups.yml
        │       └── add-users.yml
        ├── templates/
        │   ├── nginx-tuulikartta-http.conf.j2
        │   └── nginx-tuulikartta.conf.j2   # Nginx HTTP→HTTPS redirect + reverse proxy vhost
        ├── tests/
        │   ├── inventory
        │   └── test.yml
        └── vars/
            ├── certbot.yml             # domain, TLS, and renewal settings
            ├── docker-images.yml       # image names and tags
            ├── docker-packages.yml     # Docker CE package list
            ├── groups.yml              # system groups
            ├── packages.yml            # system packages
            └── users.yml               # system users
```

## Dependencies

None. The role has no dependencies on other roles.

## License

MIT

## Author Information

Maintained as part of the Tuulikartta project.
