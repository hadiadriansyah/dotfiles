# 07 — Docker Workflow

## Setup

- **Engine**: Docker 29.4.2 (CE) running natively in WSL2 (no Docker Desktop)
- **Compose**: Docker Compose plugin (built-in)
- **Auto-start**: Enabled via systemd
- **User in docker group**: Yes (no sudo needed for `docker` commands)

### Why no Docker Desktop

- Docker Desktop adds 1.5-3 GB RAM overhead
- 8 GB total RAM is tight, every GB matters
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

docker stop <container>                    # stop one
docker stop $(docker ps -q)                # stop all running
docker start <container>                   # start one
docker start $(docker ps -aq)              # start all stopped
docker restart <container>                 # restart

docker rm <container>                      # remove (must be stopped first)
docker rm -f <container>                   # force remove (stops + removes)
```

### Logs and inspection

```bash
docker logs <container>                    # full logs
docker logs <container> --tail 50          # last 50 lines
docker logs <container> -f                 # follow (live tail)
docker logs <container> --since 10m        # last 10 min

docker exec -it <container> bash           # shell into container
docker exec -it <container> sh             # if no bash (alpine)

docker inspect <container>                 # full metadata (JSON)
docker stats                               # live resource usage
```

### Specific examples for our containers

```bash
# Tail nginx access logs
docker logs nginx -f

# Access MySQL 8 shell
docker exec -it mysql-db mysql -u root -p

# Access PHP-FPM container shell
docker exec -it projects-apps bash

# Run artisan command in PHP container
docker exec projects-apps php artisan migrate

# Restart all stopped containers
docker start $(docker ps -aq)
```

## Docker Compose for new projects

When starting a new project (NOT in `/var/www/`), use docker-compose for databases.

### Example: Postgres + Redis for a Laravel/Next.js project

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
      - "1025:1025"   # SMTP
      - "8025:8025"   # Web UI

volumes:
  postgres_data:
```

### Compose commands

```bash
docker compose up -d                       # start all (detached)
docker compose up -d postgres              # start specific service
docker compose down                        # stop all
docker compose down -v                     # stop + delete volumes (DESTROYS DATA)

docker compose logs                        # logs from all
docker compose logs postgres               # specific service
docker compose logs -f                     # follow

docker compose ps                          # status of services
docker compose restart                     # restart all
docker compose pull                        # update images

docker compose exec postgres psql -U dev   # exec into running service
docker compose run postgres psql -U dev    # run one-off command
```

### Workflow

```bash
cd ~/projects/my-saas
docker compose up -d                       # start db + redis
pnpm dev                                   # native Next.js, hot reload
# ... work ...
docker compose down                        # done for the day
```

## Image management

```bash
docker images                              # list local images
docker pull mysql:8                        # download specific image
docker rmi <image>                         # remove image
docker image prune                         # remove unused images
docker system prune -a                     # NUCLEAR — remove everything unused
docker system df                           # disk usage breakdown
```

## Volume management

```bash
docker volume ls                           # list volumes
docker volume inspect <volume>             # details
docker volume rm <volume>                  # delete (CONTAINER MUST BE STOPPED)
docker volume prune                        # remove unused volumes
```

⚠️ **Volumes hold persistent data** (databases). Don't blindly run `volume rm` or `system prune --volumes`.

## Network management

```bash
docker network ls                          # list networks
docker network inspect <network>           # inspect
```

## Troubleshooting

### Cannot connect to Docker daemon

```bash
sudo systemctl status docker
sudo systemctl start docker
sudo systemctl enable docker        # auto-start on WSL boot
```

### Permission denied (need sudo)

User not in docker group:
```bash
sudo usermod -aG docker $USER
# Then logout + login (or `newgrp docker`)
```

### Container keeps crashing

Check logs:
```bash
docker logs <container> --tail 100
```

Common causes:
- Port conflict (another service uses same port)
- Missing env variable
- Volume permission issue
- Out of memory

### Port already in use

```bash
sudo lsof -i :8082
# Or
sudo ss -tlnp | grep 8082
```

Kill the conflicting process or change the Docker port mapping.

### Disk filling up

```bash
docker system df                           # see breakdown
docker system prune -a                     # cleanup unused
docker volume prune                        # cleanup unused volumes
```

If a specific image is huge:
```bash
docker images --format '{{.Size}}\t{{.Repository}}:{{.Tag}}' | sort -h
docker rmi <huge-image>
```

### Container running but app not accessible

1. Container actually running? `docker ps`
2. Port mapped correctly? `docker port <container>`
3. Service inside container responding? `docker exec <container> curl localhost:80`
4. Firewall? Less common in WSL.

### Restart all `/var/www/` containers (after WSL restart)

They should auto-start (`restart: always` policy). If not:

```bash
docker start $(docker ps -aq)
```

## Best practices

### Don't put projects in `/mnt/c/`

If you start a Docker project, put it in `~/projects/`, not `/mnt/c/Users/...`. WSL2 has 10-100x slower file IO on Windows mounts. Docker volume bind mounts will be slow.

### Use Alpine images when possible

```yaml
image: postgres:16-alpine        # ~80 MB instead of 400 MB
image: redis:7-alpine            # ~30 MB
image: node:22-alpine            # ~50 MB instead of 350 MB
```

### Stop containers when not working

For 8 GB RAM laptop, stopping containers when not coding saves 200-500 MB:

```bash
cd ~/projects/some-project
docker compose down
```

The `/var/www/` containers stay running because they're for active legacy projects.

### Use `.dockerignore`

When building images:

```
node_modules
.git
.env
*.log
vendor
.DS_Store
```

Speeds up build, reduces image size, prevents secrets leakage.

### Pin image versions

Bad: `image: postgres:latest` — breaks when `latest` changes

Good: `image: postgres:16-alpine` — predictable, reproducible

## Reference: docker-compose patterns

### PHP project with Composer

```yaml
services:
  php:
    image: php:8.3-fpm-alpine
    volumes:
      - .:/var/www/html
    ports:
      - "9000:9000"

  nginx:
    image: nginx:alpine
    ports:
      - "8080:80"
    volumes:
      - .:/var/www/html
      - ./docker/nginx.conf:/etc/nginx/conf.d/default.conf
    depends_on:
      - php
```

### Multi-environment with profiles

```yaml
services:
  db:
    image: postgres:16-alpine
    profiles: [default]

  redis:
    image: redis:7-alpine
    profiles: [default, full]

  elasticsearch:
    image: elasticsearch:8.11
    profiles: [full]
```

```bash
docker compose --profile default up -d        # db + redis
docker compose --profile full up -d           # everything
```

### Health checks

```yaml
services:
  db:
    image: postgres:16-alpine
    healthcheck:
      test: ["CMD", "pg_isready", "-U", "postgres"]
      interval: 10s
      timeout: 5s
      retries: 5

  app:
    image: my-app
    depends_on:
      db:
        condition: service_healthy
```
