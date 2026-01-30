# Agent Loop

A system for running Claude agent loops that autonomously work through PRDs (Product Requirements Documents). Supports both local and remote execution via `screen` sessions.

## Prerequisites

- PHP 8.2+
- Node.js / Composer
- `screen` (for background sessions)
- `claude` CLI (Claude Code)
- `Valet` (modify WorktreeSetupCommand.php if using another local server)
- SSH access to `claudebox` (for remote execution)

## Project Structure

```
prd/
├── backlog/              # Active PRDs ready to be worked on
│   └── <project>/
│       ├── project.md    # Feature spec with checkboxed tasks
│       └── progress.md   # Agent progress log
├── complete/             # Finished PRDs
└── _to_refine/           # Drafts created by /prd command

scripts/
├── ralph-loop.js         # Node.js loop runner (used by agent:loop)
├── ralph-loop.sh         # Simple bash loop runner
└── ralph-once.sh         # Single-iteration bash runner

app/Console/Commands/
├── AgentLoop.php         # agent:loop command
├── AgentStatusCommand.php # agent:status command
├── AgentTail.php         # agent:tail command
├── AgentKillCommand.php  # agent:kill command
├── WorktreeSetupCommand.php  # worktree:setup command
└── WorktreeIdeCommand.php    # worktree:ide command
```

## Creating a PRD

Use the `/prd` slash command in Claude Code to generate a new PRD. It will be placed in `prd/_to_refine/`. Once manually refined, move it to `prd/backlog/<project-name>/` with a `project.md` and `progress.md`.

Each `project.md` should contain a feature spec with checkboxed tasks. The agent checks off tasks as it completes them.

## Running Agent Loops

### Local

**Interactive mode** (prompts for project and iterations):

```bash
php artisan agent:loop
```

**With arguments:**

```bash
php artisan agent:loop <project> <iterations>
```

This starts a detached `screen` session running `node scripts/ralph-loop.js`, which iteratively calls the `claude` CLI against the PRD. Default iterations: 30.

**Single iteration** (runs synchronously, no screen session):

```bash
php artisan agent:loop <project> --once
```

**With a git worktree** (isolates changes on a branch):

```bash
php artisan agent:loop <project> --worktree
```

Creates a worktree at `~/www/<repo>-<project>` on the branch `agent/<project>`. Runs `composer install` and `worktree:setup` automatically for new worktrees.

**Attach immediately after starting:**

```bash
php artisan agent:loop <project> --attach
```

### Remote (claudebox)

Syncs the repo to `~/www/<repo>` on the remote host `claudebox`, then starts the agent loop there.
Make sure your SSH keys are set up for passwordless access.

```bash
php artisan agent:loop <project> --remote
php artisan agent:loop <project> --remote --attach
php artisan agent:loop <project> --remote --worktree
```

Remote worktrees are created at `~/www/<repo>-<project>` on `claudebox`.

## Managing Running Agents

Agent sessions are tracked in `.live-agents` at the project root.

**Check status of all agents (local and remote):**

```bash
php artisan agent:status
php artisan agent:status --clean   # Remove completed agents from tracking
```

**Attach to a running session:**

```bash
php artisan agent:tail <session>
php artisan agent:tail <session> --remote
```

Detach from a screen session with `Ctrl+A` then `D`, running `Ctrl+C` will terminate the agent and close the screen session.

**Kill a running agent:**

```bash
php artisan agent:kill              # Interactive selection
php artisan agent:kill <screen>     # Kill by screen name
```

## Screen Session Naming

- Local: `agent-<project>` (or `agent-<project>-wt` with `--worktree`)
- Remote: `agent-<repo>-<project>` (or `agent-<repo>-<project>-wt` with `--worktree`)

## Logs

When running via `ralph-loop.js`, logs are written to:

```
prd/<backlog|complete>/<project>/logs/<timestamp>.log
```

## How the Agent Loop Works

Each iteration the agent:

1. Reads `project.md` and `progress.md` from the PRD
2. Finds the highest-priority incomplete task
3. Implements the task and runs tests
4. Updates `project.md` checkboxes and appends to `progress.md` and `logs/`
5. Commits and pushes changes

If all tasks are complete, the agent outputs `<promise>COMPLETE</promise>` and the loop exits cleanly.

## Shell Scripts (Alternative)

The scripts in `scripts/` can be used directly without artisan:

```bash
./scripts/ralph-once.sh <project>              # Single iteration
./scripts/ralph-loop.sh <project> <iterations>  # Simple loop
node ./scripts/ralph-loop.js <project> <iterations>  # Loop with logging
```

## Loading a PRD

Sometimes it's useful to load a PRD into Claude Code for context.
For this reason there's a /load-prd` skill:

```
claude "/load-prd example"
```
