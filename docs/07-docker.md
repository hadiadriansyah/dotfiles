# 07 — Docker Workflow

## Setup

- **Engine**: Docker CE running natively in WSL2 (no Docker Desktop)
- **Compose**: Docker Compose plugin (built-in)
- **Auto-start**: Enabled via systemd
- **User in docker group**: Yes (no sudo needed for `docker` commands)

### Why no Docker Desktop

- Docker Desktop adds 1.5-3 GB RAM overhead
- Native Docker Engine in WSL is faster
- Same functionality without GUI

GUI alternative: Portainer (running in container, see below).

## Hybrid philosophy

This system uses **hybrid** approach:

- **Native runtime** (Node, PHP, Python via mise) for active development
- **Docker** for databases, services, legacy projects in `/var/www/`

Why hybrid:
- LSP works smoothly with native runtime (Neovim hot reload, Intelephense, pyright)
- Database in Docker = isolated, easy to reset, multi-version support
- Legacy projects already containerized = leave them alone

## Installation (current method — deb822 `.sources` format)

Docker's official docs moved from the old single-line `.list` APT source format to the structured **deb822** `.sources` format. Use this going forward:

```bash
# Remove any conflicting old install first
sudo apt remove -y docker docker-engine docker.io containerd runc 2>/dev/null

# Add Docker's official GPG key
sudo apt update
sudo apt install -y ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add the repository (deb822 format)
sudo tee /etc/apt/sources.list.d/docker.sources <<EOF
Types: deb
URIs: https://download.docker.com/linux/ubuntu
Suites: $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}")
Components: stable
Architectures: $(dpkg --print-architecture)
Signed-By: /etc/apt/keyrings/docker.asc
EOF

sudo apt update
sudo apt install -y \
  docker-ce docker-ce-cli containerd.io \
  docker-buildx-plugin docker-compose-plugin

# Group + systemd
sudo usermod -aG docker $USER
newgrp docker
sudo systemctl enable docker
sudo systemctl start docker

# Test
docker --version
docker compose version
docker run --rm hello-world
```

> **Old format (deprecated but still functional)** used a single `.list` file with one `deb [...] URL suite component` line and `VERSION_CODENAME` directly instead of `${UBUNTU_CODENAME:-$VERSION_CODENAME}`. Both work at the apt level; the `.sources` format is what Docker's docs currently lead with, and the `UBUNTU_CODENAME` fallback is slightly more robust on Ubuntu derivatives where `VERSION_CODENAME` can occasionally point at the underlying Debian codename instead.

Docker's official APT repo has supported new Ubuntu LTS releases (e.g. 26.04) from day one in past cycles, so this should work cleanly on either 24.04 or 26.04 without needing to wait for repo updates.

## Active containers

8 containers run permanently for legacy projects in `/var/www/`. They auto-restart with WSL.

| Container | Image | Port | Purpose |
|-----------|-------|------|---------|
| `nginx` | nginx:alpine | 80 | Reverse proxy for `/var/www/` projects |
| `projects-apps` | (custom php-fpm) | 9000 | PHP-FPM for general projects |
| `stpkm-siakad` | (custom php-fpm) | 9000 | PHP-FPM for STPKM SIAKAD project |
| `mysql-db` | mysql:8.4.5 | 3307→3306 | MySQL 8 (modern projects) |
| `mysql5-db` | mysql:5.7 | 3308→3306 | MySQL 5.7 (legacy projects) |
| `phpmyadmin` | phpmyadmin/phpmyadmin | 8082 | GUI for MySQL 8 |
| `phpmyadmin5` | phpmyadmin/phpmyadmin | 8083 | GUI for MySQL 5.7 |
| `portainer` | portainer/portainer-ce:lts | 9000 | Docker GUI |

### Access URLs (from Windows browser)

- http://localhost — main projects via nginx
- http://localhost:8082 — phpMyAdmin (MySQL 8.4)
- http://localhost:8083 — phpMyAdmin (MySQL 5.7)
- http://localhost:9000 — Portainer

### MySQL connections (for DBeaver/TablePlus on Windows)

- MySQL 8.4: `localhost:3307`
- MySQL 5.7: `localhost:3308`

Credentials are in respective `docker-compose.yml` files in `/var/www/Apps/`.

## Daily Docker commands

### Container management

```bash
docker ps                                  # running containers
docker ps -a                               # all containers (incl. stopped)
docker ps --format 'table {{.Names}}\t{{.Status}}\t{{.Ports}}'

docker stop <container>
docker stop $(docker ps -q)
docker start <container>
docker start $(docker ps -aq)
docker restart <container>

docker rm <container>
docker rm -f <container>
```

### Logs and inspection

```bash
docker logs <container>
docker logs <container> --tail 50
docker logs <container> -f
docker logs <container> --since 10m

docker exec -it <container> bash
docker exec -it <container> sh

docker inspect <container>
docker stats
```

## Docker Compose for new projects

`~/projects/my-saas/docker-compose.yml`:

```yaml
services:
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: dev
      POSTGRES_PASSWORD: dev
      POSTGRES_DB: my_saas
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

  mailpit:
    image: axllent/mailpit:latest
    ports:
      - "1025:1025"
      - "8025:8025"

volumes:
  postgres_data:
```

### Compose commands

```bash
docker compose up -d
docker compose up -d postgres
docker compose down
docker compose down -v                     # DESTROYS DATA

docker compose logs
docker compose logs postgres
docker compose logs -f

docker compose ps
docker compose restart
docker compose pull

docker compose exec postgres psql -U dev
docker compose run postgres psql -U dev
```

## Image / Volume / Network management

```bash
docker images
docker pull mysql:8
docker rmi <image>
docker image prune
docker system prune -a
docker system df

docker volume ls
docker volume inspect <volume>
docker volume rm <volume>
docker volume prune

docker network ls
docker network inspect <network>
```

⚠️ Volumes hold persistent data (databases). Don't blindly run `volume rm` or `system prune --volumes`.

## Best practices

- **Don't put projects in `/mnt/c/`** — put them in `~/projects/`, WSL2 has 10-100x slower file IO on Windows mounts.
- **Use Alpine images** where possible (`postgres:16-alpine`, `redis:7-alpine`, `node:22-alpine`).
- **Stop containers when not working** on a given project: `docker compose down`. The `/var/www/` containers stay running because they're for active legacy projects.
- **Use `.dockerignore`**: `node_modules`, `.git`, `.env`, `*.log`, `vendor`, `.DS_Store`.
- **Pin image versions**: `postgres:16-alpine`, not `postgres:latest`.

## Troubleshooting

### Cannot connect to Docker daemon

```bash
sudo systemctl status docker
sudo systemctl start docker
sudo systemctl enable docker
```

### Permission denied (need sudo)

```bash
sudo usermod -aG docker $USER
# logout + login, or `newgrp docker`
```

### Container keeps crashing

```bash
docker logs <container> --tail 100
```

### Port already in use

```bash
sudo lsof -i :8082
sudo ss -tlnp | grep 8082
```

### Disk filling up

```bash
docker system df
docker system prune -a
docker volume prune
docker images --format '{{.Size}}\t{{.Repository}}:{{.Tag}}' | sort -h
```

### Container running but app not accessible

1. `docker ps` — is it actually running?
2. `docker port <container>` — port mapped correctly?
3. `docker exec <container> curl localhost:80` — service responding inside?

### Restart all `/var/www/` containers after WSL restart

```bash
docker start $(docker ps -aq)
```

---

## Changelog vs original version of this doc

- **Install method changed**: switched from the old single-line `.list` APT source format to the current deb822 `.sources` format (matches Docker's official docs as of the July 2026 rebuild). Both formats work; `.sources` is what's now recommended.
- Confirmed Docker's APT repo supports new Ubuntu LTS releases from day one, so no waiting/pinning needed when moving to a newer Ubuntu version.
