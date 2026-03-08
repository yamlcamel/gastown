# Running Factory Droid with Gas Town

A complete guide to orchestrating multiple Droid agents with Gas Town — from
installation to multi-agent fleets.

## What Is This?

**Gas Town** is a multi-agent workspace manager. It orchestrates coding agents
through tmux sessions, assigns work from an integrated issue tracker, manages
merge queues, and monitors agent health — all without importing agent libraries
or requiring agents to know about Gas Town internals.

**Factory Droid** is a model-agnostic terminal-native AI coding agent. It supports
multiple LLM providers (Claude, GPT, Gemini, and others), has a native hooks
system, non-interactive execution mode, and a plugin ecosystem.

**Together**, Gas Town provides the orchestration layer (who works on what, when,
and how work gets merged) while Droid provides the execution layer (the actual
coding agent running in each tmux pane).

### Why Use Gas Town with Droid?

Running a single Droid instance is fine for individual tasks. But when you have
a backlog of 20 issues, Gas Town lets you:

- Spawn 5 Droid polecats working in parallel on separate issues
- Track all 20 issues in a convoy with automatic progress updates
- Auto-merge completed work through a tested merge queue
- Detect stuck agents and recover automatically
- Mix models — GPT-5.2 for complex design, Haiku for monitoring
- Decompose specs into dependency-aware task graphs that execute in waves

---

## Prerequisites

Install these before continuing:

| Tool | Install | Verify |
|------|---------|--------|
| **Gas Town** | `brew install gt` or `go install github.com/steveyegge/gastown/cmd/gt@latest` | `gt --version` |
| **Factory Droid** | `brew install --cask droid` or `curl -fsSL https://app.factory.ai/cli \| sh` | `droid --version` |
| **tmux** | `brew install tmux` | `tmux -V` |
| **Dolt** | `brew install dolt` | `dolt version` |
| **Git** | Pre-installed on macOS | `git --version` |

Droid requires a Factory account (free). Run `droid` once to authenticate.

---

## Setup

### 1. Initialize a Gas Town

A "town" is the top-level workspace. It lives at `~/gt/` by default.

```bash
gt init
```

This creates:
- `~/gt/settings/` — town-wide configuration
- `~/gt/.beads/` — town-level issue tracker (convoys, coordination)
- `~/gt/mayor/` — mayor agent workspace
- `~/gt/deacon/` — system health monitor

### 2. Activate Droid as the default agent

Gas Town ships with a built-in Droid preset. Set it as the town default:

```bash
gt config set default_agent droid
```

Or edit `~/gt/settings/config.json` directly:

```json
{
  "type": "town-settings",
  "version": 1,
  "default_agent": "droid"
}
```

### 3. Create a rig

A "rig" is a managed project — a git repository that Gas Town orchestrates agents in.

```bash
gt rig add myproject --repo https://github.com/you/myproject.git
```

This clones the repo into `~/gt/myproject/` and sets up:
- A witness agent (monitors polecat health)
- A refinery agent (merge queue processor)
- Beads issue tracking for the project

### 4. Verify the preset

```bash
gt config get agent --rig myproject
# Should output: droid
```

---

## Your First Droid Agent

Start a crew session to verify everything works:

```bash
gt crew at jack --rig myproject
```

This creates a persistent Droid session in a tmux pane. You should see Droid's
TUI start up. Gas Town's `SessionStart` hook fires automatically, running:

```
gt prime --hook && gt mail check --inject
```

This injects your role context (crew member "jack" in rig "myproject") and any
pending mail into the Droid session.

### Verify hooks are working

In the Droid session, check that Gas Town context was injected. The agent should
know its role. You can also verify manually:

```bash
gt prime
```

This outputs the role documentation, pending work, and system instructions that
Droid received on startup.

### Navigate tmux sessions

Gas Town creates tmux sessions for each agent:

- `Ctrl-b n` / `Ctrl-b p` — cycle between sessions in the same group
- `Ctrl-b s` — session switcher
- `Ctrl-b d` — detach (sessions keep running)

---

## Planning Work with Beads

Beads is Gas Town's built-in issue tracker. Issues ("beads") live in `.beads/`
directories and are managed with the `bd` command.

### Creating issues

```bash
# Basic task
bd create --title "Add user authentication" --type task

# Bug report
bd create --title "Login fails with special characters" --type bug

# Feature request
bd create --title "OAuth2 support" --type feature

# Epic (container for related tasks)
bd create --title "Auth system overhaul" --type epic
```

### Issue types

| Type | Purpose |
|------|---------|
| `epic` | Container for related tasks — never directly assigned to a polecat |
| `task` | Concrete unit of work — assignable to polecats |
| `bug` | Defect report — assignable |
| `feature` | Feature request — assignable |
| `chore` | Maintenance work — assignable |
| `decision` | Design decision record |

### Dependencies

Dependencies control execution order. There are two kinds:

**Execution-blocking** (prevents work from starting):
```bash
bd dep add <child> <blocker> --type blocks      # child waits for blocker
bd dep add <child> <blocker> --type waits-for   # soft blocking
```

**Organizational** (grouping only, never blocks):
```bash
bd dep add <child> <parent> --type parent-child  # hierarchy
bd dep add <a> <b> --type related                 # cross-reference
```

### Querying work

```bash
bd ready                    # Issues with zero blockers (ready to work)
bd ready --json             # Machine-readable
bd blocked                  # Issues waiting on dependencies
bd list --status open       # All open issues
bd list --type task         # Filter by type
bd show <id>                # Full issue details
```

### Example: building a task graph

```bash
# Create epic
bd create --title "Auth system" --type epic
# => mp-auth-001

# Create tasks under the epic
bd create --title "Design auth schema" --type task
# => mp-auth-002
bd create --title "Implement JWT middleware" --type task
# => mp-auth-003
bd create --title "Add login endpoint" --type task
# => mp-auth-004
bd create --title "Write auth tests" --type task
# => mp-auth-005

# Wire up hierarchy
bd dep add mp-auth-002 mp-auth-001 --type parent-child
bd dep add mp-auth-003 mp-auth-001 --type parent-child
bd dep add mp-auth-004 mp-auth-001 --type parent-child
bd dep add mp-auth-005 mp-auth-001 --type parent-child

# Wire up execution order
bd dep add mp-auth-003 mp-auth-002 --type blocks   # JWT needs schema first
bd dep add mp-auth-004 mp-auth-003 --type blocks   # Login needs JWT
bd dep add mp-auth-005 mp-auth-004 --type blocks   # Tests need login

# Check what's ready
bd ready
# => mp-auth-002 (Design auth schema) — the only unblocked task
```

---

## Decomposing Specs into Tasks

For larger features, Gas Town can decompose a spec into tasks automatically using
the `mol-idea-to-plan` formula. This runs a multi-agent pipeline:

```bash
gt formula run mol-idea-to-plan --problem "Add OAuth2 authentication with Google and GitHub providers"
```

### The pipeline

```
1. Intake agent         → Structures the idea into a draft PRD
2. PRD Review convoy    → 6 Droid polecats review in parallel:
   (requirements, gaps, ambiguity, feasibility, scope, stakeholders)
3. Human gate           → You answer critical questions from the review
4. Design convoy        → 6 Droid polecats design in parallel:
   (API, data model, UX, scalability, security, integration)
5. Plan Review convoy   → 5 Droid polecats validate in parallel:
   (completeness, sequencing, risk, scope-creep, testability)
6. Human gate           → You approve or reject the plan
7. Create beads         → Agent converts approved plan into beads with deps
```

There are only **two human gates** — everything else runs autonomously. The result
is a fully wired dependency graph of beads ready for dispatch.

### Running with a specific model

```bash
gt formula run mol-idea-to-plan \
  --problem "Add OAuth2 support" \
  --agent droid-gpt  # Use GPT-5.2 for planning
```

---

## Wave-Based Dispatch

For epics with complex dependency graphs, Gas Town can compute execution waves
and dispatch them automatically:

```bash
# Stage the epic (analyze deps, build DAG, compute waves)
gt convoy stage mp-auth-001

# Preview the wave plan
gt convoy stage mp-auth-001 --json
# {
#   "waves": [
#     {"wave": 1, "issues": ["mp-auth-002"]},
#     {"wave": 2, "issues": ["mp-auth-003"]},
#     {"wave": 3, "issues": ["mp-auth-004"]},
#     {"wave": 4, "issues": ["mp-auth-005"]}
#   ]
# }

# Launch — dispatches Wave 1 immediately
gt convoy launch mp-auth-001
```

Gas Town uses Kahn's algorithm to compute waves from the dependency DAG. When
Wave 1 issues complete, Wave 2 is automatically dispatched, and so on.

---

## Polecats — Ephemeral Workers

Polecats are the workhorses — ephemeral Droid sessions that pick up a single
issue, complete it, and are destroyed after merge.

### Assigning work

```bash
# Assign a single issue to a rig (auto-spawns a polecat)
gt sling mp-auth-002 myproject

# Assign multiple issues (each gets its own polecat)
gt sling mp-auth-002 mp-auth-003 mp-auth-004 myproject

# Assign to a specific named polecat
gt sling mp-auth-002 myproject/furiosa
```

### What happens when you sling

1. Gas Town spawns a fresh Droid polecat in a tmux session
2. The issue is "hooked" to the polecat (atomic assignment)
3. Droid's `SessionStart` hook fires, running `gt prime --hook`
4. `gt prime` outputs the polecat's role context and work assignment
5. The polecat works the issue autonomously

### The propulsion principle

> **If you find something on your hook, YOU RUN IT.**

There's no polling. Work is pushed to polecats via `gt sling`, and execution is
immediate. The hook IS the assignment.

### Polecat lifecycle

```
Spawned → Hooked (work assigned) → Working → gt done → Idle
                                                ↓
                                         Witness detects
                                                ↓
                                         Refinery merges
                                                ↓
                                         Worktree cleaned
                                                ↓
                                         Available for reuse
```

### Orientation commands (run by agents)

```bash
gt hook              # What's on my hook?
gt prime             # Show full role context and formula checklist
gt done              # Submit completed work (push branch, create MR)
gt done --status DEFERRED  # Hand off incomplete work
```

---

## Convoys — Batch Work Tracking

A convoy tracks a group of related issues across rigs and agents:

```bash
# Create a convoy
gt convoy create "Auth system v2" mp-auth-002 mp-auth-003 mp-auth-004

# Check progress
gt convoy status hq-cv-abc
# Auth system v2: 1/3 complete
#   ✓ mp-auth-002  Design auth schema
#   → mp-auth-003  Implement JWT middleware (in progress)
#   ○ mp-auth-004  Add login endpoint (blocked)

# List all convoys
gt convoy list
```

### Auto-convoy

When you sling a single issue, Gas Town auto-creates a convoy:

```bash
gt sling mp-auth-002 myproject
# Creates: "Work: Design auth schema" convoy tracking mp-auth-002
```

### Stranded detection

The daemon scans for "stranded" convoys (ready work with no polecats) every 30
seconds and auto-dispatches:

```bash
gt convoy stranded          # Show stranded convoys
gt convoy stranded --json   # Machine-readable
```

### Convoy lifecycle

```
OPEN → (all issues close) → LANDED
  ↑                            |
  └── (add more issues) ──────┘
```

When a convoy lands, subscribers are notified.

---

## Formulas — Automated Multi-Agent Workflows

Formulas are structured workflows defined in TOML that execute via Droid's
non-interactive mode (`droid exec`).

### Formula types

| Type | Pattern | Use Case |
|------|---------|----------|
| **Convoy** | Parallel legs + synthesis | Security audit, code review |
| **Workflow** | Sequential with `needs = [...]` | Release pipeline |
| **Expansion** | Template-based generation | Run review on N packages |
| **Aspect** | Multi-aspect parallel analysis | Architecture review |

### Running a formula

```bash
# Run a code review with parallel aspects
gt formula run code-review --pr 123

# Run with a specific agent
gt formula run code-review --agent droid

# Run the full spec-to-beads pipeline
gt formula run mol-idea-to-plan --problem "Add caching layer"
```

### How Droid exec integrates

Gas Town builds non-interactive commands from the preset's `NonInteractiveConfig`:

```bash
droid exec "Analyze the authentication module for security vulnerabilities" --output-format json --auto high
```

Each formula leg becomes a separate `droid exec` invocation. Parallel legs run
in separate tmux panes simultaneously.

### Formula resolution order

1. **Project**: `<project>/.beads/formulas/` — project-specific
2. **Town**: `~/gt/.beads/formulas/` — user customizations
3. **System**: embedded in `gt` binary — factory defaults

---

## Mixed Agent Fleets

Gas Town's `role_agents` config lets you run different agents per role:

```json
{
  "type": "town-settings",
  "version": 1,
  "default_agent": "droid",
  "role_agents": {
    "mayor": "claude",
    "witness": "droid-haiku",
    "polecat": "droid",
    "crew": "droid",
    "refinery": "droid-haiku",
    "deacon": "droid-haiku"
  }
}
```

### Registering model variants

Add multiple Droid presets with different models in `~/gt/settings/agents.json`:

```json
{
  "version": 1,
  "agents": {
    "droid-haiku": {
      "name": "droid-haiku",
      "command": "droid",
      "args": ["--skip-permissions-unsafe", "-m", "claude-haiku-4-5-20251001"],
      "process_names": ["droid"],
      "resume_flag": "--resume",
      "resume_style": "flag",
      "supports_hooks": true,
      "prompt_mode": "arg",
      "config_dir": ".factory",
      "hooks_provider": "droid",
      "hooks_dir": ".factory",
      "hooks_settings_file": "settings.json",
      "ready_delay_ms": 5000,
      "instructions_file": "AGENTS.md",
      "non_interactive": {
        "subcommand": "exec",
        "output_flag": "--output-format json"
      }
    },
    "droid-gpt": {
      "name": "droid-gpt",
      "command": "droid",
      "args": ["--skip-permissions-unsafe", "-m", "gpt-5.2"],
      "process_names": ["droid"],
      "resume_flag": "--resume",
      "resume_style": "flag",
      "supports_hooks": true,
      "prompt_mode": "arg",
      "config_dir": ".factory",
      "hooks_provider": "droid",
      "hooks_dir": ".factory",
      "hooks_settings_file": "settings.json",
      "ready_delay_ms": 5000,
      "instructions_file": "AGENTS.md",
      "non_interactive": {
        "subcommand": "exec",
        "output_flag": "--output-format json"
      }
    }
  }
}
```

### Cost optimization strategies

| Role | Recommended Model | Reasoning |
|------|-------------------|-----------|
| Mayor | Claude Opus | Complex cross-rig coordination |
| Witness | Haiku | Lightweight monitoring, low token cost |
| Polecat | Opus or GPT-5.2 | Core coding work, needs strong reasoning |
| Crew | Opus | Interactive, needs full capability |
| Refinery | Haiku | Merge queue management, mostly mechanical |
| Deacon | Haiku | Health checks, low complexity |

### Per-rig agent overrides

Override the agent for a specific rig:

```bash
gt config set agent droid-gpt --rig myproject
```

Or in `~/gt/myproject/settings/config.json`:

```json
{
  "type": "rig-settings",
  "version": 1,
  "agent": "droid-gpt",
  "role_agents": {
    "witness": "droid-haiku"
  }
}
```

---

## Scheduler — Capacity-Governed Dispatch

Prevent spawning too many polecats at once:

```bash
gt config set scheduler.max_polecats 5
```

With this set:
- `gt sling` creates scheduling entries instead of dispatching immediately
- The daemon dispatches up to 5 polecats at a time
- As polecats complete, queued work backfills automatically

### Scheduler commands

```bash
gt scheduler status          # Queue depth and active count
gt scheduler list            # Queued beads
gt scheduler run             # Manual dispatch trigger
gt scheduler pause           # Pause dispatch
gt scheduler resume          # Resume dispatch
gt scheduler clear           # Remove beads from queue
```

### Circuit breaker

After 3 consecutive dispatch failures for a bead, it's automatically removed
from the queue to prevent retry loops.

---

## Handoff & Context Cycling

Long-running sessions accumulate context debt. Gas Town handles this through
handoffs — graceful session replacement with context preservation.

### Manual handoff

```bash
gt handoff -s "Completed auth refactor, tests green, docs next"
```

This:
1. Creates mail for the next session with your context summary
2. Ends the current session (`gt done --status DEFERRED`)
3. Daemon respawns a fresh Droid session
4. `SessionStart` hook injects the handoff mail

### Automatic handoff

Gas Town's `PreCompact` hook fires before Droid compresses its context window.
This re-injects the full role context so nothing is lost during compaction.

When context pressure reaches ~70%, agents should run `gt handoff` to cycle to
a fresh session. The witness monitors for this and nudges stuck agents.

### Session resume

Droid supports `--resume [sessionId]` for continuing previous sessions:

```bash
droid --resume abc123
```

Gas Town uses this automatically when restarting polecat sessions that were
interrupted (crash recovery, not handoff).

---

## Dashboard & Monitoring

### The TUI dashboard

```bash
gt dash              # Full dashboard
gt dash -p           # Problems-first view
```

Three panels:
- **Agent tree** — all agents by role with status indicators
- **Convoy tracker** — progress bars for active convoys
- **Event stream** — real-time feed of all system activity

### Status indicators

| Symbol | Meaning |
|--------|---------|
| `●` | Working (hooked, active) |
| `○` | Idle (no hooked work) |
| `🔥` | Stuck (GUPP violation — hooked work, no progress for 30min) |

### Event symbols

| Symbol | Event |
|--------|-------|
| `+` | Created/bonded |
| `→` | In progress |
| `✓` | Completed |
| `✗` | Failed |
| `⚡` | Agent nudged |
| `🎯` | Work slung |
| `🤝` | Handoff |
| `🦉` | Patrol started |

### Keyboard shortcuts

- `Enter` — attach to selected agent's tmux session
- `n` — nudge selected agent
- `h` — trigger handoff for selected agent
- `p` — toggle problems view

### Activity feed

```bash
gt feed              # Live event stream (CLI)
gt feed --json       # Machine-readable
```

Events are curated with deduplication and aggregation — 5 molecule updates
become "agent active", 3 slings in 30s become "batch dispatch".

---

## Health Monitoring

Gas Town runs a three-tier watchdog chain:

```
Daemon (Go, mechanical, every 30s)
  └─ Boot (ephemeral AI, triage: "should Deacon wake?")
      └─ Deacon (persistent AI, continuous patrol)
          └─ Witness (per-rig, monitors polecats)
```

### GUPP violations

A GUPP (Hooked Work + No Progress) violation means an agent has assigned work
but hasn't made progress in 30 minutes. The witness:

1. Nudges the agent
2. Sends health checks (3 attempts, 30s timeout each)
3. Force-kills and redispatches to a fresh polecat if unresponsive

### Stale hook cleanup

```bash
gt deacon stale-hooks                # Find abandoned hooks
gt deacon stale-hooks --dry-run      # Preview only
gt deacon stale-hooks --max-age 30m  # Custom threshold
```

### Heartbeat freshness

| Age | Status | Action |
|-----|--------|--------|
| < 5 min | Fresh | None |
| 5-15 min | Stale | Nudge if pending mail |
| > 15 min | Very stale | Wake Deacon |

---

## Integration Branches & Merge Queue

### The Refinery

The refinery is Gas Town's merge queue processor. When a polecat completes work
(`gt done`), the witness sends `MERGE_READY` to the refinery, which:

1. Batches pending MRs (Bors-style)
2. Tests the tip of the batch
3. Merges to main if tests pass
4. Bisects on failure to find the broken MR

### Integration branches for epics

For large epics, polecats merge to a shared integration branch instead of main:

```bash
# Create integration branch for the epic
gt mq integration create mp-auth-001

# Polecats merge to the integration branch
# (configured automatically when convoy is launched)

# When all epic work is done, land to main
gt mq integration land mp-auth-001
```

---

## Slash Commands

Gas Town provisions slash commands into Droid's config directory. These appear
as `/command` in Droid's interactive mode:

| Command | Action |
|---------|--------|
| `/handoff` | Hand off to fresh session with context mail |

Commands are installed at `.factory/commands/handoff.md` and follow Droid's
custom slash command format (markdown with YAML frontmatter).

---

## Droid-Specific Features

These Droid capabilities can enhance Gas Town workflows:

### Mission mode

Droid's `--mission` flag enables its own multi-agent orchestration within a
single polecat session:

```bash
droid exec --mission --skip-permissions-unsafe "Build the full authentication system"
```

This spawns worker droids via the Factory daemon. Useful for complex single-issue
work where one polecat needs sub-agents.

### Custom droids (subagents)

Define specialized sub-agents in `.factory/droids/`:

```markdown
---
name: security-reviewer
description: Reviews code for security vulnerabilities
model: claude-opus-4-6
tools: read-only
---

You are a security review specialist. Analyze code for OWASP Top 10
vulnerabilities, injection risks, and authentication weaknesses.
```

Gas Town polecats can delegate to these custom droids for specialized sub-tasks.

### Autonomy tiers

Droid's `--auto` flag provides finer-grained control than binary YOLO mode:

| Level | Allows | Use Case |
|-------|--------|----------|
| (none) | Read-only | Witness monitoring, analysis |
| `--auto low` | File edits | Documentation, formatting |
| `--auto medium` | Package installs, git commits, builds | Standard development |
| `--auto high` | Git push, deployments | Full polecat work |
| `--skip-permissions-unsafe` | Everything | Equivalent to Claude's `--dangerously-skip-permissions` |

For Gas Town, `--skip-permissions-unsafe` is the default (polecats need full
autonomy). But you could configure a read-only witness:

```json
{
  "version": 1,
  "agents": {
    "droid-readonly": {
      "name": "droid-readonly",
      "command": "droid",
      "args": [],
      "process_names": ["droid"],
      "supports_hooks": true,
      "prompt_mode": "arg",
      "config_dir": ".factory",
      "hooks_provider": "droid",
      "hooks_dir": ".factory",
      "hooks_settings_file": "settings.json",
      "ready_delay_ms": 5000,
      "instructions_file": "AGENTS.md"
    }
  }
}
```

### Plugins

Install Droid plugins for additional capabilities:

```bash
droid plugin install <plugin-name>    # Install from marketplace
droid plugin list                     # List installed plugins
```

Plugins can add skills, commands, hooks, and MCP servers that Gas Town polecats
can use during their work.

### Spec mode

Droid's `--use-spec` flag enables specification mode — a planning-then-execution
workflow that pairs well with Gas Town formulas:

```bash
droid exec --use-spec --auto high "Implement the caching layer per the spec in SPEC.md"
```

---

## Troubleshooting

### Hooks not firing

**Symptom:** `gt prime` context not injected on session start.

1. Check that `.factory/settings.json` exists in the working directory:
   ```bash
   cat ~/gt/myproject/.factory/settings.json
   ```

2. Verify hooks are not disabled:
   ```bash
   cat ~/.factory/settings.json | jq '.hooksDisabled'
   # Should be false or absent
   ```

3. Check that `gt` is in PATH inside the Droid session:
   ```bash
   which gt
   ```
   The hook templates prepend `$HOME/go/bin:$HOME/.local/bin` to PATH, but
   verify your `gt` binary is in one of these locations.

### Process detection fails

**Symptom:** Dashboard shows agent as not running, but Droid is visible in tmux.

Check what tmux reports as the running process:
```bash
tmux display-message -p '#{pane_current_command}'
```

If this shows something other than `droid`, update the preset's `process_names`.

### Resume not working

**Symptom:** Sessions start fresh instead of resuming.

Droid stores sessions in `~/.factory/sessions/`. Check that:
1. The session ID is being captured by Gas Town
2. The `--resume` flag is being passed correctly:
   ```bash
   droid --resume <session-id>
   ```

### Agent stuck / GUPP violation

**Symptom:** Dashboard shows 🔥 for an agent.

1. Try nudging: `gt nudge myproject/polecats/furiosa "check gt hook"`
2. If unresponsive, force handoff: `gt handoff --force myproject/polecats/furiosa`
3. If still stuck, force kill and redispatch:
   ```bash
   gt deacon force-kill myproject/polecats/furiosa
   gt sling <bead-id> myproject
   ```

### Non-interactive mode fails

**Symptom:** Formula execution errors.

Verify `droid exec` works standalone:
```bash
droid exec "echo hello" --auto high
```

Check that the `FACTORY_API_KEY` environment variable is set, or that you're
authenticated via `droid` interactive login.
