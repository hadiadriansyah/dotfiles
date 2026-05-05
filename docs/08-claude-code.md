# 08 — Claude Code + tmux Workflow

## What is Claude Code

Claude Code is an agentic coding CLI from Anthropic. Unlike chat assistants, it can:
- Read multiple files in your project
- Write/modify files
- Run shell commands (with permission)
- Multi-step task execution

It runs in the terminal, making it perfect for Neovim/tmux workflows.

## Setup

- **Binary**: installed via `npm install -g @anthropic-ai/claude-code`
- **Location**: managed by mise (in current Node 22 install)
- **Authentication**: Claude.ai Pro subscription (NOT API key)

### Login (one-time)

```bash
claude
```

First launch prompts for authentication. Choose "Log in with Claude.ai", complete OAuth in browser.

### Important: don't set `ANTHROPIC_API_KEY`

If `ANTHROPIC_API_KEY` env var is set, Claude Code uses pay-per-token API instead of Pro subscription. Verify:

```bash
echo "${ANTHROPIC_API_KEY:-not set}"
# Should output: "not set"
```

If set, unset:
```bash
unset ANTHROPIC_API_KEY
# And remove from .zshrc / .profile if exported there
```

## Pro subscription limits

Pro plan shares usage between Claude.ai chat and Claude Code:
- Approximately 10-40 prompts per 5-hour rolling window for Claude Code
- Plus weekly cap

Strategy:
- Use **Sonnet 4.6** (default) for most coding tasks
- Use **Haiku 4.5** for simple lookups (90% capability, ~3x cheaper)
- Reserve **Opus 4.7** for complex refactoring, architecture decisions

## Models

### Switch model in session

```
/model              # interactive picker
/model opus         # switch to Opus
/model sonnet       # switch to Sonnet (default)
/model haiku        # switch to Haiku
```

⚠️ Mid-conversation switch warns because next response re-reads full history (uncached cost).

### Switch model on launch

```bash
claude --model claude-sonnet-4-6
claude --model claude-haiku-4-5
claude --model claude-opus-4-7
```

### Check current

```
/status
```

## tmux workflow

This system uses tmux to split terminal: Neovim left, Claude Code right.

### `work` function (in `.zshrc`)

```bash
cd /var/www/Apps/some-project
work
```

This:
1. Creates a tmux session named "work" in current folder
2. Left pane (70%): runs `nvim`
3. Right pane (30%): runs `claude`
4. Attaches to the session

To use a custom session name:
```bash
work my-project
```

To list / kill sessions:
```bash
tls                # tmux ls (alias)
tat work           # tmux attach -t work (alias)
tkill              # kill session "work"
tkill my-project   # kill specific session
```

### tmux navigation reference

| Combo | Action |
|-------|--------|
| `Ctrl+b ←/→/↑/↓` | Switch pane |
| `Ctrl+b z` | Zoom pane (toggle full screen) |
| `Ctrl+b d` | Detach (session keeps running in background) |
| `Ctrl+b Ctrl+→` | Resize pane (right) |
| `Ctrl+b Ctrl+←` | Resize pane (left) |
| `Ctrl+b [` | Scroll mode (q to exit) |
| `Ctrl+b x` | Kill pane |
| `Ctrl+b %` | Manual split vertical |
| `Ctrl+b "` | Manual split horizontal |

### Visual layout

```
┌─────────────────────────────────────┬───────────────────────┐
│                                     │                       │
│  Neovim                             │  Claude Code          │
│  (70%)                              │  (30%)                │
│                                     │                       │
│  app/Http/Controllers/UserCtrl.php  │  > Refactor this      │
│   1  <?php                          │    controller to use  │
│   2  namespace App\...              │    middleware for     │
│   3                                 │    auth checks        │
│   4  class UserController {         │                       │
│   5    function login(Request $r) { │  Reading file...      │
│   ...                               │  Modifying...         │
│                                     │  Done.                │
│                                     │                       │
└─────────────────────────────────────┴───────────────────────┘
```

## Inside Claude Code

### Slash commands

```
/help              show help
/status            current model + session info
/model [name]      change model
/clear             clear conversation history
/cost              show usage stats
/exit              exit Claude Code
```

### Effective prompting

#### Be specific

❌ "Fix the bug"
✅ "There's a bug in app/Models/User.php — the `isActive()` method returns true for soft-deleted users. Fix it."

❌ "Add tests"
✅ "Add PHPUnit tests for app/Services/AuthService.php covering: valid login, invalid password, locked account, 2FA flow."

#### Provide context

❌ "Add a button"
✅ "In resources/views/dashboard.blade.php, add a 'Export CSV' button next to the existing 'Refresh' button. Style it with the same Tailwind classes (btn-secondary). On click, hit the route `/dashboard/export`."

#### Specify constraints

✅ "Refactor app/Repositories/UserRepository.php to use query scopes. Don't change the public API of the class — existing callers should still work."

### Common workflows

#### Multi-file refactor

```
> Refactor app/Models/User.php to use Eloquent scopes for active/inactive
> users. Update all callers in app/Http/Controllers/ to use the new scopes.
> Run `php artisan test` afterward to verify nothing broke.
```

#### Generate tests

```
> Generate Pest tests for app/Services/PaymentService.php. Cover happy paths
> and error cases (insufficient funds, invalid card, network error).
```

#### Debug

```
> When I run `php artisan migrate`, I get this error:
> [paste error]
> Help me figure out what's wrong. The migration files are in database/migrations/.
```

#### Explain

```
> Read app/Http/Middleware/Authenticate.php and explain how it works to me.
> I'm new to Laravel middleware.
```

#### Generate boilerplate

```
> Create a new Laravel resource controller for managing Products.
> Include: index, show, store, update, destroy.
> Use route model binding.
> Add request validation classes.
```

### File path conventions

When mentioning files, use paths relative to the Claude Code working directory:

```
> Edit app/Models/User.php           # relative
> Edit /var/www/Apps/.../User.php    # absolute also works
```

Claude Code starts from the directory where you ran `claude`. So:

```bash
cd /var/www/Apps/some-project
claude
> # Now "app/Models/..." resolves to /var/www/Apps/some-project/app/Models/...
```

## Permission model

Claude Code asks before:
- Modifying files outside the working directory
- Running commands (especially destructive ones like `rm`, `git push --force`)
- Network requests

You can grant permissions per-action or session.

```
/permissions       # review/manage permissions
```

## Integration tips

### Auto-reload Neovim after Claude edits

Neovim auto-reloads files modified externally if `:set autoread` is on (LazyVim default).

If not auto-reloading:
```vim
:e!         " force reload current file
:checktime  " check all open buffers
```

### Compare Claude's changes

Before accepting:

```bash
# In a separate terminal:
git diff
```

Or in Neovim with gitsigns: hunks show in gutter.

### Undo Claude's changes

If you don't like what Claude did:

```bash
git checkout app/Models/User.php       # undo single file
git stash                              # stash all changes
git reset --hard HEAD                  # nuke ALL uncommitted changes (CAREFUL)
```

Best practice: commit before letting Claude make big changes. Then `git diff HEAD~1` to see what changed.

## Workflow examples

### Daily fullstack dev

```bash
cd /var/www/Apps/my-laravel-app
docker compose up -d      # if needed
work                       # tmux split
```

In Neovim (left): edit `app/Models/User.php`
In Claude Code (right): "Add a `lockAccount()` method that locks the user after 3 failed logins. Update the login controller to call it."

Switch to Neovim (`Ctrl+b ←`), see Claude's changes, refine manually if needed.

### Code review before commit

In Claude Code:
```
> Review the staged changes (git diff --cached) and tell me if there are
> any bugs, security issues, or anti-patterns.
```

### Generate documentation

```
> Generate JSDoc comments for src/utils/auth.ts based on the existing implementation.
```

### Help with terminal commands

```
> I want to find all PHP files modified in the last 7 days that contain "deprecated".
> Give me the bash command.
```

## Cost optimization

### Commit early, commit often

Smaller change = smaller context = lower cost + clearer reviews.

### Use Haiku for simple tasks

```
/model haiku
```

Then for "rename this variable everywhere", "fix this typo", "format this file" → Haiku is fine.

Switch back to Sonnet for complex work.

### Clear context periodically

```
/clear
```

Each turn re-sends full conversation history. Long sessions = expensive turns. Clear when starting a new task.

### Use ultrathink for hard problems

Add `ultrathink` anywhere in prompt:

```
> ultrathink: design a caching strategy for this API. Consider hit rate, invalidation, and memory limits.
```

This requests deeper reasoning for that turn only.

## Resources

- Docs: https://docs.claude.com/claude-code/
- Pricing: https://www.anthropic.com/pricing
- Status: https://status.claude.com/
