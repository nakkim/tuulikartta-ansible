Tuulikartta Server
==================

An Ansible role for deploying and managing the Tuulikartta weather observation application server infrastructure. This role sets up Docker, manages users and groups, deploys Docker containers for the Tuulikartta server and processor, and configures automated data processing via cron jobs.

Requirements
------------

- Ansible 2.1 or higher
- Target system: CenTOS 10 or compatible
- Python 3 on the target host
- Ansible collections:
  - `community.docker` (for Docker container management)

You can install required collections with:
```bash
ansible-galaxy collection install community.docker
```

Role Variables
--------------

### Docker Images (vars/docker-images.yml)

The Docker images and their versions are defined in `vars/docker-images.yml`:

```yaml
tuulikartta_processor:
  image: "docker.io/nakkim/tuulikartta-processor"
  tag: "v1.0.0"

tuulikartta_server:
  image: "docker.io/nakkim/tuulikartta-server"
  tag: "v1.0.2"
```

### Users and Groups (vars/)

Users are defined in `vars/users.yml`:
- **runner**: Primary user for running the Tuulikartta services (UID: 4200)

Groups are defined in `vars/groups.yml`:
- **runner**: Primary group for the runner user (GID: 3600)

### Defaults (defaults/main.yml)

Currently, there are no default variables set. All configuration is managed through the vars files.

What This Role Does
-------------------

1. **User Management**: Creates the `runner` user and required groups with specific UIDs/GIDs
2. **Directory Setup**: Creates necessary directories for data storage (`/data/processed`)
3. **Docker Installation**: Installs and configures Docker on the target system
4. **Container Deployment**:
   - Pulls the tuulikartta-processor Docker image
   - Deploys and runs the tuulikartta-server container on port 80
   - Mounts `/data/processed` for serving processed wind data
5. **Cron Jobs**: Sets up automated data processing job that runs every minute

Dependencies
------------

None. This is a standalone role.

Example Playbook
----------------

### Basic Usage

```yaml
---
- hosts: tuulikartta_servers
  gather_facts: true
  tasks:
    - name: Install tuulikartta server
      include_tasks: tasks/main.yml
  serial: 1
```

### Using as a Role

```yaml
---
- hosts: tuulikartta_servers
  become: yes
  roles:
    - tuulikartta-server
```

### Running the Playbook

```bash
ansible-playbook \
  --private-key=/path/to/private/key \
  --inventory=inventories/production/hosts.yml \
  --user=root \
  tuulikartta-server.yml
```

Inventory Setup
--------------

Configure your target servers in `inventories/production/hosts.yml`:

```yaml
---
all:
  children:
    tuulikartta_servers:
      hosts:
        your.server.ip.address:
```

Architecture
-----------

- **Tuulikartta Server**: Web server container serving web application on port 80
- **Tuulikartta Processor**: Processing container running a python script run via cron every minute to process weather observation data
- **Data Directory**: `/data/processed` stores the processed weather observation data files

The processor container runs periodically to fetch and process new weather observation data, storing results in `/data/processed`. The server container continuously serves this data via HTTP.

Files Structure
--------------

```
.
├── defaults/           # Default variables (currently empty)
├── files/             # Static files
├── handlers/          # Ansible handlers
├── inventories/       # Inventory files
│   └── production/
├── meta/              # Role metadata
├── tasks/             # Task definitions
│   ├── cron/         # Cron job setup
│   ├── directories/  # Directory creation
│   ├── docker/       # Docker installation and container management
│   ├── packages/     # Package management
│   └── users/        # User and group management
├── templates/         # Jinja2 templates
├── tests/            # Test playbooks
└── vars/             # Variable definitions

```

License
-------

MIT

Author Information
------------------

Maintained as part of the Tuulikartta project.
