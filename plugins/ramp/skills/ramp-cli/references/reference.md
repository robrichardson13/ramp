# Ramp CLI Reference

## ramp.yaml Configuration

```yaml
name: my-project

# Repository definitions
repos:
  - path: repos                              # Directory for clones
    git: git@github.com:org/frontend.git     # Git URL
    auto_refresh: true                       # Pull on ramp up (default: true)
    env_files:                               # Environment file templating
      - source: .env.example                 # Template file
        dest: .env                           # Output file
        replace:                             # Variable substitution
          PORT: "${RAMP_PORT_1}"
          API_URL: "http://localhost:${RAMP_PORT_2}"
          APP_NAME: "app-${RAMP_WORKTREE_NAME}"

  - path: repos
    git: git@github.com:org/api.git
    env_files:
      - source: .env
        dest: .env
        replace:
          PORT: "${RAMP_PORT_2}"
          DATABASE_URL: "${RAMP_DATABASE_URL}"

# Interactive prompts (stored in local.yaml)
prompts:
  - name: RAMP_IDE
    question: "Which IDE do you use?"
    options:
      - value: vscode
        label: VS Code (or Cursor, Windsurf, other forks)
      - value: intellij
        label: IntelliJ IDEA
      - value: other
        label: Other
    default: vscode

  - name: RAMP_DATABASE
    question: "Which database?"
    options:
      - value: postgres
        label: PostgreSQL
      - value: mysql
        label: MySQL
    default: postgres

# Branch and port settings
default-branch-prefix: feature/    # Prefix for new branches
base_port: 3000                    # Starting port number
max_ports: 100                     # Total port range
ports_per_feature: 4               # Ports allocated per feature

# Lifecycle scripts
setup: scripts/setup.sh            # After ramp up
cleanup: scripts/cleanup.sh        # Before ramp down

# Custom commands
# command field: no spaces → file path, spaces → shell command (bash -l -c)
commands:
  - name: dev
    command: scripts/feature/dev.sh    # File path (no spaces)
    scope: feature                     # Requires feature: ramp run dev <feature>

  - name: doctor
    command: scripts/source/doctor.sh  # File path
    scope: source                      # No feature: ramp run doctor

  - name: test
    command: bun test                  # Inline shell command (has spaces)
    scope: feature

  - name: deploy
    command: scripts/source/deploy.sh
    scope: source

  - name: logs
    command: tail -f $RAMP_TREES_DIR/api/logs/app.log  # Inline shell
    scope: feature

# Lifecycle hooks (command field supports file paths and inline shell)
hooks:
  - event: up                      # After feature creation
    command: scripts/notify.sh

  - event: up
    command: echo "Feature created!"  # Inline shell command

  - event: down                    # Before feature deletion
    command: scripts/backup.sh

  - event: run                     # After custom commands
    command: scripts/log.sh
    for: deploy                    # Only for specific command

  - event: run
    command: scripts/cleanup-tests.sh
    for: test-*                    # Wildcard matching
```

## local.yaml (User Preferences)

Gitignored file storing personal settings:

```yaml
# Prompt responses (auto-populated by ramp config)
prompts:
  RAMP_IDE: vscode
  RAMP_DATABASE: postgres

# Local-only commands
commands:
  - name: my-debug
    command: scripts/local/debug.sh
    scope: feature

# Local-only hooks
hooks:
  - event: up
    command: echo "Feature created!"
```

## Command Detection Heuristic

The `command` field uses a simple rule: if the value contains **no spaces**, it's treated as a file path and executed directly. If it contains **spaces**, it's run as a shell command via `bash -l -c`. File paths with spaces in their names will be misinterpreted — avoid spaces in script paths.

## Passing Arguments to Commands

Use `--` after the command name to pass arguments to scripts:

```bash
ramp run test my-feature -- --filter=payments --verbose
```

Arguments are available in scripts as:
- **Positional params**: `$1`, `$2`, `$@` (preserves quoting)
- **`RAMP_ARGS` env var**: space-joined string of all arguments

Use positional params for values that may contain spaces.

## Command Scopes

| Scope | Working Directory | Usage |
|-------|-------------------|-------|
| `source` | `$RAMP_PROJECT_DIR` | Project-wide commands (doctor, deploy) |
| `feature` | `$RAMP_TREES_DIR` | Feature-specific commands (dev, test) |

## Port Allocation

Ports are allocated sequentially from `base_port`:
- Feature 1: `base_port` to `base_port + ports_per_feature - 1`
- Feature 2: `base_port + ports_per_feature` to `base_port + 2*ports_per_feature - 1`
- Ports are released and reused after `ramp down`

## Environment Variable Naming

Repository source paths are auto-generated (point to clones in `repos/`):
- `frontend` → `RAMP_REPO_PATH_FRONTEND`
- `api-server` → `RAMP_REPO_PATH_API_SERVER`
- `my.repo` → `RAMP_REPO_PATH_MY_REPO`

Rules: uppercase, replace `-` and `.` with `_`

To access repos in a feature worktree, use `$RAMP_TREES_DIR/<repo-name>` instead.

## Hook Events

| Event | When | Context |
|-------|------|---------|
| `up` | After setup.sh completes | Feature fully created |
| `down` | Before cleanup.sh runs | Feature still exists |
| `run` | After command completes | Use `for:` to filter |

## Env File Templating

Available variables in `replace`:
- `${RAMP_PORT}`, `${RAMP_PORT_1}`, etc.
- `${RAMP_WORKTREE_NAME}`
- `${RAMP_PROJECT_DIR}`, `${RAMP_TREES_DIR}`
- Any prompt variable: `${RAMP_IDE}`, `${RAMP_DATABASE}`
- Any repo source path: `${RAMP_REPO_PATH_FRONTEND}`

## Configuration Hierarchy

1. **Project** (`.ramp/ramp.yaml`) - Shared, committed
2. **Local** (`.ramp/local.yaml`) - Personal, gitignored
3. **User** (`~/.config/ramp/ramp.yaml`) - Global, all projects

Execution: All hooks run (project → local → user). Commands: first match wins.

## Common Patterns

### Setup Script Pattern

```bash
#!/bin/bash
set -e

cd "$RAMP_TREES_DIR"

# Copy templates
cp -r "$RAMP_PROJECT_DIR/.ramp/templates/." .

# Install dependencies in parallel
(cd "$RAMP_TREES_DIR/frontend" && bun install) &
(cd "$RAMP_TREES_DIR/api" && bun install) &
wait

# Generate Prisma client
cd "$RAMP_TREES_DIR/api"
bunx prisma generate

# Open IDE
case "$RAMP_IDE" in
    vscode) code "$RAMP_TREES_DIR" ;;
    intellij) idea "$RAMP_TREES_DIR" ;;
esac

echo "Feature $RAMP_WORKTREE_NAME ready on ports $RAMP_PORT_1-$RAMP_PORT_4"
```

### Cleanup Script Pattern

```bash
#!/bin/bash

cd "$RAMP_TREES_DIR"

# Stop any running services
pkill -f "node.*$RAMP_PORT_1" 2>/dev/null || true
pkill -f "node.*$RAMP_PORT_2" 2>/dev/null || true

# Stop docker containers if any
docker compose down 2>/dev/null || true

echo "Cleaned up $RAMP_WORKTREE_NAME"
```

### Dev Command Pattern

```bash
#!/bin/bash
set -e

cd "$RAMP_TREES_DIR"

# Start services with allocated ports
(cd "$RAMP_TREES_DIR/api" && PORT=$RAMP_PORT_1 bun run dev) &
(cd "$RAMP_TREES_DIR/frontend" && PORT=$RAMP_PORT_2 bun run dev) &

wait
```

## Troubleshooting

### Port Already in Use
```bash
ramp status                    # Check which feature has the port
ramp down <conflicting-feature>
ramp up <your-feature>
```

### Worktree Locked
```bash
cd repos/<repo>
git worktree list              # Find orphaned worktrees
git worktree prune             # Clean up
```

### Reset Port Allocations
```bash
rm .ramp/port_allocations.json
ramp up <feature>              # Fresh allocation
```
