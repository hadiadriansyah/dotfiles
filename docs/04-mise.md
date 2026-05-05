# 04 — mise (Multi-Language Version Manager)

> mise (pronounced "miz") is the central tool for managing all programming language versions on this system. It replaces the combination of nvm + pyenv + phpenv + rbenv + SDKMAN with a single tool.

## Status on this system

- **Binary**: `~/.local/bin/mise`
- **Activate hook**: in `~/.zshrc` via `eval "$(~/.local/bin/mise activate zsh)"`
- **Runtimes installed**: see `mise list` output below
- **Config storage**: `~/.local/share/mise/`
- **Shims**: `~/.local/share/mise/shims/`

## Concepts

### Tool

A "tool" in mise is a programming language or CLI utility. Examples: `node`, `python`, `php`, `go`, `pnpm`, `uv`.

### Version

Tools can have multiple versions installed in parallel. mise picks the active one based on configuration.

### Scope

- **Global**: applies everywhere (set with `--global`)
- **Local (project)**: applies in a project folder (set with `mise use` in that folder, creates `.mise.toml`)
- **Shell**: applies to current shell session only (`mise shell`)

Resolution order (highest priority first): shell > local > global.

## Daily commands

### Check what's active

```bash
mise current                        # show active versions for all tools
mise current node                   # show active Node version
mise list                           # all installed versions, all tools
mise list node                      # all installed Node versions
```

### Install new version

```bash
mise install node@22                # latest 22.x.x
mise install node@22.11.0           # exact version
mise install node@latest            # latest ever (may be pre-release!)
mise install node@lts               # latest LTS
mise install python@3.12            # latest 3.12.x
mise install php@8.3                # latest 8.3.x
mise install go@1.23                # latest 1.23.x
```

### Set as active default

```bash
mise use --global node@22           # global default
mise use node@20                    # local (current folder), creates .mise.toml
mise use node@18.20.0               # exact version
```

`mise use` also installs if not already installed.

### Search available versions

```bash
mise ls-remote node                 # all available Node versions
mise ls-remote node | tail -20      # 20 latest
mise ls-remote node 22              # only 22.x.x versions
mise ls-remote python 3.12          # only 3.12.x
mise ls-remote --all                # very long list
```

### Uninstall

```bash
mise uninstall node@18.20.0         # remove specific version
mise uninstall python@3.11          # remove all 3.11.x
```

### Update

```bash
mise outdated                       # show tools with available updates
mise upgrade                        # upgrade everything to latest
mise upgrade node                   # upgrade only Node
mise self-update                    # update mise itself
```

## Version specifiers

| Specifier | Resolves to | Use case |
|-----------|-------------|----------|
| `node@22` | latest patch in major 22 | Most common — auto-pick patch updates |
| `node@22.11` | latest patch in 22.11 | Pin to minor |
| `node@22.11.0` | exact 22.11.0 | Production parity, lockfile |
| `node@latest` | absolute latest | ⚠️ May be pre-release (alpha/beta) |
| `node@lts` | latest LTS | Node-specific (only even numbers) |
| `node@lts/jod` | LTS codename "Jod" (Node 22) | Pin to specific LTS line |

### When to use which

- **Daily dev**: `node@22` (auto-pick patch)
- **Project/team**: `node@22.11.0` in `.mise.toml` (everyone uses same exact version)
- **CI/CD**: exact version (matches production)
- **Personal exploration**: `node@latest` if you want bleeding edge

## Per-project configuration

### Auto-switch with `.mise.toml`

In a project folder, create `.mise.toml`:

```toml
[tools]
node = "22"
php = "8.3"
python = "3.12"
```

Now whenever you `cd` into this folder, mise auto-switches to these versions. `cd` out, it switches back to global defaults.

### Generate from current versions

```bash
cd ~/projects/my-laravel-app
mise use node@22 php@8.3            # creates .mise.toml automatically
cat .mise.toml
```

### Compatible with `.tool-versions` (asdf format)

mise also reads `.tool-versions` files (asdf-compatible):

```
node 22.11.0
php 8.3.14
python 3.12.7
```

Useful for projects that other team members use with asdf.

### Compatible with `.nvmrc`

```bash
echo "22" > .nvmrc
mise current node    # respects .nvmrc
```

## Tool-specific notes

### Node.js + pnpm

```bash
mise use --global node@22
mise use --global pnpm@latest
node --version
pnpm --version
```

Global npm packages installed via mise-managed Node:
```bash
npm install -g @anthropic-ai/claude-code   # installs to mise's Node bin
```

⚠️ When you upgrade Node version, global packages don't auto-migrate. Reinstall after upgrade.

### Python + uv

```bash
mise use --global python@3.12
mise use --global uv@latest
python --version
uv --version
```

Note: Python compiles from source on first install (1-3 minutes). Subsequent versions reuse cached source.

For Python project deps, prefer `uv` over pip:
```bash
uv venv                       # create .venv
uv pip install requests       # install package
uv pip freeze > requirements.txt
```

### PHP + Composer

```bash
mise use --global php@8.3
php --version
```

⚠️ PHP compiles from source. **Requires apt build dependencies** (already installed):
```
build-essential autoconf libtool bison re2c pkg-config
libxml2-dev libssl-dev libsqlite3-dev libbz2-dev
libcurl4-openssl-dev libgd-dev libjpeg-dev libpng-dev
libfreetype6-dev libwebp-dev libxpm-dev libonig-dev
libreadline-dev libtidy-dev libxslt1-dev libzip-dev
zlib1g-dev libargon2-dev libsodium-dev libldap2-dev libpq-dev
```

Composer is installed separately (not via mise) at `/usr/local/bin/composer`:
```bash
curl -sS https://getcomposer.org/installer | php
sudo mv composer.phar /usr/local/bin/composer
```

### Go (not installed currently)

If needed:
```bash
mise use --global go@1.23
go version
go install golang.org/x/tools/gopls@latest      # LSP for Neovim
```

## Real workflow examples

### Switch Node for a legacy project

```bash
cd ~/projects/old-app                # global is node@22
node --version                        # 22.x.x
mise use node@18                      # creates .mise.toml with node = "18"
node --version                        # 18.x.x
cd ~                                  # back to global
node --version                        # 22.x.x again
```

### New Laravel project

```bash
mkdir ~/projects/new-laravel
cd ~/projects/new-laravel
cat > .mise.toml << EOF
[tools]
php = "8.3"
node = "22"
EOF
mise install                          # install all tools listed in .mise.toml
composer create-project laravel/laravel .
```

### Try a new Node version safely

```bash
mise install node@24                  # install but not activate
mise shell node@24                    # activate in this shell only
node --version                        # 24.x.x in current shell only
exit                                  # close shell, back to default
```

### Cleanup unused versions

```bash
mise list node
# shows: 18.20.0, 20.10.0, 22.11.0
mise uninstall node@18.20.0
mise uninstall node@20.10.0
```

## Configuration file

Global config: `~/.config/mise/config.toml` (created on first `mise use --global`).

Edit:
```bash
nvim ~/.config/mise/config.toml
```

Example:
```toml
[tools]
node = "22"
pnpm = "latest"
python = "3.12"
uv = "latest"
php = "8.3"
```

After manual edit:
```bash
mise install                          # install all listed tools
```

## Troubleshooting

### `command not found: mise`

`.zshrc` not sourced or mise activate hook missing. Check:
```bash
grep mise ~/.zshrc
exec zsh
```

### `node: command not found` after install

Shims not in PATH. Run:
```bash
mise reshim
exec zsh
```

### Version doesn't switch in folder

Check `.mise.toml` syntax:
```bash
cat .mise.toml
mise current
```

### Install fails with compile error

Usually missing build dependencies. For PHP/Python:
```bash
sudo apt install build-essential libxml2-dev libssl-dev libsqlite3-dev libbz2-dev libcurl4-openssl-dev libffi-dev
```

### Permission errors on `~/.local/share/mise/`

Don't run mise commands with `sudo`. mise is user-level. If permissions got broken:
```bash
sudo chown -R $USER:$USER ~/.local/share/mise ~/.local/bin/mise
```

### Slow first install of Python/PHP

That's source compilation, not a bug. Takes 1-5 minutes depending on CPU. Subsequent installs of patches are fast (cached source).

## Comparison with old tools

| Old approach | Replaced by mise |
|--------------|------------------|
| nvm install 22 | mise install node@22 |
| nvm use 22 | mise use node@22 |
| nvm alias default 22 | mise use --global node@22 |
| pyenv install 3.12.0 | mise install python@3.12 |
| sdk install java 17.0 | mise install java@17 |
| asdf install nodejs 22.0.0 | mise install node@22 |
| Volta `volta install node@22` | mise install node@22 |

## Resources

- Docs: https://mise.jdx.dev/
- GitHub: https://github.com/jdx/mise
- Available tools: https://mise.jdx.dev/registry.html
