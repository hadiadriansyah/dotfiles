# 01 — System Overview

## Hardware

- **Laptop**: Lenovo IdeaPad (model 81YM)
- **CPU**: AMD Ryzen 5 4500U with Radeon Graphics (6 cores, ~2.4 GHz)
- **RAM**: 8 GB physical
- **OS**: Windows 11 Pro 64-bit (Build 26200)

## WSL configuration

WSL2 with Ubuntu 24.04 LTS (Noble Numbat).

### Resource limits (`%USERPROFILE%\.wslconfig`)

```ini
[wsl2]
memory=4GB
processors=4
swap=2GB
localhostForwarding=true
```

This limits WSL to 4GB RAM (out of 8GB), leaving the other 4GB for Windows + browser. Adjust if RAM upgrade is done.

### systemd enabled (`/etc/wsl.conf`)

```ini
[boot]
systemd=true
```

This is required for Docker to auto-start with WSL.

## Filesystem layout

| Location | Purpose |
|----------|---------|
| `/home/adha/` | Home directory (USE THIS for projects, not `/mnt/c/`) |
| `/var/www/` | Legacy projects directory (Docker-based, see [07-docker.md](07-docker.md)) |
| `/usr/local/bin/` | System-wide manually-installed binaries (nvim, starship, lazygit) |
| `/opt/nvim/` | Neovim installation (extracted AppImage) |
| `~/.local/bin/` | User binaries (mise, claude) |
| `~/.local/share/mise/` | mise-managed runtime installations |

⚠️ **Important**: Always work in `/home/adha/` filesystem, NOT `/mnt/c/...`. WSL2 has 10-100x slower file access on Windows-mounted paths.

## What's installed

### Shell environment
- **Zsh** 5.9 — primary shell
- **Starship** — cross-shell prompt
- **zsh-autosuggestions** — fish-like history suggestions
- **zsh-syntax-highlighting** — color valid/invalid commands

### Modern CLI tools
- **ripgrep** (`rg`) — fast text search
- **fd-find** (`fd`) — modern find replacement
- **bat** — cat with syntax highlighting
- **eza** — modern ls with icons + git status
- **fzf** — fuzzy finder (Ctrl+R, Ctrl+T)
- **zoxide** — smart cd
- **lazygit** — TUI for git
- **GitHub CLI** (`gh`) — GitHub from terminal
- **tmux** — terminal multiplexer
- **btop** — system monitor

### Version manager
- **mise** — multi-language version manager (replaces Volta/NVM/SDKMAN/pyenv)

### Programming languages (managed by mise)
- **Node.js** 22 (LTS) + **pnpm**
- **Python** 3.12 + **uv**
- **PHP** 8.3 + **Composer**

### Editor
- **Neovim** 0.12.x — primary editor
- **LazyVim** — Neovim distribution
- **LSP servers** (via Mason): typescript-language-server, intelephense (PHP), pyright (Python), tailwindcss-language-server, eslint-lsp, vue-language-server, lua-language-server
- **Formatters**: prettier, stylua, php-cs-fixer

### AI tooling
- **Claude Code** CLI (Anthropic)

### Container platform
- **Docker Engine** 29.4.2 + **Docker Compose** plugin (running natively in WSL, not Docker Desktop)

## What's NOT installed (intentionally)

- **Go** — not currently needed; install with `mise use --global go@1.23` when needed
- **Java/Kotlin/Ruby** — install via mise when needed
- **MySQL/PostgreSQL native** — all databases run in Docker
- **Apache/nginx native** — all web servers run in Docker
- **Docker Desktop** — replaced with native Docker Engine for memory efficiency

## Disk usage reference

After full clean install (~May 2026):
- Root filesystem: ~19 GB / 1 TB
- WSL VHD: managed by Windows, dynamically grows
- Major consumers: Docker images (~3-5 GB), Neovim plugins (~200-500 MB), mise runtimes (~1 GB)

Free space monitoring:
```bash
df -h ~
docker system df
```

## Network

- **WSL2 NAT**: Auto port forwarding to localhost (configured in `.wslconfig`)
- **Cloudflare WARP**: Active on Windows side (in PATH)
- **DNS**: WSL2 default

Common ports used by Docker projects:
- `80` — nginx (web)
- `3307` — MySQL 8.4.5
- `3308` — MySQL 5.7
- `8082` — phpMyAdmin (MySQL 8)
- `8083` — phpMyAdmin (MySQL 5.7)
- `9000` — Portainer (Docker GUI)

See [07-docker.md](07-docker.md) for full container details.
