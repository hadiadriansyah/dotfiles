# Reference snapshots

This folder contains snapshots of system state. Regenerate with:

```bash
# .zshrc current
cp ~/.zshrc reference/zshrc-current.txt

# Installed packages
{
  echo "=== mise list ==="
  mise list
  echo ""
  echo "=== apt installed (manually) ==="
  apt-mark showmanual
  echo ""
  echo "=== global npm packages ==="
  npm list -g --depth=0
  echo ""
  echo "=== composer global ==="
  composer global show 2>/dev/null
} > reference/installed-packages.txt

# Docker info
{
  echo "=== Docker version ==="
  docker --version
  docker compose version
  echo ""
  echo "=== Containers ==="
  docker ps -a --format 'table {{.Names}}\t{{.Status}}\t{{.Ports}}'
  echo ""
  echo "=== Images ==="
  docker images
  echo ""
  echo "=== Volumes ==="
  docker volume ls
} > reference/docker-compose-info.txt
```

Run periodically (e.g., monthly) to keep snapshots up to date.

To make these gitignored, uncomment the relevant lines in `.gitignore`. To version them with the repo, leave gitignore as is.
