# 09 — Troubleshooting

A collection of issues encountered (or likely to be encountered) and their solutions. When you face an issue, search this doc first (`Ctrl+F`).

## Shell

### `command not found` for installed tool

After editing `.zshrc`, the change isn't applied. Reload:

```bash
exec zsh
# or
source ~/.zshrc
```

If still not found, check PATH:

```bash
echo $PATH | tr ':' '\n'
which <tool>
```

The directory containing the tool must be in PATH.

### `zsh: event not found: ...` when typing string with `!`

Zsh's history expansion. Already mitigated by `setopt NO_BANG_HIST` in `.zshrc`. If it still appears, use single quotes:

```bash
echo 'hello world!'        # works
echo "hello world!"        # may error
```

### `zsh: no matches found: pattern*`

Zsh strict glob. When wildcard matches nothing, Zsh errors instead of leaving literal (Bash leaves it).

Fix per command:
```bash
ls *.txt 2>/dev/null       # suppress error
```

Or globally in `.zshrc`:
```zsh
setopt NULL_GLOB           # treat unmatched as empty
```

### Slow shell startup (>500ms)

Profile:
```bash
time zsh -i -c exit
```

Common causes:
- Stale completion cache: `rm ~/.zcompdump && exec zsh`
- Plugin loading slow: `zsh -xv -i -c exit 2>&1 | tail -100`
- mise activate slow: shouldn't be — but `mise self-update` if old

### Prompt shows weird characters (□, ?, missing icons)

Nerd Font not active in terminal. Set in Windows Terminal:
- Settings → Profiles → Ubuntu → Appearance → Font face
- Choose Nerd Font (e.g., "JetBrainsMono Nerd Font")

Verify Nerd Font installed in Windows: open `C:\Windows\Fonts`, search "Nerd".

## WSL

### WSL slow

If working in `/mnt/c/...`:
- Stop. Move project to `/home/adha/`.
- File access on `/mnt/c/` is 10-100x slower in WSL2.

Verify:
```bash
pwd                        # should be /home/adha/... or /var/www/...
```

### High RAM usage by WSL

Check `.wslconfig`:
```bash
cat /mnt/c/Users/<your-windows-user>/.wslconfig
```

Should have:
```ini
[wsl2]
memory=4GB
```

Without limit, WSL uses up to ~80% of physical RAM.

After editing `.wslconfig`, from PowerShell:
```powershell
wsl --shutdown
```

Reopen WSL.

### Cannot reach localhost from Windows browser

Ensure `localhostForwarding=true` in `.wslconfig`.

If still failing, check the service is binding to `0.0.0.0` not `127.0.0.1`:
```bash
ss -tlnp | grep <port>
```

A service bound to `127.0.0.1` inside WSL might not be reachable from Windows. Bind to `0.0.0.0`.

### systemd not running

Check:
```bash
ps -p 1 -o comm=           # should output: systemd
```

If `init` instead, edit `/etc/wsl.conf`:
```ini
[boot]
systemd=true
```

Then from PowerShell:
```powershell
wsl --shutdown
```

## mise / runtime

### `mise: command not found`

mise activate hook missing in `.zshrc`. Add:
```zsh
eval "$(~/.local/bin/mise activate zsh)"
```

Reload: `exec zsh`.

If `~/.local/bin/mise` doesn't exist:
```bash
curl https://mise.run | sh
```

### `node: command not found` after `mise install`

mise needs to refresh shims:
```bash
mise reshim
exec zsh
```

### mise install fails to compile (PHP / Python)

Build dependencies missing. For PHP:
```bash
sudo apt install -y build-essential autoconf libtool bison re2c \
  pkg-config libxml2-dev libssl-dev libsqlite3-dev libbz2-dev \
  libcurl4-openssl-dev libgd-dev libjpeg-dev libpng-dev \
  libfreetype6-dev libwebp-dev libxpm-dev libonig-dev \
  libreadline-dev libtidy-dev libxslt1-dev libzip-dev \
  zlib1g-dev libargon2-dev libsodium-dev libldap2-dev libpq-dev
```

For Python:
```bash
sudo apt install -y build-essential libssl-dev libffi-dev \
  libsqlite3-dev libbz2-dev liblzma-dev libreadline-dev zlib1g-dev
```

### Wrong version active in a project folder

Check `.mise.toml`:
```bash
cat .mise.toml
mise current
```

If `.mise.toml` is missing, version comes from global. Set:
```bash
mise use node@22 php@8.3
```

This creates `.mise.toml` in current folder.

### Global npm packages disappear after `mise upgrade node`

Expected behavior. mise installs npm packages per Node version. After upgrading:
```bash
mise upgrade node
npm install -g @anthropic-ai/claude-code   # reinstall
```

To preserve, list what you have:
```bash
npm list -g --depth=0
```

## Neovim / LazyVim

### Neovim opens but no plugins load

Check Lazy:
```vim
:Lazy
```

If "no plugins" or errors, the bootstrap failed. Try:
```vim
:Lazy sync
```

If still broken:
```bash
rm -rf ~/.local/share/nvim ~/.local/state/nvim ~/.cache/nvim
nvim
```

LazyVim re-bootstraps from scratch.

### LSP not active in a file

```vim
:LspInfo
```

If empty, check:
1. Filetype recognized? `:set filetype?`
2. LSP installed? `:Mason` — install if missing
3. LazyVim Extra enabled? `:LazyExtras`
4. Restart LSP: `:LspRestart`

### Treesitter parser failed

```vim
:TSInstall <language>
:TSUpdate
```

Common: `php`, `javascript`, `typescript`, `tsx`, `python`, `lua`, `markdown`, `bash`, `vim`.

### Mason package install fails

Check Mason logs:
```vim
:MasonLog
```

Common cause: missing system dep. Examples:
- gopls: needs `go` in PATH
- prettier: needs `node` in PATH
- php-cs-fixer: needs `php` in PATH

Verify in shell:
```bash
which node go php
```

### Slow Neovim startup

```vim
:Lazy profile
```

Lists plugins by load time. Plugins > 50ms should be lazy-loaded.

### Restore old config from backup

```bash
ls -d ~/.config/nvim.backup-*
mv ~/.config/nvim ~/.config/nvim.broken
mv ~/.config/nvim.backup-YYYYMMDD-HHMMSS ~/.config/nvim
rm -rf ~/.local/share/nvim ~/.cache/nvim
nvim
```

## Docker

### `Cannot connect to the Docker daemon`

```bash
sudo systemctl status docker
sudo systemctl start docker
sudo systemctl enable docker        # auto-start
```

If status shows `inactive`, also check:
```bash
sudo journalctl -u docker --tail 30
```

### `permission denied while trying to connect to Docker daemon`

User not in `docker` group:
```bash
sudo usermod -aG docker $USER
```

Then logout + login (or `newgrp docker`).

### Container won't start

```bash
docker logs <container> --tail 100
```

Common causes:
- Port conflict: `sudo ss -tlnp | grep <port>`
- Volume permission: `docker exec -it <container> ls -la /path/inside`
- Missing env: `docker inspect <container> | grep -A 20 Env`
- Out of memory: `docker stats`, `free -h`

### Disk filling up from Docker

```bash
docker system df                   # see breakdown
docker system prune                # safe: remove dangling
docker system prune -a             # aggressive: also unused images
docker builder prune               # build cache
```

⚠️ `docker system prune -a --volumes` deletes volumes. **Don't run on dev** unless you've backed up data.

### MySQL container shows healthy but can't connect

Wait — MySQL takes time on first start (init databases). Watch logs:
```bash
docker logs mysql-db -f
```

Wait for `ready for connections`. Or check health:
```bash
docker inspect --format='{{.State.Health.Status}}' mysql-db
```

### Project at `/var/www/...` not loading

After WSL restart, containers should auto-start. Verify:
```bash
docker ps
```

If stopped, start:
```bash
docker start $(docker ps -aq)
```

If specific container won't start, check its logs.

## Claude Code

### `claude: command not found`

Reinstall:
```bash
npm install -g @anthropic-ai/claude-code
exec zsh
which claude
```

If `npm` itself missing, reactivate mise:
```bash
exec zsh
which node npm
```

### Authentication loops or fails

```bash
claude auth logout
claude auth login
```

Choose "Log in with Claude.ai" and complete browser flow.

### Using API instead of subscription

```bash
echo $ANTHROPIC_API_KEY
```

If output isn't empty, you're on API tier. To use Pro subscription:
```bash
unset ANTHROPIC_API_KEY
```

Remove from `.zshrc` and `.profile` if exported there:
```bash
grep -n ANTHROPIC ~/.zshrc ~/.profile
```

Then restart Claude Code.

### Hit usage limit

Pro plan limits reset on rolling 5-hour windows. Wait, or:
- Upgrade to Max ($100/mo)
- Use API for excess

Optimize:
- Use `/model haiku` for simple tasks
- `/clear` to reset conversation between unrelated tasks
- Smaller, focused prompts

## Git

### Forgot to set Git identity (first commit fails)

```bash
git config --global user.name "Hadi Adriansyah"
git config --global user.email "your-email@example.com"
```

### Permission denied (publickey) when pushing

Set up SSH key for GitHub:
```bash
ssh-keygen -t ed25519 -C "your-email@example.com"
cat ~/.ssh/id_ed25519.pub             # copy this
gh ssh-key add ~/.ssh/id_ed25519.pub  # via gh CLI
```

Or use HTTPS with `gh auth login`.

### Accidentally committed secrets

Don't push! Remove from history:
```bash
git reset --soft HEAD~1               # uncommit (keep changes staged)
# Edit out the secret
git add -A
git commit -m "..."
```

If already pushed: rotate the secret immediately, then use `git filter-repo` to rewrite history (but secret is already exposed — just rotate).

## Network

### `apt install` fails: `Temporary failure resolving`

DNS issue in WSL. Try:
```bash
sudo nano /etc/resolv.conf
```

Add:
```
nameserver 8.8.8.8
nameserver 1.1.1.1
```

If `/etc/resolv.conf` is symlink and gets reset, edit `/etc/wsl.conf`:
```ini
[network]
generateResolvConf=false
```

Then `wsl --shutdown`.

### `npm install` very slow

Could be:
- WSL filesystem on `/mnt/c/`: move project to `~/projects/`
- pnpm not used: try `pnpm install` (3x faster than npm)
- Cache empty: subsequent installs faster

### Cannot reach external HTTPS

Check Cloudflare WARP if active. Sometimes needs disable for specific domains.

## Disk space

### Free up space (general)

```bash
# apt cache
sudo apt autoremove -y
sudo apt clean

# Docker
docker system prune
docker volume prune

# pnpm/npm cache
pnpm store prune
npm cache clean --force

# Old Neovim backups
ls -d ~/.config/nvim.backup-*
# Remove ones you don't need:
rm -rf ~/.config/nvim.backup-OLD-DATE
```

### Find large files

```bash
du -sh ~/* 2>/dev/null | sort -h | tail -20
du -sh ~/.* 2>/dev/null | sort -h | tail -20
```

```bash
# Files > 100 MB
fd -S +100m ~ -x ls -lh {}
```

### WSL VHD bloat

WSL2's virtual disk grows but doesn't shrink automatically. To compact (from PowerShell, after `wsl --shutdown`):

```powershell
diskpart
# Inside diskpart:
select vdisk file="C:\path\to\ext4.vhdx"
attach vdisk readonly
compact vdisk
detach vdisk
exit
```

The path is usually `%LOCALAPPDATA%\Packages\CanonicalGroupLimited.UbuntuXX.XXLTS_*\LocalState\ext4.vhdx`.

## Recovery

### Broken `.zshrc` — can't even open shell

If syntax error makes shell unusable:
```bash
# From PowerShell, use a different shell or bash:
wsl bash
nano ~/.zshrc                     # fix syntax
```

Or if `bash` itself is also broken (rare):
```powershell
wsl --shutdown
wsl --user root                   # boot as root
```

Edit / restore from backup:
```bash
ls ~/.zshrc.backup-* 2>/dev/null
cat ~/.zshrc.backup-XXX > ~/.zshrc
```

### Catastrophic: re-init WSL

Last resort. Backup first:
```bash
# Export project files
tar -czf ~/backup-projects.tar.gz ~/projects /var/www
# Move to /mnt/c/ for safety
mv ~/backup-projects.tar.gz /mnt/c/Users/<you>/Desktop/

# Export databases (or trust Docker volumes)
```

Then from PowerShell:
```powershell
wsl --shutdown
wsl --unregister Ubuntu                     # NUKES the WSL distro
wsl --install Ubuntu                        # reinstall fresh
```

Re-clone this dotfiles repo and re-follow each docs/*.md.

## When all else fails

1. Read the error message carefully (don't skim)
2. Search the exact error in this doc
3. Search GitHub issues for the failing tool
4. Ask Claude Code: `claude` then `> [paste error]. What's wrong and how do I fix it?`
5. Stack Overflow / project's Discord
6. As nuclear option: backup, re-init WSL, follow docs from scratch
