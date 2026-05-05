# 06 — Neovim + LazyVim

## Setup

- **Neovim**: 0.12.x (installed via AppImage extracted to `/opt/nvim/`)
- **Binary symlink**: `/usr/local/bin/nvim` → `/opt/nvim/AppRun`
- **Distribution**: LazyVim (https://www.lazyvim.org/)
- **Config**: `~/.config/nvim/`
- **Plugin data**: `~/.local/share/nvim/`
- **Cache**: `~/.cache/nvim/`

## Folder structure

LazyVim uses a convention you should follow:

```
~/.config/nvim/
├── init.lua                       # Entry point — DO NOT EDIT
├── lazy-lock.json                 # Plugin version lockfile
├── lazyvim.json                   # LazyVim Extras config
├── lua/
│   ├── config/
│   │   ├── autocmds.lua           # Custom autocmds
│   │   ├── keymaps.lua            # Custom keymaps (edit here!)
│   │   ├── lazy.lua               # Plugin manager bootstrap — DO NOT EDIT
│   │   └── options.lua            # Custom options (edit here!)
│   └── plugins/
│       └── *.lua                  # Custom plugin specs (add files here!)
└── stylua.toml                    # Lua formatter config
```

### Where to make changes

- **Add new plugin**: create file in `lua/plugins/` (e.g., `my-plugin.lua`)
- **Override existing plugin config**: create file in `lua/plugins/` that returns plugin spec
- **Custom keymaps**: edit `lua/config/keymaps.lua`
- **Custom options**: edit `lua/config/options.lua`
- **Don't touch** `init.lua` or `lua/config/lazy.lua`

## LazyVim Extras enabled

- `lang.typescript`
- `lang.php`
- `lang.python`
- `lang.tailwind`
- `lang.json`

These are stored in `~/.config/nvim/lazyvim.json`. Manage via:

```vim
:LazyExtras
```

## LSP servers (installed via Mason)

| Language | LSP | Tool name in Mason |
|----------|-----|--------------------|
| TypeScript/JavaScript | tsserver | `typescript-language-server` |
| PHP | Intelephense | `intelephense` |
| Python | Pyright | `pyright` |
| Tailwind CSS | tailwindcss-language-server | `tailwindcss-language-server` |
| Vue | Volar | `vue-language-server` |
| Lua | lua-language-server | `lua-language-server` |
| ESLint | eslint-lsp | `eslint-lsp` |

### Formatters

- `prettier` (JS/TS/JSON/CSS/HTML/YAML/Markdown)
- `php-cs-fixer` (PHP)
- `stylua` (Lua)
- Ruff (Python — comes via `lang.python` extra)

### Mason commands

```vim
:Mason                  " open Mason UI
:MasonInstall <name>    " install package
:MasonUninstall <name>  " uninstall
:MasonUpdate            " update Mason registry
:MasonLog               " show install logs (debug)
```

## Survival keymaps

Leader key is `<Space>`. Press `<Space>` and wait — which-key popup shows everything.

### Files

| Keymap | Action |
|--------|--------|
| `<Space>ff` | Find file (Telescope fuzzy) |
| `<Space>fr` | Recent files |
| `<Space>fg` | Live grep (search text) |
| `<Space>fb` | List open buffers |
| `<Space>e` | Toggle file explorer (neo-tree) |
| `<Space>w` | Save file |
| `<Space>q` | Quit |

### Buffers / windows

| Keymap | Action |
|--------|--------|
| `<Tab>` / `<S-Tab>` | Next/prev buffer |
| `<Space>bd` | Close current buffer |
| `<C-w>v` | Vertical split |
| `<C-w>s` | Horizontal split |
| `<C-h/j/k/l>` | Navigate windows |
| `<C-w>=` | Equalize window sizes |

### LSP

| Keymap | Action |
|--------|--------|
| `gd` | Go to definition |
| `gD` | Go to declaration |
| `gI` | Go to implementation |
| `gr` | Show references |
| `gy` | Go to type definition |
| `K` | Hover documentation |
| `<Space>cf` | Format file |
| `<Space>ca` | Code action (quick fix, refactor) |
| `<Space>cr` | Rename symbol |
| `]d` / `[d` | Next/prev diagnostic |
| `]e` / `[e` | Next/prev error |
| `<Space>cd` | Show diagnostic at cursor |

### Git

| Keymap | Action |
|--------|--------|
| `<Space>gg` | Open lazygit |
| `]h` / `[h` | Next/prev hunk |
| `<Space>ghs` | Stage hunk |
| `<Space>ghr` | Reset hunk |
| `<Space>ghp` | Preview hunk |
| `<Space>gb` | Toggle git blame line |

### Search

| Keymap | Action |
|--------|--------|
| `/` | Search in file |
| `n` / `N` | Next/prev match |
| `*` | Search word under cursor |
| `<Space>sk` | Search keymaps (find what you forgot) |
| `<Space>sh` | Search help |
| `<Space>sw` | Search word in project |
| `<Space>:` | Command history |

### Navigation within file

| Keymap | Action |
|--------|--------|
| `gg` | Top of file |
| `G` | Bottom of file |
| `0` / `$` | Start / end of line |
| `^` | First non-blank of line |
| `Ctrl+u` / `Ctrl+d` | Half-page up/down |
| `Ctrl+f` / `Ctrl+b` | Full-page forward/back |
| `f<char>` | Jump to next occurrence of char on line |
| `s<chars>` | Flash jump to anywhere (LazyVim's flash plugin) |

### Edit basics

| Keymap | Action |
|--------|--------|
| `i` / `a` | Insert before / after cursor |
| `o` / `O` | New line below / above |
| `dd` | Delete line |
| `yy` | Yank (copy) line |
| `p` / `P` | Paste below / above |
| `u` / `<C-r>` | Undo / redo |
| `<C-c>` / `<Esc>` | Exit insert mode |
| `>>` / `<<` | Indent / dedent line |
| `gcc` | Toggle comment line |
| `gc<motion>` | Comment range (e.g. `gcip` paragraph) |

### Plugin management

| Command | Action |
|---------|--------|
| `:Lazy` | Plugin manager UI |
| `:Lazy update` | Update all plugins |
| `:Lazy sync` | Install + update + clean |
| `:LazyExtras` | LazyVim Extras manager |
| `:checkhealth` | Diagnostic |

## Adding a custom plugin

Create `~/.config/nvim/lua/plugins/my-plugin.lua`:

```lua
return {
  {
    "github-user/plugin-name",
    event = "VeryLazy",       -- lazy-load
    config = function()
      require("plugin-name").setup({
        -- options
      })
    end,
    keys = {
      { "<leader>x", "<cmd>SomeCommand<cr>", desc = "Description" },
    },
  },
}
```

LazyVim auto-detects new files in `lua/plugins/`. Restart Neovim → plugin auto-installs.

## Adding a custom keymap

Edit `~/.config/nvim/lua/config/keymaps.lua`:

```lua
local map = vim.keymap.set

map("n", "<leader>tt", "<cmd>terminal<cr>", { desc = "Open terminal" })
map("n", "<leader>cs", function()
  vim.cmd("split")
  vim.cmd("terminal")
end, { desc = "Open terminal in split" })
```

Reload: `:source $MYVIMRC` or restart Neovim.

## Customizing options

Edit `~/.config/nvim/lua/config/options.lua`:

```lua
vim.opt.relativenumber = true     -- relative line numbers
vim.opt.tabstop = 2               -- tab width
vim.opt.shiftwidth = 2            -- indent width
vim.opt.expandtab = true          -- spaces, not tabs
vim.opt.wrap = false              -- no line wrap
vim.opt.scrolloff = 8             -- 8 lines above/below cursor
```

## Common workflows

### Edit a file with autocomplete

```bash
cd /var/www/Apps/some-project
nvim app/Models/User.php
```

- Start typing — autocomplete pops up
- `Tab` to select, `Enter` to confirm
- `K` on a function to see docs
- `gd` to jump to definition

### Search and refactor

```vim
:Telescope live_grep
```

(or `<Space>fg`)

Type pattern, see results, Enter to jump. Use `<C-q>` to send all to quickfix list, then iterate with `:cdo s/old/new/g | update`.

### Multi-file edit with Claude Code

```bash
work                              # tmux: Neovim left, Claude Code right
```

In Claude Code (right pane):
> Refactor app/Models/User.php to use scopes for active/inactive users

Claude reads, modifies, saves. Switch to Neovim (`Ctrl+b ←`), file auto-reloads.

### Git blame / history

```vim
:Gitsigns blame_line             " or <Space>gb
:Gitsigns toggle_current_line_blame
```

Or `<Space>gg` to open lazygit for full history navigation.

## Troubleshooting

### LSP not active in a file

```vim
:LspInfo
```

Shows attached LSP. If empty:
- Filetype recognized? `:set filetype?`
- LSP installed? `:Mason`
- Restart LSP: `:LspRestart`

### Plugin install fails

```vim
:Lazy
```

Check status. Errors show with red. Click on plugin → inspect log. Common cause: network or missing system dep (e.g., gopls needs `go` in PATH).

### Treesitter parser error

```vim
:TSInstall <language>
:TSUpdate
```

Common languages: `php`, `javascript`, `typescript`, `tsx`, `python`, `lua`, `vim`, `markdown`, `bash`.

### Slow startup

```vim
:Lazy profile
```

Shows plugin load times. Lazy-load slow ones (`event = "VeryLazy"`).

### Recover from broken config

Backups are at `~/.config/nvim.backup-YYYYMMDD-HHMMSS`.

To restore old config:
```bash
mv ~/.config/nvim ~/.config/nvim.broken
mv ~/.config/nvim.backup-XXX ~/.config/nvim
rm -rf ~/.local/share/nvim   # plugin data, will re-download
nvim
```

## Resources

- LazyVim docs: https://www.lazyvim.org/
- Neovim docs: `:help` in Neovim, or https://neovim.io/doc/
- Plugin search: https://dotfyle.com/neovim/plugins/trending
- Cheat sheet: `<Space>sk` → search any keymap
