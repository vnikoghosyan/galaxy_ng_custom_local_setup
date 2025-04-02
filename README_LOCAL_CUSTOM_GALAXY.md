## Content

> Initial Source: (https://github.com/ansible/galaxy_ng)

- [Galaxy NG Custom Local Development Setup](#galaxy-ng-custom-local-development-setup)

- [Galaxy NG Minimal File-set](#galaxy-ng--minimal-file-set)


---
# Galaxy NG Custom Local Development Setup

This document describes the custom modifications made to the original Galaxy NG repository in order to run a reliable local development environment with persistent data, secure permissions, and support for container restart without data loss.

---

## ✨ Goals Achieved

- Setup a fully functioning local Galaxy NG development environment using Docker Compose.
- Run API, Content, Worker, Manager, and Nginx containers in a dev context.
- Publish namespaces and collections locally.
- Configure persistent data volumes for PostgreSQL and Pulp.
- Resolve permission issues without using root inside containers unnecessarily.
- Disable IPv6 and make services bind only to IPv4.
- Provide custom nginx configuration to support /api, /ui, /content, etc.
- Add structured configuration via `standalone.yaml`.

---

## ✅ Key Customizations

### 1. **Custom `standalone.yaml`**

- Defined full service stack (api, content, worker, manager, postgres, redis, nginx, etc).
- Enabled debug mode using `stdin_open: true` and `tty: true`.
- Defined `common-env` section with environment variables for all services.
- Used volumes for `/etc/pulp/certs`, `/var/lib/pulp`, and new PostgreSQL persistence.
- All services configured to wait for migrations to complete before startup.
- Nginx exposes Galaxy NG on `http://localhost/`.

### 2. **Persistent Volumes**

Added named volumes:

```yaml
docker-compose volumes:
  var_lib_pulp:
  etc_pulp_certs:
  pgdata:           # New for PostgreSQL persistence
```

Mount in `postgres` service:

```yaml
volumes:
  - pgdata:/var/lib/postgresql/data
```

This ensures all published collections, users, and settings persist after container restarts.

### 3. **Permission Fixes and `galaxy` User UID/GID**

- Created a dedicated `galaxy` user with UID and GID of `1050`, avoiding collision with host users.
- Modified `Dockerfile.base` to accept `USER_ID` and `USER_GROUP` args.
- Adjusted `standalone.yaml` builds to pass those args dynamically.
- Ownership of mounted source and cert directories updated for UID/GID 1050.
- All internal scripts are now executable for `galaxy` user.

### 4. **Nginx Configuration**

Custom `dev/nginx/nginx.conf`:

- Serves `api`, `content`, and `ui` routes using proxy_pass.
- Static files (e.g., UI) proxied from `galaxy_ng` app.
- Uses port 5001 internally, exposed as port 80 on the host.

### 5. **Disabling IPv6**

- Gunicorn was failing to bind on IPv6 (::), so all apps are forced to bind to `0.0.0.0`.
- Docker networks configured with:

```yaml
networks:
  service-mesh:
    driver_opts:
      com.docker.network.enable_ipv6: "false"
```

### 6. **Service Health and Restart Policies**

- Nginx configured with `restart: on-failure` for robustness.
- PostgreSQL has `healthcheck` to ensure readiness.
- Manager and worker containers include debug output and readiness banners.

---

## ⚡ Usage Instructions

### Build and Start Environment
```bash
docker compose -f dev/compose/standalone.yaml up -d --build
```

### Stop Without Losing Data
```bash
docker compose -f dev/compose/standalone.yaml down
```

### Rebuild With Clean Volumes (will remove published content!)
```bash
docker compose -f dev/compose/standalone.yaml down -v
```

### Interact with Manager Container
```bash
docker compose -f dev/compose/standalone.yaml exec manager /bin/bash
```

---

## 🎓 Publishing a Collection

1. Prepare your collection tarball.
2. Run:
```bash
ansible-galaxy collection publish my_namespace-my_collection-1.0.0.tar.gz --server http://localhost/api/galaxy/
```
3. Use credentials: `admin:admin`.

---

## 🔗 References

- [Galaxy NG GitHub](https://github.com/ansible/galaxy_ng)
- [Pulp Documentation](https://docs.pulpproject.org/)

---

## 🛡️ Security Notes

- Avoid using the `root` user in containers in production environments.
- Always use trusted GPG keys for signing collections.
- Ensure secrets are stored securely (e.g., key passwords).

## 🚀 Next Steps

- Automate volume backups for PostgreSQL and Pulp.
- Add bind mounts to external directories for more transparent storage.
- Configure external authentication or SSO integration.
- Integrate with Galaxy CLI or deploy to staging.
- Upgrade PostgreSQL and Redis versions

---


# Galaxy NG — Minimal File-set

This document describes a **minimal and customized Galaxy NG setup**, tailored for local development. It is based on the [official Galaxy NG repository](https://github.com/ansible/galaxy_ng) and has been **refactored** to support:

- Lightweight container builds
- root user execution for services (to avoid permission errors)
- Persistent volume management
- IPv6-free environments
- Reliable startup behavior
- Clean project structure
- Minimal file footprint
- GPG signing support

---

## ✅ Key Features & Modifications

### 🔹 1. Minimal File Structure

Only the required directories and files are retained from the original repository:

#### 📁 Project Structure Overview

Only the **required** components are retained from the original repo.

This is a **minimal reproducible Galaxy NG development setup** based on the official `ansible/galaxy_ng` repository. It is optimized for local development and containerized deployment using Docker Compose.

To build and run this setup, ensure the following **essential directory and file structure** is present:

```bash
.
├── Dockerfile                         # Main Dockerfile for 'base' image
├── ansible.cfg                        # Required for Ansible config path setup
├── dev/                               # Includes Docker Compose, signing, and GPG setup scripts
│   ├── Dockerfile.base
│   ├── common
│   │   ├── DUMP_CONTAINER_ENV_VARS.sh
│   │   ├── RUN_INTEGRATION.sh
│   │   ├── RUN_INTEGRATION_STAGE.sh
│   │   ├── RUN_PERFORMANCE.sh
│   │   ├── RUN_PERFORMANCE_STAGE.sh
│   │   ├── __init__.py
│   │   ├── ansible-sign-pub.gpg
│   │   ├── ansible-sign-pub.txt
│   │   ├── ansible-sign.key
│   │   ├── collection_sign.sh
│   │   ├── container_auth_private_key.pem
│   │   ├── container_auth_public_key.pem
│   │   ├── container_sign.sh
│   │   ├── dependency_bumper.py
│   │   ├── docker-compose-ui.yaml
│   │   ├── dynaconf_hooks.py
│   │   ├── galaxy_ng.env
│   │   ├── postgres.env
│   │   ├── settings.py
│   │   ├── setup_test_data.py
│   │   └── tdd.py
│   ├── compose
│   │   ├── Dockerfile.dev
│   │   ├── Dockerfile.nginx
│   │   ├── bin
│   │   ├── database
│   │   ├── signing
│   │   └── standalone.yaml
│   └── nginx
│       └── nginx.conf
├── docker/                            # Entry scripts, settings, and bin utilities
│   ├── bin
│   │   ├── start-api
│   │   ├── start-api-reload
│   │   ├── start-content-app
│   │   ├── start-worker
│   │   ├── start-worker-reload
│   │   ├── wait-for-migrations
│   │   └── wait-for-tcp
│   ├── entrypoint.sh
│   └── etc
│       └── settings.py
├── galaxy-operator/                   # Operator binaries used in image
│   └── bin
│       └── readyz.py
├── galaxy_ng/                         # Main application codebase
│   ├── __init__.py
│   ├── __pycache__
│   │   └── __init__.cpython-311.pyc
│   ├── _vendor
│   │   ├── __init__.py
│   │   ├── __pycache__
│   │   └── automated_logging
│   ├── app
│   │   ├── __init__.py
│   │   ├── __pycache__
│   │   ├── access_control
│   │   ├── api
│   │   ├── auth
│   │   ├── common
│   │   ├── constants.py
│   │   ├── dynaconf_hooks.py
│   │   ├── dynamic_settings.py
│   │   ├── exceptions.py
│   │   ├── management
│   │   ├── metrics_collection
│   │   ├── migrations
│   │   ├── models
│   │   ├── pipelines.py
│   │   ├── renderers.py
│   │   ├── settings.py
│   │   ├── signals
│   │   ├── static
│   │   ├── tasks
│   │   ├── urls.py
│   │   ├── utils
│   │   ├── views.py
│   │   ├── viewsets.py
│   │   └── webserver_snippets
│   ├── contrib
│   │   ├── __init__.py
│   │   └── cloudwatch.py
│   ├── locale
│   │   ├── es
│   │   ├── fr
│   │   ├── ja
│   │   ├── ko
│   │   ├── nl
│   │   └── zh
│   ├── openapi
│   │   ├── __init__.py
│   │   └── __pycache__
│   ├── social
│   │   ├── __init__.py
│   │   └── pipeline
│   ├── tests
│   │   ├── __init__.py
│   │   ├── __pycache__
│   │   ├── constants.py
│   │   ├── integration
│   │   ├── performance
│   │   └── unit
│   └── ui
│       ├── __init__.py
│       ├── __pycache__
│       └── urls.py
├── pyproject.toml                     # Used by PEP 517/518 for editable install
├── requirements/                      # Python requirements for base image
│   ├── requirements.common.txt
│   ├── requirements.dev.txt
│   └── requirements.insights.txt
└── setup.py                           # setuptools configuration with build hooks
```

> ⚠️ Do **not** omit the `galaxy_ng/` or `galaxy-operator/` directories; these are **critical** for a working build.
---
### 🔹 2. `Dockerfile` Customizations

- Base image built using `Dockerfile` (UBI8 + Python 3.12)
- All commands executed as `galaxy` user with `UID/GID=1005`
- Collects static files via `django-admin collectstatic`
- Makes entry scripts and certs group-writable for container flexibility

### 🔹 3. `Dockerfile.dev` for Development Image

- Adds developer tools like `ipython`, `pulp-cli`, `django-extensions`
- Installs GPG signing scripts and keys
- Fixes executable permissions for signing tools
- Uses `COPY --chown=galaxy:galaxy` to ensure ownership alignment

### 🔹 4. `Dockerfile.base` Customizations

- Base image in `dev/compose` directory was built using Python 3.12

### 🔹 5. NGINX Configuration (Port Changed to 80)

- NGINX is configured to **expose Galaxy NG on port 80** on the host.
- The service handles paths like `/api`, `/content`, `/ui`.

Access the UI at: [http://localhost](http://localhost)

### 🔹 6. Compose Customizations in `standalone.yaml`

- Located in: `dev/compose/standalone.yaml`
- Includes services:
  - `api`, `content`, `worker`, `manager`, `migrations`, `postgres`, `redis`, `nginx`
- For services `api`, `content`, `worker`, `manager` the user was changed from `galaxy` to `root` (Due to permission errors with mounted scripts)
- Defines shared environment via `common-env`
- Mounts volumes for:
  - `/etc/pulp/certs`
  - `/var/lib/pulp`
  - PostgreSQL data (`pgdata`)
- Ensures services wait for dependencies (migrations, DB readiness)

### 🔹 7. Disable IPv6

- All services use `0.0.0.0` to bind only on IPv4
- Gunicorn and nginx avoid using `::` to ensure compatibility with VMs where IPv6 is disabled

---

## 🛠️ How to Build & Run

### 🧱 Step 1: Build and Launch

```bash
cd dev/compose
docker compose -f standalone.yaml up -d --build
```

## 🛡️ Security Notes

- Avoid using the `root` user in containers in production environments.
- Always use trusted GPG keys for signing collections.
- Ensure secrets are stored securely (e.g., key passwords).

Maintained by: `vnikoghosyan@gmail.com`


