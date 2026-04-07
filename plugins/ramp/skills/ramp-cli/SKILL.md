---
name: ramp-cli
description: Help with ramp CLI for multi-repo development. Use when working in .ramp/ directories, writing ramp.yaml configs, setup/cleanup scripts, or managing git worktree features.
---

# Ramp CLI

Ramp is a CLI tool for managing multi-repo feature development using git worktrees. It creates isolated feature workspaces with automatic environment setup.

## Quick Reference

```bash
ramp init                     # Initialize new project
ramp install                  # Clone all configured repos
ramp up <feature>             # Create feature workspace
ramp down <feature>           # Cleanup feature
ramp status                   # Show active features and ports
ramp run <cmd> [feature] [-- args]  # Run custom command
ramp refresh                  # Update all source repos
ramp prune                    # Batch cleanup merged features
ramp config                   # Manage local preferences
```

## Project Structure

```
project/
├── .ramp/
│   ├── ramp.yaml              # Main config (committed)
│   ├── local.yaml             # User prefs (gitignored)
│   ├── port_allocations.json  # Port tracking (gitignored)
│   ├── scripts/
│   │   ├── setup.sh           # Runs after `ramp up`
│   │   ├── cleanup.sh         # Runs before `ramp down`
│   │   ├── source/            # Source-scoped commands
│   │   └── feature/           # Feature-scoped commands
│   └── templates/             # Files copied during setup
├── repos/                     # Source repo clones
└── trees/                     # Feature workspaces
    └── <feature>/             # Created by `ramp up`
```

## Key Concepts

- **Feature**: Named development context with branches across all repos, dedicated directory, and allocated ports
- **Worktree**: Git worktree checkout allowing parallel work without branch switching
- **Trees Directory**: Contains feature workspaces (`trees/<feature>/`)
- **Repos Directory**: Authoritative clones of source repositories
- **Port Allocation**: Automatic unique port assignment per feature

## Environment Variables in Scripts

All ramp scripts receive these environment variables:

### Core Variables

| Variable | Description |
|----------|-------------|
| `RAMP_PROJECT_DIR` | Absolute path to project root |
| `RAMP_TREES_DIR` | Path to feature's tree directory |
| `RAMP_WORKTREE_NAME` | Feature name |
| `RAMP_PORT` | First allocated port (alias for RAMP_PORT_1) |
| `RAMP_PORT_1` ... `RAMP_PORT_N` | Allocated ports |
| `RAMP_ARGS` | Space-joined arguments passed after `--` separator |

### Repository Paths

Auto-generated for each repo (uppercase, underscores):
- `RAMP_REPO_PATH_<REPO_NAME>` - Path to repo source clone (in `repos/`)

### User Preferences

Variables from prompts in ramp.yaml (e.g., `RAMP_IDE`, `RAMP_DATABASE`).

## Script Template

```bash
#!/bin/bash

# Exit on error
set -e

# Colors
GREEN='\033[0;32m'
RED='\033[0;31m'
YELLOW='\033[0;33m'
NC='\033[0m'

echo -e "${GREEN}Starting setup for $RAMP_WORKTREE_NAME${NC}"

# Work in the feature worktree
cd "$RAMP_TREES_DIR"

# Access repos in the feature worktree
cd "$RAMP_TREES_DIR/frontend"
bun install

# Source repos are in RAMP_REPO_PATH_* (for reference/diffing)
# e.g., $RAMP_REPO_PATH_FRONTEND points to repos/frontend

# Use allocated ports
echo "API running on port $RAMP_PORT_1"

# Check IDE preference
if [ "$RAMP_IDE" = "vscode" ]; then
    code .
fi
```

## Script Guidelines

1. **Use absolute paths or env vars** - Scripts run from different directories
2. **Check `$RAMP_IDE`** - Respect user's IDE preference
3. **Exit codes matter** - Non-zero in setup.sh triggers rollback
4. **Cleanup runs first** - cleanup.sh has access to files before deletion
5. **Templates for feature files** - Put in `templates/` and copy in setup.sh

## Custom Commands

**IMPORTANT**: Creating a script file is not enough. Commands must be registered in `ramp.yaml` to be discovered by `ramp run`.

### Two Steps Required

1. **Create the script** in `.ramp/scripts/feature/` or `.ramp/scripts/source/`
2. **Register in ramp.yaml** under the `commands:` section

```yaml
# .ramp/ramp.yaml
commands:
  - name: dev           # ramp run dev <feature>
    command: scripts/feature/dev.sh
    scope: feature      # Runs in feature's tree directory

  - name: doctor        # ramp run doctor
    command: scripts/source/doctor.sh
    scope: source       # Runs in project root
```

### Inline Shell Commands

The `command` field accepts either a file path or a shell command. If the value contains spaces, it runs via `bash -l -c` instead of being executed as a file path.

```yaml
commands:
  - name: dev
    command: scripts/feature/dev.sh           # No spaces → file path
    scope: feature

  - name: test
    command: bun test --filter payments       # Spaces → shell command
    scope: feature

  - name: logs
    command: tail -f $RAMP_TREES_DIR/api/logs/app.log
    scope: feature
```

This also works in hooks (see `hooks:` in ramp.yaml).

**Caveat**: File paths containing spaces (e.g., `my scripts/run.sh`) will be misinterpreted as shell commands. Avoid spaces in script paths.

### Passing Arguments

Use `--` to pass arguments to custom commands. Arguments are available as positional params (`$1`, `$2`, `$@`) and via the `RAMP_ARGS` env var.

```bash
ramp run test my-feature -- --filter=payments
# In script: $1="--filter=payments", RAMP_ARGS="--filter=payments"

ramp run check -- --cwd backend --verbose    # source-scoped, no feature needed
# In script: $1="--cwd" $2="backend" $3="--verbose"
#            RAMP_ARGS="--cwd backend --verbose"
```

Note: `RAMP_ARGS` is space-joined. Use positional params (`$1`, `$2`) for values containing spaces.

### Scopes

| Scope | Working Dir | Usage | Example |
|-------|-------------|-------|---------|
| `feature` | `$RAMP_TREES_DIR` | Feature-specific | `ramp run dev myfeature` |
| `source` | `$RAMP_PROJECT_DIR` | Project-wide | `ramp run doctor` |

## Command Flags

```bash
ramp up <feature> --prefix hotfix/     # Override branch prefix
ramp up <feature> --no-refresh         # Skip repo updates
ramp up <feature> --target origin/foo  # Create from existing branch
ramp up --from claude/feature-123      # Create from remote branch
ramp status --json                     # Machine-readable output
ramp status --tree                     # Just output current feature name
```

## Detailed Reference

For complete configuration options, see [reference.md](references/reference.md).
