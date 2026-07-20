# 04 — mise (Multi-Language Version Manager)

> mise (pronounced "miz") is the central tool for managing all programming language versions on this system. It replaces the combination of nvm + pyenv + phpenv + rbenv + SDKMAN with a single tool.
>
> **Last updated**: July 2026 (after rebuild on new laptop). Default global versions bumped — see "Current global versions" below and the changelog at the bottom.

## Status on this system

- **Binary**: `~/.local/bin/mise`
- **Activate hook**: in `~/.zshrc` via `eval "$(~/.local/bin/mise activate zsh)"`
- **Config storage**: `~/.local/share/mise/`
- **Shims**: `~/.local/share/mise/shims/`

## Current global versions

| Tool | Version | Notes |
|---|---|---|
| Node | 24.x (Active LTS) | was 22 previously; 22 is now Maintenance LTS |
| pnpm | latest | currently 11.x |
| Python | 3.14.x | was 3.12 previously |
| uv | latest | |
| PHP | 8.4.x | was 8.3 previously; **must be a non-RC patch**, see warning below |

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

### Search available versions (do this BEFORE `use --global`)

```bash
mise ls-remote node                 # all available Node versions
mise ls-remote node 24              # only 24.x.x versions
mise ls-remote php 8.4              # only 8.4.x versions — check for RC tags!
mise ls-remote python 3.14          # only 3.14.x
```

⚠️ **Always eyeball the list for `RC`, `beta`, `alpha` tags** before running `mise use --global`. `mise use --global <tool>@<major>` resolves to the *latest available build under that line*, which can occasionally be a pre-release rather than a final stable release (this happened with PHP 8.4 during the July 2026 rebuild — it resolved to `8.4.24RC1`).

### Install new version

```bash
mise install node@24                # latest 24.x.x
mise install node@24.18.0           # exact version
mise install python@3.14            # latest 3.14.x
mise install php@8.4                # latest 8.4.x — verify it's not an RC (see above)
```

### Set as active default

```bash
mise use --global node@24           # global default
mise use node@20                    # local (current folder), creates .mise.toml
```

`mise use` also installs if not already installed.

### If you land on an RC/pre-release by accident

```bash
mise ls-remote php 8.4               # find the latest NON-rc version, e.g. 8.4.23
mise uninstall php@8.4.24RC1
mise use --global php@8.4.23
php --version                        # confirm no "RC" in the output
```

### Uninstall

```bash
mise uninstall node@18.20.0
mise uninstall python@3.11
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
| `node@24` | latest patch in major 24 | Most common — auto-pick patch updates |
| `node@24.18` | latest patch in 24.18 | Pin to minor |
| `node@24.18.0` | exact 24.18.0 | Production parity, lockfile |
| `node@latest` | absolute latest | ⚠️ May be Current/pre-LTS, not necessarily LTS |
| `node@lts` | latest LTS | Node-specific (only even majors, and only once promoted to LTS) |

### Node LTS cadence, for reference

Node promotes a new even-numbered major to "Current" every April, then to "Active LTS" the following October. So at any given time there's a ~6 month window where the newest even major (e.g. Node 26 as of mid-2026) is out but not yet LTS — the previous even major (Node 24) is the safer default for that window.

### When to use which

- **Daily dev**: `node@24` (current Active LTS at time of writing)
- **Project/team**: exact version in `.mise.toml` (everyone uses same build)
- **CI/CD**: exact version (matches production)
- **Personal exploration**: `node@latest` if you want the bleeding edge (accept it might not be LTS yet)

## Per-project configuration

### Auto-switch with `.mise.toml`

```toml
[tools]
node = "24"
php = "8.4"
python = "3.14"
```

### Compatible with `.tool-versions` (asdf format) and `.nvmrc`

Same as before — mise reads both formats transparently.

## Tool-specific notes

### Node.js + pnpm

```bash
mise use --global node@24
mise use --global pnpm@latest
```

Global npm packages installed via mise-managed Node:
```bash
npm install -g @anthropic-ai/claude-code
npm install -g tree-sitter-cli   # needed for Neovim's nvim-treesitter health check
npm install -g neovim            # only if you use Node-based Neovim plugins
```

⚠️ When you upgrade Node version, global packages don't auto-migrate. Reinstall after upgrade.

### Python + uv

```bash
mise use --global python@3.14
mise use --global uv@latest
```

Note: Python compiles from source on first install (1–3 minutes).

```bash
uv venv                       # create .venv
uv pip install requests       # install package
uv pip install --python "$(which python)" pynvim   # only if you use Python-based Neovim plugins
```

### PHP + Composer

```bash
mise use --global php@8.4
```

⚠️ PHP compiles from source. **Requires apt build dependencies**, including `libicu-dev` (newly required for the `intl` extension — missing this causes an ICU-related configure/pkg-config failure during `mise install php@8.4`):

```
build-essential autoconf libtool bison re2c pkg-config
libxml2-dev libssl-dev libsqlite3-dev libbz2-dev
libcurl4-openssl-dev libgd-dev libjpeg-dev libpng-dev
libfreetype6-dev libwebp-dev libxpm-dev libonig-dev
libreadline-dev libtidy-dev libxslt1-dev libzip-dev
zlib1g-dev libargon2-dev libsodium-dev libldap2-dev libpq-dev
libicu-dev
```

Composer is installed separately (not via mise):
```bash
curl -sS https://getcomposer.org/installer | php
sudo mv composer.phar /usr/local/bin/composer
```

### Go (not installed currently)

```bash
mise use --global go@1.23
go install golang.org/x/tools/gopls@latest
```

## Troubleshooting

### `command not found: mise`

```bash
grep mise ~/.zshrc
exec zsh
```

### `node: command not found` after install

```bash
mise reshim
exec zsh
```

### PHP install fails with an ICU / `intl` related configure error

```bash
sudo apt install -y libicu-dev
mise install php@8.4
```

### PHP install succeeds but `php --version` shows `RC`

You've landed on a pre-release build. Re-check available versions and pin to the last stable patch:

```bash
mise ls-remote php 8.4
mise uninstall php@8.4.XXRCX
mise use --global php@8.4.XX    # last non-RC patch
```

### Install fails with other compile errors

```bash
sudo apt install build-essential libxml2-dev libssl-dev libsqlite3-dev libbz2-dev libcurl4-openssl-dev libffi-dev
```

### Permission errors on `~/.local/share/mise/`

Don't run mise commands with `sudo`. If permissions got broken:
```bash
sudo chown -R $USER:$USER ~/.local/share/mise ~/.local/bin/mise
```

## Comparison with old tools

| Old approach | Replaced by mise |
|--------------|------------------|
| nvm install 22 | mise install node@24 |
| nvm use 22 | mise use node@24 |
| pyenv install 3.12.0 | mise install python@3.14 |
| asdf install nodejs 22.0.0 | mise install node@24 |

## Resources

- Docs: https://mise.jdx.dev/
- GitHub: https://github.com/jdx/mise
- Available tools: https://mise.jdx.dev/registry.html

---

## Changelog vs original version of this doc

- Global defaults bumped: Node 22 → 24, Python 3.12 → 3.14, PHP 8.3 → 8.4.
- Added `libicu-dev` to PHP build dependencies (required for `intl` extension on PHP 8.4).
- Added explicit warning + recovery steps for mise resolving to an RC/pre-release build (happened with PHP 8.4.24RC1 during the July 2026 rebuild).
- Added `mise ls-remote` as a required step *before* `mise use --global`, specifically to eyeball for RC/beta tags.
- Added Node LTS cadence explanation (Current in April → Active LTS the following October).
- Added `tree-sitter-cli` and `pynvim`/`neovim` npm/pip installs under the relevant tool sections (needed to clear Neovim `:checkhealth` warnings, not previously documented here).
