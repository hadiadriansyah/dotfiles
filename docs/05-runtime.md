# 05 — Programming Language Runtimes

All runtimes are managed by [mise](04-mise.md). This file covers per-language workflow, common project setup, and tooling.

## Currently installed

| Language | Version | Package manager | Status |
|----------|---------|-----------------|--------|
| Node.js | 22 (LTS) | pnpm | ✅ |
| Python | 3.12 | uv | ✅ |
| PHP | 8.3 | Composer | ✅ |
| Go | — | (built-in) | ❌ Not installed |

## Node.js + pnpm

### Versions

```bash
node --version         # v22.x.x
pnpm --version         # 9.x.x or 10.x.x
```

### Why pnpm over npm

- **Disk efficient**: hardlinks shared deps across projects (saves GBs)
- **Faster installs**: ~2-3x faster than npm
- **Strict**: prevents phantom dependencies (better than npm/yarn)

### Common commands

```bash
# Project setup
pnpm init                              # create package.json
pnpm add lodash                        # install dep
pnpm add -D typescript                 # devDep
pnpm add -g vercel                     # global (not recommended unless needed)

# Workflow
pnpm install                           # install from package.json
pnpm dev                               # run "dev" script
pnpm build                             # run "build" script
pnpm test                              # run "test" script
pnpm exec eslint src/                  # run binary from node_modules
pnpm dlx create-next-app@latest        # run package without installing (like npx)

# Workspace (monorepo)
pnpm -r install                        # install all workspaces
pnpm -F mypackage dev                  # run dev in specific workspace

# Maintenance
pnpm outdated                          # show outdated deps
pnpm update                            # update deps within semver range
pnpm update --latest                   # update to latest (may break)
pnpm audit                             # security audit
pnpm store prune                       # cleanup unused store entries
```

### Quick start: Next.js project

```bash
cd ~/projects
pnpm dlx create-next-app@latest my-app
cd my-app
mise use node@22                       # pin to Node 22
pnpm dev                               # http://localhost:3000
```

### Quick start: Vite + React

```bash
cd ~/projects
pnpm create vite@latest my-app -- --template react-ts
cd my-app
pnpm install
pnpm dev
```

### Global tools

Some packages are useful globally. Install via `pnpm add -g`:

```bash
pnpm add -g @anthropic-ai/claude-code   # already installed
pnpm add -g serve                       # static file server
pnpm add -g typescript                  # global tsc
```

⚠️ **Caveat with mise + global packages**: When you upgrade Node, global packages don't migrate. Reinstall after `mise upgrade node`.

## Python + uv

### Versions

```bash
python --version       # Python 3.12.x
uv --version           # uv 0.x.x
```

### Why uv over pip + venv + pip-tools

- **10-100x faster** (Rust)
- Replaces pip + venv + pip-tools + pyenv combo
- Lockfile by default
- Compatible with `requirements.txt`, `pyproject.toml`

### Common commands

```bash
# Initialize project
uv init my-project                     # creates pyproject.toml + venv
cd my-project

# Or in existing folder
cd existing-project
uv venv                                # create .venv

# Install packages
uv pip install requests                # one-shot install
uv add requests                        # add to pyproject.toml
uv add --dev pytest                    # devDep

# Run scripts (auto-uses .venv)
uv run python script.py
uv run pytest

# Sync deps from lockfile
uv sync                                # like pnpm install

# Update deps
uv lock --upgrade                      # like pnpm update --latest
```

### Without uv (traditional)

If you must use plain pip:

```bash
python -m venv .venv
source .venv/bin/activate
pip install requests
deactivate
```

uv is just better. Use it.

### Quick start: FastAPI project

```bash
cd ~/projects
uv init my-api
cd my-api
uv add fastapi uvicorn
cat > main.py << EOF
from fastapi import FastAPI
app = FastAPI()
@app.get("/")
def root(): return {"hello": "world"}
EOF
uv run uvicorn main:app --reload
```

### Quick start: Data analysis script

```bash
mkdir -p ~/projects/analysis && cd ~/projects/analysis
uv init
uv add pandas matplotlib jupyter
uv run jupyter lab
```

## PHP + Composer

### Versions

```bash
php --version          # PHP 8.3.x
composer --version     # 2.x.x
```

### Composer commands

```bash
# Project setup
composer create-project laravel/laravel my-app    # create from template
composer init                                      # interactive composer.json
composer install                                   # install from composer.json
composer update                                    # update deps within constraints

# Add packages
composer require laravel/sanctum                   # add dep
composer require --dev phpstan/phpstan            # devDep
composer remove vendor/package                    # remove dep

# Run scripts
composer run-script test                          # run "test" script
composer dump-autoload                            # rebuild autoloader

# Global packages
composer global require laravel/installer         # global laravel command
composer global update
```

### Add Composer global bin to PATH

If you install global packages, add to `~/.zshrc`:

```bash
export PATH="$HOME/.config/composer/vendor/bin:$PATH"
```

(Already covered by `~/.local/bin` in PATH; check with `which laravel` after global install.)

### Quick start: Laravel project (native)

```bash
cd ~/projects
composer create-project laravel/laravel my-laravel-app
cd my-laravel-app
mise use php@8.3 node@22                # pin versions
cp .env.example .env
php artisan key:generate
php artisan migrate
php artisan serve                       # http://localhost:8000
```

For database, run a Postgres/MySQL container alongside (see [07-docker.md](07-docker.md)).

### Quick start: Standalone PHP script

```php
<?php
// scripts/cleanup.php
$files = glob('/tmp/*.log');
foreach ($files as $f) {
    if (filemtime($f) < strtotime('-7 days')) unlink($f);
}
```

```bash
php scripts/cleanup.php
```

## Go (not installed)

Install when needed:

```bash
mise use --global go@1.23
go version
```

Setup workspace:
```bash
mkdir -p ~/projects/go && cd ~/projects/go
go mod init github.com/hadiadriansyah/myapp
```

Common commands:
```bash
go run main.go              # run without build
go build                    # compile to binary
go test ./...               # run tests
go install ./...            # install binary to $GOPATH/bin

# Tools (gopls for Neovim LSP, etc)
go install golang.org/x/tools/gopls@latest
go install github.com/air-verse/air@latest          # hot reload
go install honnef.co/go/tools/cmd/staticcheck@latest # linter
```

After installing Go via mise, add tool path to PATH:
```bash
export PATH="$HOME/go/bin:$PATH"
```

## Tooling per language

### Linting + formatting

| Language | Linter | Formatter |
|----------|--------|-----------|
| Node/TS | ESLint | Prettier |
| Python | Ruff (built into uv ecosystem) | Ruff format / Black |
| PHP | PHPStan | PHP-CS-Fixer |
| Go | staticcheck | gofumpt |

All available as Neovim LSP/formatters via Mason. See [06-neovim.md](06-neovim.md).

### Testing

| Language | Common test runner |
|----------|--------------------|
| Node/TS | Vitest, Jest |
| Python | pytest |
| PHP | Pest, PHPUnit |
| Go | built-in `go test` |

## Multi-language project example

A typical fullstack project might have:

```
my-saas/
├── backend/          # Laravel (PHP 8.3)
│   ├── composer.json
│   └── .mise.toml    # php = "8.3"
├── frontend/         # Next.js (Node 22)
│   ├── package.json
│   └── .mise.toml    # node = "22", pnpm = "latest"
├── scripts/          # Python automation
│   ├── pyproject.toml
│   └── .mise.toml    # python = "3.12", uv = "latest"
└── docker-compose.yml  # postgres + redis
```

mise auto-switches versions when you `cd` into each subfolder.

## Performance tips

### Hot reload setup

| Language | Hot reload tool |
|----------|-----------------|
| Node | `pnpm dev` (most frameworks have it) |
| Python (FastAPI) | `uvicorn --reload` |
| Python (Flask) | `flask --debug run` |
| PHP (Laravel) | `php artisan serve` (built-in) + `npm run dev` (Vite) |
| Go | `air` (install via `go install`) |

### Fast deps install

```bash
# Node
pnpm install --prefer-offline          # use cache aggressively

# Python
uv sync                                # always fast

# PHP
composer install --prefer-dist --no-progress
```
