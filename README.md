# Dotfiles — WSL2 Fullstack Development Setup

> **Author**: Hadi Adriansyah
> **System**: Ubuntu 24.04 LTS on WSL2 (Windows 11)
> **Last updated**: May 2026

This repository documents my fullstack development environment built on WSL2. It serves as both a reference for daily work and a guide for rebuilding the setup on a new machine.

## Table of contents

0. **[Installation guide](docs/00-installation.md)** — Reproduce setup from fresh Ubuntu (start here for new machine)
1. [System overview](docs/01-overview.md) — Hardware, WSL config, what's installed
2. [Shell](docs/02-shell.md) — Zsh, Starship, plugins
3. [CLI tools](docs/03-tools.md) — Modern CLI replacements (rg, fd, bat, eza, fzf, etc)
4. [mise](docs/04-mise.md) — Multi-language version manager (full reference)
5. [Runtime](docs/05-runtime.md) — Node, Python, PHP — installed languages
6. [Neovim](docs/06-neovim.md) — LazyVim, LSP, keymaps
7. [Docker](docs/07-docker.md) — Container workflow for `/var/www/` projects
8. [Claude Code](docs/08-claude-code.md) — AI coding assistant + tmux workflow
9. [Troubleshooting](docs/09-troubleshoot.md) — Common issues and solutions

## Quick start (rebuild on new machine)

If I ever need to rebuild this setup on a new WSL2 instance:

```bash
# 1. Install WSL2 + Ubuntu 24.04 from Windows
# 2. Clone this repo
cd ~
git clone https://github.com/hadiadriansyah/dotfiles.git
cd dotfiles

# 3. Follow installation guide step-by-step
less docs/00-installation.md
# or
nvim docs/00-installation.md
```

The installation guide is the single source of truth for reproducing the entire setup. Estimated 90–120 minutes (most time is waiting for Python/PHP compiles).

## Daily workflow reference

```bash
# Start coding session (Neovim + Claude Code in tmux)
cd /var/www/Apps/my-project
work

# Manage runtime versions per project
mise use node@20          # set Node 20 for current project
mise current              # check active versions

# Git workflow with lazygit
lg                        # opens lazygit TUI

# Search code in project
rg "function login"       # ripgrep — fast text search
fd config.php             # find files by name

# Modern file listing
ll                        # eza with icons + git status
lt                        # tree view, 2 levels

# Smart cd
z apps                    # zoxide — jump to frequent folders

# Docker management
docker compose up -d      # start services for current project
docker compose down       # stop them
```

## Repository structure

```
~/dotfiles/
├── README.md                  # This file
├── .gitignore
├── docs/                      # Detailed documentation
│   ├── 00-installation.md     # FROM-ZERO INSTALL GUIDE (start here)
│   ├── 01-overview.md
│   ├── 02-shell.md
│   ├── 03-tools.md
│   ├── 04-mise.md
│   ├── 05-runtime.md
│   ├── 06-neovim.md
│   ├── 07-docker.md
│   ├── 08-claude-code.md
│   └── 09-troubleshoot.md
└── reference/                 # Snapshots of current state
    ├── zshrc-current.txt
    ├── installed-packages.txt
    └── docker-compose-info.txt
```

## Philosophy

- **Hybrid approach**: Native runtime (Node, PHP, Python) on host; databases and services in Docker
- **Keyboard-first**: Neovim, tmux, terminal-driven workflow
- **Modern tools**: Latest stable versions, prefer Rust-based CLI tools (ripgrep, fd, eza, mise)
- **Documented**: Everything I install is documented here, so I never forget what I have

## License

Personal dotfiles — feel free to copy ideas, but configuration is tailored to my workflow.
