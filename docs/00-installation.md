# 00 — Installation Guide (From Zero to Complete Setup)

> Reproduce this entire WSL2 fullstack development environment from a fresh Ubuntu install.
> Estimated total time: **1.5–3 hours** (mostly waiting for compiles).
>
> **Last updated**: July 2026 — after rebuilding on a new laptop (Lenovo LOQ 15, Ryzen 7 7735HS, 16GB RAM). See changelog at the bottom for what changed vs the original version of this doc.

## Prerequisites

- Windows 10/11 with WSL2 enabled
- Ubuntu installed via `wsl --install Ubuntu` (or a specific version like `Ubuntu-26.04` if you want to pin it — see note below)
- Active internet connection
- Claude.ai Pro subscription (for Claude Code authentication, optional)
- Nerd Font installed in Windows (for terminal icons; e.g., JetBrainsMono Nerd Font)
- Windows Terminal configured to use that Nerd Font

Verify WSL version (in PowerShell):

```powershell
wsl --list --verbose
```

Should show your Ubuntu distro with `VERSION 2`.

> **Note on distro naming**: `wsl --install Ubuntu` installs whatever the current LTS is and *may* auto-upgrade to the next LTS via the Microsoft Store mechanism. `wsl --install Ubuntu-26.04` (or whichever version is current) pins to that specific release — it won't silently change later. For a fully reproducible setup, pinning is slightly safer, but in practice both resolve to the same thing at install time.

---

## Phase 1 — Foundation

### 1.1 Configure WSL resource limits

Edit `.wslconfig` from PowerShell:

```powershell
notepad $env:USERPROFILE\.wslconfig
```

Paste (adjust `memory`/`processors`/`swap` to your actual hardware — see table below):

```ini
[wsl2]
memory=8GB
processors=6
swap=4GB
localhostForwarding=true

[experimental]
autoMemoryReclaim=gradual
```

**Sizing guide** (rule of thumb: leave enough headroom for Windows + whatever you run alongside WSL):

| Physical RAM | Suggested `memory=` | Suggested `processors=` |
|---|---|---|
| 8 GB | 4GB | 4 |
| 16 GB | 8GB (10GB if WSL is your main workload and Windows-side apps are light) | 6 |
| 32 GB | 16-20GB | 8+ |

> **`autoMemoryReclaim=gradual`**: lets WSL release unused memory back to Windows gradually instead of holding onto it until `wsl --shutdown`. Belongs in `[experimental]`, **not** `[wsl2]` — putting it in the wrong section produces a silent "Unknown key" warning on every `wsl` launch (harmless, but annoying). Requires a reasonably recent WSL version; run `wsl --update` first if it's not recognized.

Save, close. Restart WSL:

```powershell
wsl --shutdown
```

Reopen WSL Ubuntu.

### 1.2 Verify resource limits applied

```bash
free -h && nproc
```

### 1.3 Update system

```bash
sudo apt update && sudo apt upgrade -y
```

### 1.4 Install build essentials

```bash
sudo apt install -y \
  build-essential curl wget git unzip \
  ca-certificates gnupg lsb-release \
  software-properties-common file
```

### 1.5 Enable systemd

Required for Docker auto-start.

```bash
sudo nano /etc/wsl.conf
```

Add:

```ini
[boot]
systemd=true
```

From PowerShell:

```powershell
wsl --shutdown
```

Reopen WSL. Verify:

```bash
ps -p 1 -o comm=
```

Expected: `systemd`

---

## Phase 2 — Shell (Zsh + Starship + Plugins)

### 2.1 Install Zsh

```bash
sudo apt install -y zsh
zsh --version
```

### 2.2 Install Starship prompt

```bash
curl -sS https://starship.rs/install.sh | sh
starship --version
```

### 2.3 Install Zsh plugins

```bash
mkdir -p ~/.zsh/plugins
git clone https://github.com/zsh-users/zsh-autosuggestions ~/.zsh/plugins/zsh-autosuggestions
git clone https://github.com/zsh-users/zsh-syntax-highlighting ~/.zsh/plugins/zsh-syntax-highlighting
git clone https://github.com/zsh-users/zsh-history-substring-search ~/.zsh/plugins/zsh-history-substring-search
ls ~/.zsh/plugins/
```

### 2.4 Create `.zshrc`

See `reference/zshrc-current.txt` for the full, current version (includes tool aliases, integrations, and the `work()` tmux function). Build it in one shot:

```bash
rm -f ~/.zshrc
cat > ~/.zshrc << 'EOF'
# ============================================================
# Zsh configuration — Hadi Adriansyah
# ============================================================

HISTFILE=~/.zsh_history
HISTSIZE=10000
SAVEHIST=10000
setopt SHARE_HISTORY HIST_IGNORE_DUPS HIST_IGNORE_SPACE

autoload -Uz compinit && compinit
zstyle ':completion:*' menu select
zstyle ':completion:*' matcher-list 'm:{a-zA-Z}={A-Za-z}'

setopt AUTO_CD INTERACTIVE_COMMENTS NO_BEEP NO_BANG_HIST

export VISUAL="nvim"
export EDITOR="$VISUAL"
export SUDO_EDITOR="nvim"

typeset -U path
path=(~/.local/bin ~/bin $path)

alias ..='cd ..'
alias ...='cd ../..'
alias grep='grep --color=auto'
alias fd='fdfind'
alias bat='batcat'
alias ls='eza --icons'
alias ll='eza -lah --icons --git'
alias la='eza -a --icons'
alias lt='eza --tree --level=2 --icons'
alias lg='lazygit'
alias tls='tmux ls'
alias tat='tmux attach -t'

source ~/.zsh/plugins/zsh-autosuggestions/zsh-autosuggestions.zsh
source ~/.zsh/plugins/zsh-syntax-highlighting/zsh-syntax-highlighting.zsh
source ~/.zsh/plugins/zsh-history-substring-search/zsh-history-substring-search.zsh

bindkey "^[[A" history-substring-search-up
bindkey "^[[B" history-substring-search-down
bindkey "^[OA" history-substring-search-up
bindkey "^[OB" history-substring-search-down

eval "$(~/.local/bin/mise activate zsh)"
eval "$(zoxide init zsh)"
[ -f /usr/share/doc/fzf/examples/key-bindings.zsh ] && source /usr/share/doc/fzf/examples/key-bindings.zsh
[ -f /usr/share/doc/fzf/examples/completion.zsh ] && source /usr/share/doc/fzf/examples/completion.zsh

work() {
  local name="${1:-work}"
  if tmux has-session -t "$name" 2>/dev/null; then
    tmux attach -t "$name"
  else
    tmux new-session -d -s "$name" -c "$PWD"
    tmux send-keys -t "$name" "nvim ." C-m
    tmux split-window -h -t "$name" -p 40 -c "$PWD"
    tmux send-keys -t "$name" "claude" C-m
    tmux split-window -v -t "$name" -p 35 -c "$PWD"
    tmux select-pane -t "$name":0.0
    tmux attach -t "$name"
  fi
}
tkill() { tmux kill-session -t "${1:-work}"; }

eval "$(starship init zsh)"
EOF
```

> **Note**: `mise`, `zoxide`, `fzf` referenced here aren't installed yet at this point — that's expected. `zoxide`/`fzf` lines are guarded and stay silent; the `mise activate` line will print a harmless "not found" until Phase 4 is done.

### 2.5 Set Zsh as default shell

```bash
chsh -s $(which zsh)
getent passwd $USER | cut -d: -f7
exec zsh
```

Expected: `/usr/bin/zsh`, and you should land on the Starship prompt (not the `zsh-newuser-install` menu — if you see that menu, it means `.zshrc` wasn't created yet).

---

## Phase 3 — Modern CLI Tools

### 3.1 Install tools available via apt

```bash
sudo apt install -y \
  ripgrep fd-find bat fzf tmux btop zoxide
```

### 3.2 Install eza (modern `ls`)

```bash
sudo mkdir -p /etc/apt/keyrings
wget -qO- https://raw.githubusercontent.com/eza-community/eza/main/deb.asc | sudo gpg --dearmor -o /etc/apt/keyrings/gierens.gpg
echo "deb [signed-by=/etc/apt/keyrings/gierens.gpg] http://deb.gierens.de stable main" | sudo tee /etc/apt/sources.list.d/gierens.list
sudo chmod 644 /etc/apt/keyrings/gierens.gpg /etc/apt/sources.list.d/gierens.list
sudo apt update && sudo apt install -y eza
eza --version
```

### 3.3 Install lazygit

```bash
LAZYGIT_VERSION=$(curl -s "https://api.github.com/repos/jesseduffield/lazygit/releases/latest" | grep -Po '"tag_name": "v\K[^"]*')
curl -Lo /tmp/lazygit.tar.gz "https://github.com/jesseduffield/lazygit/releases/download/v${LAZYGIT_VERSION}/lazygit_${LAZYGIT_VERSION}_Linux_x86_64.tar.gz"
tar xf /tmp/lazygit.tar.gz -C /tmp lazygit
sudo install /tmp/lazygit -D -t /usr/local/bin/
rm /tmp/lazygit /tmp/lazygit.tar.gz
lazygit --version
```

### 3.4 Install GitHub CLI (gh)

```bash
sudo mkdir -p -m 755 /etc/apt/keyrings
wget -qO- https://cli.github.com/packages/githubcli-archive-keyring.gpg | sudo tee /etc/apt/keyrings/githubcli-archive-keyring.gpg > /dev/null
sudo chmod go+r /etc/apt/keyrings/githubcli-archive-keyring.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/githubcli-archive-keyring.gpg] https://cli.github.com/packages stable main" | sudo tee /etc/apt/sources.list.d/github-cli.list > /dev/null
sudo apt update && sudo apt install -y gh
gh --version
```

---

## Phase 4 — mise + Programming Languages

### 4.1 Install mise

```bash
curl https://mise.run | sh
exec zsh
mise --version
mise doctor
```

> If `node`/`php`/`python` say "command not found" right after `mise use`, run `mise reshim` then `exec zsh` — shims sometimes need an explicit refresh.

### 4.2 Node.js + pnpm

Check what's current vs LTS before picking a version — Node has an even-numbered LTS cadence (new major goes Current in April, becomes Active LTS the following October):

```bash
mise ls-remote node 24
mise use --global node@24
mise use --global pnpm@latest
node --version
pnpm --version
```

> As of mid-2026: Node 24 = Active LTS (recommended), Node 26 = Current (not LTS until Oct 2026), Node 22 = Maintenance LTS.

### 4.3 Python 3.14 + uv

```bash
mise ls-remote python 3.14
mise use --global python@3.14
mise use --global uv@latest
python --version
uv --version
```

⚠️ Compiles from source (1–3 minutes).

> Python doesn't have an LTS concept like Node — every release (3.12/3.13/3.14) is equally "stable" once released; the difference is just age and third-party package readiness.

### 4.4 Install PHP build dependencies

```bash
sudo apt install -y \
  build-essential autoconf libtool bison re2c pkg-config \
  libxml2-dev libssl-dev libsqlite3-dev libbz2-dev \
  libcurl4-openssl-dev libgd-dev libjpeg-dev libpng-dev \
  libfreetype6-dev libwebp-dev libxpm-dev libonig-dev \
  libreadline-dev libtidy-dev libxslt1-dev libzip-dev \
  zlib1g-dev libargon2-dev libsodium-dev libldap2-dev libpq-dev \
  libicu-dev
```

> **`libicu-dev` added** — PHP 8.4's `intl` extension needs ICU headers at compile time. Without it, `mise install php@8.4` fails with an ICU-related `pkg-config`/configure error. This wasn't needed for PHP 8.3 builds in earlier versions of this doc.

### 4.5 Install PHP 8.4

⚠️ Compiles from source (3–7 minutes).

```bash
mise ls-remote php 8.4
mise use --global php@8.4
php --version
```

> **Watch for RC versions.** `mise use --global php@8.4` may resolve to the latest available build under that line, which can occasionally be a Release Candidate (e.g. `8.4.24RC1`) rather than a final stable release. Check the output of `php --version` — if it says `RC` anywhere, pin to the last non-RC version explicitly:
> ```bash
> mise uninstall php@8.4.24RC1
> mise use --global php@8.4.23   # use whatever the latest non-RC patch is
> ```

### 4.6 Install Composer

```bash
curl -sS https://getcomposer.org/installer | php
sudo mv composer.phar /usr/local/bin/composer
composer --version
```

### 4.7 Verify all runtimes

```bash
echo "Node:     $(node --version)"
echo "pnpm:     $(pnpm --version)"
echo "Python:   $(python --version)"
echo "uv:       $(uv --version)"
echo "PHP:      $(php --version | head -1)"
echo "Composer: $(composer --version | head -1)"
mise list
```

### 4.8 Install Go (optional, when needed)

```bash
mise use --global go@1.23
go install golang.org/x/tools/gopls@latest
```

---

## Phase 5 — Neovim + LazyVim

### 5.1 Install Neovim (AppImage extracted)

```bash
cd /tmp
curl -LO https://github.com/neovim/neovim/releases/latest/download/nvim-linux-x86_64.appimage
chmod u+x nvim-linux-x86_64.appimage
./nvim-linux-x86_64.appimage --appimage-extract
sudo rm -rf /opt/nvim 2>/dev/null
sudo mv squashfs-root /opt/nvim
sudo ln -sf /opt/nvim/AppRun /usr/local/bin/nvim
rm /tmp/nvim-linux-x86_64.appimage
cd ~
nvim --version | head -1
```

### 5.2 Install LazyVim starter

```bash
git clone https://github.com/LazyVim/starter ~/.config/nvim
rm -rf ~/.config/nvim/.git
ls ~/.config/nvim
```

> On a rebuild with a pre-existing `~/.config/nvim`, back it up first: `mv ~/.config/nvim ~/.config/nvim.backup-$(date +%Y%m%d-%H%M%S)`.

### 5.3 First launch — auto-bootstrap plugins

```bash
nvim
```

Wait for plugin install (2–5 min), then `:qa` and reopen.

### 5.4 Fix `:checkhealth` issues before installing LSPs

Run `:checkhealth` and address these commonly-missing pieces (all found during the 2026 rebuild — none of this was in earlier versions of this doc):

**`tree-sitter-cli` not found** (blocks some treesitter features):
```bash
npm install -g tree-sitter-cli
tree-sitter --version
```

**Node provider warning** (only matters if you use Node-based Neovim plugins):
```bash
npm install -g neovim
```

**Python provider warning** (only matters if you use Python-based Neovim plugins):
```bash
uv pip install --python "$(which python)" pynvim
```

**Clipboard**: recent WSL/WSLg builds already ship `win32yank` — check with `which win32yank.exe` before assuming you need to install it manually. If genuinely missing:
```bash
sudo apt install -y unzip
curl -sLo /tmp/win32yank.zip https://github.com/equalsraf/win32yank/releases/latest/download/win32yank-x64.zip
unzip -o /tmp/win32yank.zip -d /tmp
chmod +x /tmp/win32yank.exe
sudo mv /tmp/win32yank.exe /usr/local/bin/
rm /tmp/win32yank.zip
```

Ignorable warnings: luarocks/hererocks (only needed by rare plugins), `snacks.image` missing `magick`/`ghostty`/`gs`/`tectonic`/`mmdc` (optional image/PDF/LaTeX/Mermaid rendering, not needed for PHP/Node/Python work), Perl/Ruby providers (irrelevant if you don't use those languages).

### 5.5 Install LSP servers via Mason

Mason is lazy-loaded — open `:Mason` once first so the plugin loads, *then* run `:MasonInstall`, or the command won't be recognized yet:

```vim
:Mason
:q
:MasonInstall typescript-language-server intelephense pyright tailwindcss-language-server eslint-lsp lua-language-server vue-language-server prettier stylua php-cs-fixer
```

### 5.6 Enable LazyVim Extras

```vim
:LazyExtras
```

Press `x` on: `lang.typescript`, `lang.php`, `lang.python`, `lang.tailwind`, `lang.json`. `q` to close, then `:qa` and reopen.

---

## Phase 6 — Docker

### 6.1 Remove any existing Docker installation

```bash
sudo apt remove -y docker docker-engine docker.io containerd runc 2>/dev/null
```

### 6.2 Install Docker Engine (native, no Desktop)

Using the current deb822 (`.sources`) format, per Docker's current official docs:

```bash
sudo apt update
sudo apt install -y ca-certificates curl

sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

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
```

> **Changed from the old `.list` one-liner format.** Docker's official docs moved to the deb822 `.sources` format (structured key-value, replacing the old single-line `deb [...] URL suite component` syntax). Both work; `.sources` is what Docker's docs now lead with.

### 6.3 Add user to docker group

```bash
sudo usermod -aG docker $USER
newgrp docker
```

### 6.4 Enable Docker via systemd

```bash
sudo systemctl enable docker
sudo systemctl start docker
sudo systemctl status docker
```

### 6.5 Test Docker

```bash
docker --version
docker compose version
docker run --rm hello-world
```

---

## Phase 7 — Claude Code (Optional)

### 7.1 Install Claude Code CLI

```bash
npm install -g @anthropic-ai/claude-code
claude --version
which claude
```

### 7.2 Verify no API key environment

```bash
echo "${ANTHROPIC_API_KEY:-not set}"
grep -n ANTHROPIC ~/.zshrc ~/.profile 2>/dev/null
```

### 7.3 Authenticate

```bash
claude
```

Choose "Log in with Claude.ai".

### 7.4 tmux helper (already included in `.zshrc` from Phase 2.4)

Verify `work`, `tls`, `tat`, `tkill` all work:

```bash
cd ~/projects/some-project    # any folder
work
```

---

## Phase 8 — Final Configuration

### 8.1 Git identity

```bash
git config --global user.name "Hadi Adriansyah"
git config --global user.email "your-email@example.com"
git config --global init.defaultBranch main
git config --global pull.rebase false
```

### 8.2 GitHub CLI auth

```bash
gh auth login
```

### 8.3 SSH key (if using SSH for git)

```bash
ssh-keygen -t ed25519 -C "your-email@example.com"
gh ssh-key add ~/.ssh/id_ed25519.pub --title "WSL Ubuntu - <laptop name>"
```

### 8.4 Full verification

```bash
echo "=== Shell ===" && echo "Shell: $SHELL" && zsh --version
echo "=== Runtimes ===" && mise list
echo "=== CLI tools ===" && for tool in rg fdfind batcat eza fzf tmux btop zoxide lazygit gh starship; do command -v $tool >/dev/null 2>&1 && echo "$tool: OK" || echo "$tool: MISSING"; done
echo "=== Editor ===" && nvim --version | head -1
echo "=== Docker ===" && docker --version && docker compose version
echo "=== AI ===" && claude --version 2>/dev/null || echo "claude: not installed"
```

---

## Changelog vs original version of this doc

Discovered while rebuilding on a new laptop (Lenovo LOQ 15, 16GB RAM) in July 2026:

- **`.wslconfig`**: added `[experimental] autoMemoryReclaim=gradual` section (must be under `[experimental]`, not `[wsl2]`). Resource numbers now scaled to 16GB RAM instead of the original 8GB laptop.
- **Node**: 22 → 24 (Active LTS as of mid-2026; 22 is now Maintenance LTS, 26 is Current/not-yet-LTS).
- **Python**: 3.12 → 3.14 (no LTS concept for Python; picked latest stable for consistency with prior machine).
- **PHP**: 8.3 → 8.4. Added `libicu-dev` to build deps (required for `intl` extension, wasn't needed before). Watch out for `mise` resolving to an RC build — always check `php --version` output for `RC` and pin to the last stable patch if so.
- **Docker install**: switched from the old `.list` one-line APT source format to the current deb822 `.sources` format, matching Docker's current official docs.
- **Neovim `:checkhealth`**: new gaps not covered in the original doc — `tree-sitter-cli` (via npm), Node/Python Neovim providers (optional, only if you use JS/Python-based plugins), and a note that `win32yank` may already be present via WSLg on newer builds (check before manually installing).
- **Mason**: note added that `:Mason` must be opened once (loads the plugin) before `:MasonInstall` will be recognized as a command, on fresh LazyVim starter installs.

## Common gotchas during install

### "command not found" after install
Reload shell: `exec zsh`. If still missing, check PATH and verify install location.

### `zsh: event not found: !`
Already mitigated by `setopt NO_BANG_HIST` in `.zshrc`. Use single quotes for strings with `!`: `echo 'hello!'` not `echo "hello!"`.

### `zsh: no matches found: pattern*`
Wildcard matched nothing and Zsh errors strictly. Either suppress (`2>/dev/null`) or `setopt NULL_GLOB`.

### PHP compile fails with library not found
Re-run Phase 4.5 (build dependencies). Common missing: `libonig-dev`, `libzip-dev`, `libpq-dev`.

### Python compile slow
That's normal — first install takes 1–3 minutes. mise compiles from source for portability across Linux distros.

### Neovim shows missing icons (□, ?)
Nerd Font not active in Windows Terminal. Settings → Profiles → Ubuntu → Appearance → Font face → choose installed Nerd Font.

### Docker `permission denied`
Forgot to logout/login after `usermod -aG docker`. Workaround: `newgrp docker` activates group in current shell.

### LSP not active in Neovim file
Check `:LspInfo`. If empty: filetype not recognized, or LSP not installed via Mason. Install: `:MasonInstall <name>`.

### Mason install gopls fails
Go not installed. Skip until Phase 4.9 done.

---

## Time budget reference

If you do this in one sitting:

| Phase | Description | Time |
|-------|-------------|------|
| 1 | Foundation | 15 min |
| 2 | Shell setup | 10 min |
| 3 | CLI tools | 15 min |
| 4 | mise + runtimes (Python compile, PHP compile) | 15–25 min |
| 5 | Neovim + LazyVim | 15 min |
| 6 | Docker | 10 min |
| 7 | Claude Code (optional) | 5 min |
| 8 | Final config | 5 min |
| **Total** | | **90–120 min** |

Most time is waiting (Python compile, PHP compile, plugin downloads).

## After install — start using

1. **Test shell**: open new terminal, prompt should be Starship multi-line
2. **Test workflow**: `cd ~/projects/some-project && work` (if Claude Code installed)
3. **Read references**: see other docs in this folder for daily commands
4. **Push dotfiles to GitHub**: see [README.md](../README.md)

Done! 🎉
