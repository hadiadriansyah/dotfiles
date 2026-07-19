# 00 — Installation Guide (From Zero to Complete Setup)

> Reproduce this entire WSL2 fullstack development environment from a fresh Ubuntu 24.04 install.
> Estimated total time: **1.5–3 hours** (mostly waiting for compiles).

## Prerequisites

- Windows 10/11 with WSL2 enabled
- Ubuntu 24.04 LTS installed via Microsoft Store or `wsl --install -d Ubuntu`
- Active internet connection
- Claude.ai Pro subscription (for Claude Code authentication, optional)
- Nerd Font installed in Windows (for terminal icons; e.g., JetBrainsMono Nerd Font)
- Windows Terminal configured to use that Nerd Font

Verify WSL version (in PowerShell):

```powershell
wsl --list --verbose
```

Should show `Ubuntu` with `VERSION 2`.

---

## Phase 1 — Foundation

### 1.1 Configure WSL resource limits

Edit `.wslconfig` from PowerShell:

```powershell
notepad $env:USERPROFILE\.wslconfig
```

Paste:

```ini
[wsl2]
memory=4GB
processors=4
swap=2GB
localhostForwarding=true
```

Save, close. Restart WSL:

```powershell
wsl --shutdown
```

Reopen WSL Ubuntu.

**Why**: Without limit, WSL can consume up to ~80% of physical RAM, starving Windows. 4 GB allocation leaves room for browser + editor on 8 GB systems. Adjust to 8 GB if you upgrade to 16 GB physical RAM.

### 1.2 Verify resource limits applied

```bash
free -h && nproc
```

Expected:
- `Mem total`: ~3.7 GB (out of 4 GB allocated, with kernel overhead)
- `Swap total`: 2.0 GB
- `nproc`: 4

### 1.3 Update system

```bash
sudo apt update && sudo apt upgrade -y
```

Estimated: 5–15 minutes on first run.

### 1.4 Install build essentials

```bash
sudo apt install -y \
  build-essential curl wget git unzip \
  ca-certificates gnupg lsb-release \
  software-properties-common file
```

**Why**: `build-essential` provides gcc/g++/make needed for compiling Python, PHP, native Node modules. Other packages support adding third-party APT repositories.

### 1.5 Enable systemd

Required for Docker auto-start.

```bash
sudo nano /etc/wsl.conf
```

Add (or merge with existing content):

```ini
[boot]
systemd=true
```

Save (`Ctrl+O`, `Enter`, `Ctrl+X`).

From PowerShell:

```powershell
wsl --shutdown
```

Reopen WSL. Verify:

```bash
ps -p 1 -o comm=
```

Expected output: `systemd`

---

## Phase 2 — Shell (Zsh + Starship + Plugins)

### 2.1 Install Zsh

```bash
sudo apt install -y zsh
zsh --version
```

Expected: `zsh 5.9` or newer.

### 2.2 Install Starship prompt

```bash
curl -sS https://starship.rs/install.sh | sh
starship --version
```

Confirm install location (default `/usr/local/bin`) when prompted. Enter sudo password.

### 2.3 Install Zsh plugins (autosuggestions + syntax highlighting + history search)

These are scripts (not apt packages), cloned from GitHub:

```bash
mkdir -p ~/.zsh/plugins
git clone https://github.com/zsh-users/zsh-autosuggestions ~/.zsh/plugins/zsh-autosuggestions
git clone https://github.com/zsh-users/zsh-syntax-highlighting ~/.zsh/plugins/zsh-syntax-highlighting
git clone https://github.com/zsh-users/zsh-history-substring-search ~/.zsh/plugins/zsh-history-substring-search
ls ~/.zsh/plugins/
```

Expected output: three folders.

### 2.4 Create `.zshrc`

Reset any existing `.zshrc` and create new:

```bash
rm -f ~/.zshrc

cat > ~/.zshrc << 'EOF'
# ============ History ============
HISTFILE=~/.zsh_history
HISTSIZE=10000
SAVEHIST=10000
setopt SHARE_HISTORY HIST_IGNORE_DUPS HIST_IGNORE_SPACE

# ============ Completion ============
autoload -Uz compinit && compinit
zstyle ':completion:*' menu select
zstyle ':completion:*' matcher-list 'm:{a-zA-Z}={A-Za-z}'

# ============ Behavior ============
setopt AUTO_CD INTERACTIVE_COMMENTS NO_BEEP NO_BANG_HIST

# ============ Editor ============
export VISUAL="nvim"
export EDITOR="$VISUAL"
export SUDO_EDITOR="nvim"

# ============ PATH (dedupe) ============
typeset -U path
path=(~/.local/bin ~/bin $path)

# ============ Aliases ============
alias ll='ls -lah'
alias la='ls -A'
alias ..='cd ..'
alias ...='cd ../..'
alias grep='grep --color=auto'

# ============ Plugins ============
source ~/.zsh/plugins/zsh-autosuggestions/zsh-autosuggestions.zsh
source ~/.zsh/plugins/zsh-syntax-highlighting/zsh-syntax-highlighting.zsh

# History substring search (UP/DOWN filter by typed text)
# Must be AFTER syntax-highlighting, BEFORE starship
source ~/.zsh/plugins/zsh-history-substring-search/zsh-history-substring-search.zsh
bindkey "^[[A" history-substring-search-up
bindkey "^[[B" history-substring-search-down
bindkey "^[OA" history-substring-search-up
bindkey "^[OB" history-substring-search-down

# ============ Starship prompt (must be last) ============
eval "$(starship init zsh)"
EOF
```

**Note**: Tool aliases (`fd`, `bat`, `eza`, `lg`) and integrations (`mise`, `zoxide`, `fzf`) are appended later in Phase 3.

**Why each section**:
- **History**: Zsh defaults don't persist history; these settings fix that
- **Completion**: enables tab-completion menu with case-insensitive matching
- **Behavior**: `AUTO_CD` (type folder name to cd), `NO_BANG_HIST` (avoid `event not found` on `!`)
- **PATH dedupe**: `typeset -U path` prevents duplicate PATH entries
- **Plugins**: must source autosuggestions before syntax-highlighting (which must be last)
- **Starship**: must be last so its `PROMPT` setting isn't overridden

### 2.5 Set Zsh as default shell

```bash
chsh -s $(which zsh)
getent passwd $USER | cut -d: -f7
```

Expected output: `/usr/bin/zsh`

Enter Zsh now (without restart):

```bash
exec zsh
```

Prompt should change to Starship's multi-line format.

---

## Phase 3 — Modern CLI Tools

### 3.1 Install tools available via apt

```bash
sudo apt install -y \
  ripgrep fd-find bat fzf tmux btop zoxide
```

Verify:

```bash
echo "rg:    $(rg --version | head -1)"
echo "fd:    $(fdfind --version)"
echo "bat:   $(batcat --version)"
echo "fzf:   $(fzf --version)"
echo "tmux:  $(tmux -V)"
echo "btop:  $(btop --version)"
echo "zox:   $(zoxide --version)"
```

**Note**: Ubuntu installs `fd` as `fdfind` and `bat` as `batcat` (binary name conflicts with other packages). We alias them to standard names later.

### 3.2 Install eza (modern `ls`)

eza is not in Ubuntu 24.04 default repo. Add maintainer's repo:

```bash
sudo mkdir -p /etc/apt/keyrings
wget -qO- https://raw.githubusercontent.com/eza-community/eza/main/deb.asc | sudo gpg --dearmor -o /etc/apt/keyrings/gierens.gpg
echo "deb [signed-by=/etc/apt/keyrings/gierens.gpg] http://deb.gierens.de stable main" | sudo tee /etc/apt/sources.list.d/gierens.list
sudo chmod 644 /etc/apt/keyrings/gierens.gpg /etc/apt/sources.list.d/gierens.list
sudo apt update && sudo apt install -y eza
eza --version
```

### 3.3 Install lazygit (TUI for Git)

Not in apt. Install latest binary from GitHub:

```bash
LAZYGIT_VERSION=$(curl -s "https://api.github.com/repos/jesseduffield/lazygit/releases/latest" | grep -Po '"tag_name": "v\K[^"]*')
echo "Installing lazygit v${LAZYGIT_VERSION}..."
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

### 3.5 Add tool aliases and integrations to `.zshrc`

```bash
cat >> ~/.zshrc << 'EOF'

# ============ Tool aliases ============
alias fd='fdfind'
alias bat='batcat'
alias ls='eza --icons'
alias ll='eza -lah --icons --git'
alias la='eza -a --icons'
alias lt='eza --tree --level=2 --icons'
alias lg='lazygit'

# ============ Tool integrations ============
# zoxide: smart cd — provides 'z' command
eval "$(zoxide init zsh)"

# fzf: fuzzy finder — Ctrl+R (history), Ctrl+T (file search)
[ -f /usr/share/doc/fzf/examples/key-bindings.zsh ] && source /usr/share/doc/fzf/examples/key-bindings.zsh
[ -f /usr/share/doc/fzf/examples/completion.zsh ] && source /usr/share/doc/fzf/examples/completion.zsh
EOF
```

**Note**: mise activate hook is added in Phase 4 (after mise is installed).

### 3.6 Reload and verify

```bash
exec zsh
type fd bat ls ll lg
type z
```

All should resolve to aliases or shell functions.

---

## Phase 4 — mise + Programming Languages

### 4.1 Install mise

```bash
curl https://mise.run | sh
~/.local/bin/mise --version
```

mise installs to `~/.local/bin/mise` (user-level, no sudo).

### 4.2 Activate mise in `.zshrc`

```bash
cat >> ~/.zshrc << 'EOF'

# ============ mise (multi-language version manager) ============
eval "$(~/.local/bin/mise activate zsh)"
EOF

exec zsh
mise --version
```

### 4.3 Install Node.js 22 + pnpm

```bash
mise use --global node@22
mise use --global pnpm@latest
node --version
pnpm --version
```

`mise use` installs and sets as global default in one step.

### 4.4 Install Python 3.12 + uv

⚠️ Python compiles from source (1–3 minutes).

```bash
mise use --global python@3.12
mise use --global uv@latest
python --version
uv --version
```

**Why uv**: 10–100x faster than pip, replaces pip + venv + pyenv combo.

### 4.5 Install PHP build dependencies

PHP must compile from source. Install dev libraries first:

```bash
sudo apt install -y \
  build-essential autoconf libtool bison re2c pkg-config \
  libxml2-dev libssl-dev libsqlite3-dev libbz2-dev \
  libcurl4-openssl-dev libgd-dev libjpeg-dev libpng-dev \
  libfreetype6-dev libwebp-dev libxpm-dev libonig-dev \
  libreadline-dev libtidy-dev libxslt1-dev libzip-dev \
  zlib1g-dev libargon2-dev libsodium-dev libldap2-dev libpq-dev libicu-dev
```

**Why**: PHP needs these to enable common extensions (mysql, gd, zip, curl, sodium, etc). Without them, PHP compiles but lacks essential features. Estimated 1–2 minutes install.

### 4.6 Install PHP 8.3

⚠️ PHP compiles from source (3–7 minutes; CPU intensive).

```bash
mise use --global php@8.3
php --version
```

### 4.7 Install Composer

Composer uses its own installer (not via mise):

```bash
curl -sS https://getcomposer.org/installer | php
sudo mv composer.phar /usr/local/bin/composer
composer --version
```

### 4.8 Verify all runtimes

```bash
echo "Node:     $(node --version)"
echo "pnpm:     $(pnpm --version)"
echo "Python:   $(python --version)"
echo "uv:       $(uv --version)"
echo "PHP:      $(php --version | head -1)"
echo "Composer: $(composer --version | head -1)"
mise list
```

### 4.9 Install Go (optional, when needed)

Skip until you have a Go project. To install later:

```bash
mise use --global go@1.23
go install golang.org/x/tools/gopls@latest        # LSP for Neovim
```

---

## Phase 5 — Neovim + LazyVim

### 5.1 Install Neovim 0.12 (AppImage extracted)

WSL2 sometimes has FUSE issues. Extract AppImage instead of running directly:

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

Expected: `NVIM v0.12.x`

**Why this approach**: AppImage normally requires FUSE which WSL2 lacks by default. Extracting and symlinking achieves the same result without FUSE dependency. Future updates: re-run this section with new download.

### 5.2 Install LazyVim starter

If you have an existing Neovim config, back it up first:

```bash
TIMESTAMP=$(date +%Y%m%d-%H%M%S)
mv ~/.config/nvim ~/.config/nvim.backup-$TIMESTAMP 2>/dev/null
rm -rf ~/.local/share/nvim ~/.local/state/nvim ~/.cache/nvim
```

Clone LazyVim starter:

```bash
git clone https://github.com/LazyVim/starter ~/.config/nvim
rm -rf ~/.config/nvim/.git
ls ~/.config/nvim
```

Expected: `LICENSE`, `README.md`, `init.lua`, `lazy-lock.json`, `lua/`, `stylua.toml`.

### 5.3 First Neovim launch — auto-bootstrap plugins

```bash
nvim
```

LazyVim auto-installs plugin manager (lazy.nvim) and ~50 base plugins (2–5 minutes). Wait for completion. Press `Enter` if prompted. When done:

```vim
:qa
```

Reopen for clean state:

```bash
nvim
```

Verify no errors. Use `:checkhealth` to inspect.

### 5.4 Install LSP servers via Mason

In Neovim, install language servers used by your work:

```vim
:MasonInstall typescript-language-server intelephense pyright tailwindcss-language-server eslint-lsp lua-language-server vue-language-server prettier stylua php-cs-fixer
```

Wait 2–5 minutes for downloads. Verify with `:Mason` (look for ✓ marks under "Installed").

**Skipped**: `gopls` requires `go` in PATH. Install when you install Go (Phase 4.9).

### 5.5 Enable LazyVim Extras

LazyVim Extras add language-specific tuning beyond basic LSP:

```vim
:LazyExtras
```

Navigate to and press `x` on each:
- `lang.typescript`
- `lang.php`
- `lang.python`
- `lang.tailwind`
- `lang.json`

`q` to close. Then `:qa` and reopen Neovim. New plugins auto-install.

**Why Extras vs Mason**: Mason downloads LSP binary; Extras configure how Neovim uses it (Blade template support for PHP, Tailwind class autocomplete, JSON schema validation, etc).

### 5.6 Verify LSP works

Open a project file:

```bash
nvim /var/www/Apps/some-project/some-file.php
# or any PHP/JS/TS file
```

Type to trigger autocomplete. Press `K` on a function name for hover docs. Press `gd` to jump to definition.

---

## Phase 6 — Docker

### 6.1 Remove any existing Docker installation

If migrating from Docker Desktop or older install:

```bash
sudo apt remove -y docker docker-engine docker.io containerd runc 2>/dev/null
```

### 6.2 Install Docker Engine (native, no Desktop)

```bash
sudo apt update
sudo apt install -y ca-certificates curl gnupg lsb-release

sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt update
sudo apt install -y \
  docker-ce docker-ce-cli containerd.io \
  docker-buildx-plugin docker-compose-plugin
```

### 6.3 Add user to docker group (no sudo needed)

```bash
sudo usermod -aG docker $USER
```

Logout + login (or `newgrp docker`) to apply group change.

### 6.4 Enable Docker via systemd

```bash
sudo systemctl enable docker
sudo systemctl start docker
sudo systemctl status docker
```

Expected: `active (running)`. Press `q` to exit status view.

### 6.5 Test Docker

```bash
docker --version
docker compose version
docker run --rm hello-world
```

The hello-world container should download, run, print message, and exit cleanly.

---

## Phase 7 — Claude Code (Optional)

### 7.1 Install Claude Code CLI

```bash
npm install -g @anthropic-ai/claude-code
claude --version
which claude
```

Path should resolve to mise-managed Node directory.

### 7.2 Verify no API key environment

```bash
echo "${ANTHROPIC_API_KEY:-not set}"
```

Should output: `not set`. If a key is set, you'll be charged per-token instead of using Pro subscription. Unset:

```bash
unset ANTHROPIC_API_KEY
```

Check for it in shell init files:

```bash
grep -n ANTHROPIC ~/.zshrc ~/.profile 2>/dev/null
```

If found, remove those lines.

### 7.3 Authenticate

```bash
claude
```

Choose "Log in with Claude.ai", complete browser OAuth flow. After success, exit:

```
/exit
```

### 7.4 Add tmux helper to `.zshrc`

```bash
cat >> ~/.zshrc << 'EOF'

# ============ tmux quick workflows ============
work() {
  local name="${1:-work}"
  if tmux has-session -t "$name" 2>/dev/null; then
    tmux attach -t "$name"
  else
    tmux new-session -d -s "$name" -c "$PWD"
    tmux send-keys -t "$name" "nvim" C-m
    tmux split-window -h -t "$name" -p 30 -c "$PWD"
    tmux send-keys -t "$name" "claude" C-m
    tmux select-pane -t "$name":0.0
    tmux attach -t "$name"
  fi
}

alias tls='tmux ls'
alias tat='tmux attach -t'
tkill() { tmux kill-session -t "${1:-work}"; }
EOF

exec zsh
```

### 7.5 Test workflow

```bash
cd /var/www/Apps/some-project    # or any project folder
work
```

tmux opens with Neovim (left, 70%) and Claude Code (right, 30%). Switch panes with `Ctrl+b ←` / `Ctrl+b →`.

---

## Phase 8 — Final Configuration

### 8.1 Set Git identity

```bash
git config --global user.name "Your Name"
git config --global user.email "your-email@example.com"
git config --global init.defaultBranch main
git config --global pull.rebase false
```

### 8.2 Authenticate GitHub CLI

```bash
gh auth login
```

Choose:
- GitHub.com
- HTTPS (or SSH)
- Login with browser
- Authorize in browser

### 8.3 Generate SSH key (if using SSH for git)

```bash
ssh-keygen -t ed25519 -C "your-email@example.com"
# Press Enter to accept default location
# Optionally set passphrase
```

Add to GitHub:

```bash
gh ssh-key add ~/.ssh/id_ed25519.pub --title "WSL Ubuntu"
```

### 8.4 Verify final state

```bash
echo "=== Shell ==="
echo "Shell: $SHELL"
zsh --version

echo "=== Runtimes ==="
mise list

echo "=== CLI tools ==="
for tool in rg fd bat eza fzf tmux btop zoxide lazygit gh starship; do
  if command -v $tool >/dev/null 2>&1; then
    echo "$tool: OK"
  else
    echo "$tool: MISSING"
  fi
done

echo "=== Editor ==="
nvim --version | head -1

echo "=== Docker ==="
docker --version
docker compose version

echo "=== AI ==="
claude --version 2>/dev/null || echo "claude: not installed"
```

All should report installed.

---

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
