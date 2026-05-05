# 02 — Shell

## Components

| Component | Purpose | Location |
|-----------|---------|----------|
| **Zsh** | Primary interactive shell | `/usr/bin/zsh` |
| **Starship** | Cross-shell prompt | `/usr/local/bin/starship` |
| **zsh-autosuggestions** | History-based command suggestions | `~/.zsh/plugins/` |
| **zsh-syntax-highlighting** | Color valid/invalid commands | `~/.zsh/plugins/` |
| **Nerd Font** | Terminal font with programming icons | Windows Terminal config |

## `~/.zshrc` annotated

The full `.zshrc` is also archived in [reference/zshrc-current.txt](../reference/zshrc-current.txt). Here is what each section does:

### History

```zsh
HISTFILE=~/.zsh_history
HISTSIZE=10000
SAVEHIST=10000
setopt SHARE_HISTORY HIST_IGNORE_DUPS HIST_IGNORE_SPACE
```

- `HISTFILE` — without this, Zsh doesn't persist history at all
- `10000` lines — large enough for months of work
- `SHARE_HISTORY` — open two terminals, history syncs between them
- `HIST_IGNORE_DUPS` — typing `ls` 5 times saves only 1 entry
- `HIST_IGNORE_SPACE` — commands prefixed with space don't enter history (good for sensitive data)

### Completion

```zsh
autoload -Uz compinit && compinit
zstyle ':completion:*' menu select
zstyle ':completion:*' matcher-list 'm:{a-zA-Z}={A-Za-z}'
```

- `compinit` — Zsh's completion system (tab completion for commands, paths, git subcommands, etc)
- `menu select` — Tab+Tab shows interactive menu
- `matcher-list` — case-insensitive matching (`doc` matches `Documents`)

### Behavior

```zsh
setopt AUTO_CD INTERACTIVE_COMMENTS NO_BEEP
setopt NO_BANG_HIST
```

- `AUTO_CD` — typing folder name (no `cd`) navigates to it
- `INTERACTIVE_COMMENTS` — `# comments` work in interactive shell
- `NO_BEEP` — no annoying beep on errors
- `NO_BANG_HIST` — disable `!` history expansion (avoids `event not found` errors when `!` appears in strings)

### Editor

```zsh
export VISUAL="nvim"
export EDITOR="$VISUAL"
export SUDO_EDITOR="nvim"
```

Tools like `git commit`, `crontab -e`, `sudoedit` use Neovim.

### PATH dedupe

```zsh
typeset -U path
path=(~/.local/bin ~/bin $path)
```

`typeset -U path` makes the `path` array unique — prevents PATH duplicates from misbehaving init scripts.

### Aliases

```zsh
alias ll='ls -lah'
alias la='ls -A'
alias ..='cd ..'
alias ...='cd ../..'
alias grep='grep --color=auto'
```

Plus tool-specific aliases (see [03-tools.md](03-tools.md) for `fd`, `bat`, `eza`, etc).

### Plugins (loaded BEFORE prompt)

```zsh
source ~/.zsh/plugins/zsh-autosuggestions/zsh-autosuggestions.zsh
source ~/.zsh/plugins/zsh-syntax-highlighting/zsh-syntax-highlighting.zsh
```

Order matters: syntax-highlighting MUST be loaded last (it hooks into the line editor).

### Tool integrations

```zsh
eval "$(~/.local/bin/mise activate zsh)"
eval "$(zoxide init zsh)"
[ -f /usr/share/doc/fzf/examples/key-bindings.zsh ] && source /usr/share/doc/fzf/examples/key-bindings.zsh
```

These tools install binaries but require shell hooks to be functional. Without these `eval`/`source` calls, the binaries exist but commands like `mise`, `z`, and `Ctrl+R` fzf don't work.

### Starship prompt (LAST)

```zsh
eval "$(starship init zsh)"
```

Starship sets `PROMPT`. Anything that sets `PROMPT` after this would override Starship — so it's the last line.

### tmux helper function

```zsh
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
```

`work [name]` spawns a tmux session with Neovim (left, 70%) and Claude Code (right, 30%). See [08-claude-code.md](08-claude-code.md).

## Key features in action

### Autosuggestions

Type `cd ` (with a space). If you've cd'd before, a gray suggestion appears after the cursor. Press `→` to accept, or keep typing.

### Syntax highlighting

- `lsxxxx` shows in **red** (command not found)
- `ls` shows in **green** (valid command)

Catches typos before pressing Enter.

### Tab completion menu

```bash
cd /usr/lo<Tab><Tab>
```

Interactive menu appears. Navigate with arrow keys, Enter to select.

### Auto-cd

```bash
Documents     # without `cd`, just types the folder name
```

Equivalent to `cd Documents`.

## Customization tips

### Add a personal alias

Edit `~/.zshrc`, find the `# Aliases` section, add:

```zsh
alias myalias='some command'
```

Reload:

```bash
source ~/.zshrc
# or
exec zsh
```

### Change Starship prompt

Default Starship config is auto. To customize:

```bash
mkdir -p ~/.config
starship preset gruvbox-rainbow -o ~/.config/starship.toml
```

Available presets: `nerd-font-symbols`, `bracketed-segments`, `plain-text-symbols`, `tokyo-night`, `gruvbox-rainbow`, `pastel-powerline`, `pure-preset`.

Or edit `~/.config/starship.toml` manually. Reference: https://starship.rs/config/

### Disable a feature

For example, to disable autosuggestions, comment out the line:

```zsh
# source ~/.zsh/plugins/zsh-autosuggestions/zsh-autosuggestions.zsh
```

Reload shell.

## Troubleshooting

### Shell shows weird characters (□, ?)

Nerd Font not active in Windows Terminal. Open Settings → Profiles → Ubuntu → Appearance → Font face → choose installed Nerd Font (e.g., "JetBrainsMono Nerd Font").

### `event not found` error

Triggered by `!` in strings. Already mitigated by `setopt NO_BANG_HIST`. If still occurring, use single quotes: `echo 'hello!'` instead of `echo "hello!"`.

### `command not found: mise/z/...` after editing `.zshrc`

Reload shell: `exec zsh`. If it persists, check the order of `eval` lines in `.zshrc` (mise/zoxide must be after PATH setup).

### Slow shell startup

Profile with:

```bash
time zsh -i -c exit
```

If > 500ms, find the culprit:

```bash
zsh -xv -i -c exit 2>&1 | tail -50
```

Likely causes: `compinit` running on stale dump (delete `~/.zcompdump`), or a slow plugin.
