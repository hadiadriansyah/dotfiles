# 03 — Modern CLI Tools

Modern alternatives to traditional Unix utilities. All are installed system-wide and aliased in `~/.zshrc`.

## Quick reference

| Modern tool | Replaces | Alias |
|-------------|----------|-------|
| `ripgrep` (rg) | `grep -r` | — |
| `fd-find` | `find` | `fd` |
| `bat` | (additional, not replacing `cat`) | `bat` (Ubuntu installs as `batcat`) |
| `eza` | `ls` | `ls`, `ll`, `la`, `lt` |
| `fzf` | (no replacement, new fuzzy finder) | — |
| `zoxide` | `cd` (additional command `z`) | — |
| `lazygit` | `git` (TUI, not replacement) | `lg` |
| `gh` | (GitHub web UI) | — |
| `tmux` | (terminal multiplexer) | — |
| `btop` | `top`, `htop` | — |

## ripgrep (rg)

Fast text search, replaces `grep -r`. Auto-respects `.gitignore` and skips `node_modules`, `.git/`, etc.

### Common usage

```bash
rg "function login"                 # search in current directory
rg "TODO" -t php                    # only PHP files
rg "Auth::" /var/www/Apps           # search in specific path
rg -i "user"                        # case-insensitive
rg -l "config"                      # list filenames only
rg -c "error"                       # count matches per file
rg -A 3 "function"                  # 3 lines AFTER match
rg -B 2 "function"                  # 2 lines BEFORE match
rg -C 2 "function"                  # 2 lines CONTEXT (before & after)
rg --hidden ".env"                  # include hidden files
rg --no-ignore "secret"             # ignore .gitignore
```

### Performance

10-100x faster than `grep -r` on large codebases (multi-threaded, written in Rust).

## fd (fdfind)

Fast file finder, replaces `find`. Note: Ubuntu installs as `fdfind`, aliased to `fd` in `.zshrc`.

### Common usage

```bash
fd config                           # find anything matching "config"
fd -e php                           # files with .php extension
fd -e js -e ts                      # multiple extensions
fd -t f config.php                  # only files (not folders)
fd -t d node_modules                # only directories
fd -H ".env"                        # include hidden files
fd config /var/www                  # search in specific path
fd -E vendor                        # exclude vendor folders
fd -x grep "TODO" {}                # execute command on each match
```

### Why fd over find

- Auto-ignores `.gitignore`
- Faster (multi-threaded)
- Sane default (case-insensitive smart, regex, colored output)
- Simpler syntax: `fd -e php` vs `find . -name "*.php" -not -path "*/vendor/*"`

## bat (batcat)

Cat with syntax highlighting and line numbers. Aliased to `bat` (Ubuntu installs as `batcat`).

### Common usage

```bash
bat ~/.zshrc                        # show file with syntax highlight
bat app/Models/User.php             # auto-detect language
bat -p file.txt                     # plain mode (no decorations)
bat -r 10:20 file.php               # only lines 10-20
bat --diff file.php                 # show with git diff markers
bat --list-languages                # list supported languages
```

### Important: bat does NOT replace cat

`cat` is still aliased to `cat` (not `bat`) because:
- Scripts and pipes expect plain `cat` output
- `bat`'s formatting can break pipes

Use `bat` for human reading, `cat` for scripting.

### Use as pager for help

```bash
mise --help | bat -l help
```

## eza

Modern `ls` with icons (Nerd Font), git status, tree view.

### Aliases (in `.zshrc`)

```zsh
alias ls='eza --icons'
alias ll='eza -lah --icons --git'
alias la='eza -a --icons'
alias lt='eza --tree --level=2 --icons'
```

### Common usage

```bash
ls                                  # with icons
ll                                  # long format + git status
la                                  # all files
lt                                  # tree, 2 levels deep
eza --tree --level=3                # custom tree depth
eza -lah --git --icons --header     # full info with header
eza -lah --sort=modified            # sort by modification time
eza -lah --sort=size                # sort by size
```

### Git status indicators (in `ll`)

- `M` — modified
- `N` — new (untracked)
- `D` — deleted
- `R` — renamed
- `T` — type changed
- `I` — ignored
- `-` — unchanged

## fzf

Interactive fuzzy finder. The most powerful tool here once you adapt.

### Keyboard shortcuts (active in shell)

| Shortcut | Action |
|----------|--------|
| `Ctrl+R` | Search command history (game changer) |
| `Ctrl+T` | Search files in current directory |
| `Alt+C` | cd to subdirectory (fuzzy) |

### Inside fzf interface

| Key | Action |
|-----|--------|
| Arrow keys / `Ctrl+J/K` | Navigate |
| Type | Filter results (fuzzy match) |
| `Enter` | Select |
| `Esc` / `Ctrl+C` | Cancel |

### Usage examples

```bash
# Pick a file to edit
nvim "$(fzf)"

# cd to a project (fuzzy match folder name)
cd "$(fd -t d -d 3 . ~/projects | fzf)"

# Kill a process by name
ps aux | fzf | awk '{print $2}' | xargs kill

# Pick a git branch
git checkout "$(git branch | fzf | tr -d ' ')"
```

### Real workflow win

> "I ran a long docker compose command 2 weeks ago, what was it?"

Press `Ctrl+R`, type `docker`, scroll through matches, hit Enter. Done in 2 seconds.

## zoxide

Smart `cd`. Learns the directories you visit most. Replaces the older `z` script.

### Usage

```bash
cd /var/www/Apps/some-project       # visit a folder normally
cd ~                                # go elsewhere
z apps                              # smart-cd back (matches partial path)
z some                              # also works (matches "some-project")
zi                                  # interactive picker (uses fzf)
```

### Database

zoxide builds a frecency database (frequency + recency). The more you visit a folder, the higher it ranks.

```bash
zoxide query --list                 # show all tracked folders
zoxide remove /old/path             # remove from database
```

### Limitation

zoxide only knows about folders you've cd'd to manually before. It can't jump to folders you've never visited.

## lazygit

Terminal UI for git. Replaces most git commands for daily work.

### Launch

```bash
lazygit
# or alias
lg
```

### Key commands inside lazygit

| Key | Action |
|-----|--------|
| `?` | Show help (use this!) |
| `Tab` | Switch panel (Files, Branches, Commits, Stash) |
| `space` | Stage/unstage file or hunk |
| `c` | Commit |
| `P` | Push |
| `p` | Pull |
| `s` | Stash |
| `Enter` | Drill into details |
| `q` | Quit |
| `<` `>` | Switch branch |

### Workflow speedup

Replaces sequences like:

```bash
git status
git diff app/Models/User.php
git add app/Models/User.php
git diff --cached
git commit -m "Update User model"
git push origin main
```

With: `lg` → space → c → write message → P. ~5 seconds total.

## GitHub CLI (gh)

GitHub operations without leaving the terminal.

### Authentication (one-time setup)

```bash
gh auth login
```

Choose:
- GitHub.com
- HTTPS (or SSH)
- Login with browser
- Authorize in browser

### Common commands

```bash
gh repo create my-project --private          # new private repo
gh repo clone owner/repo                     # clone (no URL needed)
gh repo view --web                           # open repo in browser

# Pull requests
gh pr create                                 # interactive PR creation
gh pr list                                   # PRs in current repo
gh pr checkout 42                            # checkout PR #42 locally
gh pr review --approve
gh pr merge --squash

# Issues
gh issue create --title "Bug: X"
gh issue list
gh issue close 5

# CI/Actions
gh run list                                  # recent workflow runs
gh run watch                                 # follow latest run
```

## tmux

Terminal multiplexer. Persistent sessions, multiple panes, transferable to remote SSH workflow.

See [08-claude-code.md](08-claude-code.md) for the `work` function that pre-configures tmux with Neovim + Claude Code.

### Default keybindings

All tmux commands use prefix `Ctrl+b`, then a key:

| Combo | Action |
|-------|--------|
| `Ctrl+b %` | Split vertical |
| `Ctrl+b "` | Split horizontal |
| `Ctrl+b ←/→/↑/↓` | Navigate panes |
| `Ctrl+b z` | Zoom pane (toggle) |
| `Ctrl+b d` | Detach session (still running) |
| `Ctrl+b x` | Kill pane |
| `Ctrl+b [` | Scroll mode (q to exit) |
| `Ctrl+b c` | New window |
| `Ctrl+b n/p` | Next/prev window |
| `Ctrl+b ,` | Rename window |

### Session management

```bash
tmux new -s mysession                # new named session
tmux ls                              # list sessions
tmux attach -t mysession             # reattach
tmux kill-session -t mysession       # destroy
```

### Aliases (in `.zshrc`)

```zsh
alias tls='tmux ls'
alias tat='tmux attach -t'
tkill() { tmux kill-session -t "${1:-work}"; }
```

## btop

Modern system monitor. Replaces `top` and `htop`.

```bash
btop
```

### Inside btop

| Key | Action |
|-----|--------|
| `q` | Quit |
| `?` | Help |
| `Esc` | Menu |
| `+/-` | Expand/collapse panels |
| Mouse | Click anywhere |

Useful when troubleshooting "why is my laptop slow?" — see CPU/RAM usage by process.

## Quick command cheat sheet

```bash
# Find & search
rg "pattern"                # search text recursively
fd "filename"               # find files by name
fzf                         # interactive picker (also Ctrl+R, Ctrl+T)

# View files
bat file.php                # syntax-highlighted cat
ll                          # detailed ls with git
lt                          # tree view

# Navigate
cd /var/www/Apps            # standard cd
z apps                      # smart cd (after visiting once)
zi                          # interactive cd

# Git
lg                          # lazygit TUI
gh pr create                # create PR
gh repo view --web          # open repo in browser

# System
btop                        # system monitor
tmux new -s work            # tmux session
work                        # custom: Neovim + Claude Code split
```
