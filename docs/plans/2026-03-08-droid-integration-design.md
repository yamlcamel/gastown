# Design: Factory Droid CLI Integration

**Date:** 2026-03-08
**Status:** Implemented

## Problem

Gas Town orchestrates multiple AI coding agents but has no built-in support
for Factory Droid CLI. Droid is a model-agnostic terminal agent with native
hooks, session resume, non-interactive execution, and a plugin system — all
of which map cleanly to Gas Town's agent abstraction. Users want to run Droid
agents (polecats, crew, witnesses) inside Gas Town's orchestration framework.

Additionally, there is no single guide that takes a Droid user from zero to
running a multi-agent fleet. Existing docs assume Claude as the default agent
and prior Gas Town knowledge.

## Goals

1. First-class Droid preset (built-in + portable example JSON)
2. Dedicated Droid hook installer (not reusing Claude's)
3. Comprehensive guide covering Gas Town + Droid from scratch, including
   work planning and decomposition

## Non-Goals

- Modifying Droid CLI itself
- Deprecating or replacing any existing agent support
- Covering Gas Town development/contribution workflows

## Deliverables

### 1. Built-in Droid Preset (`internal/config/agents.go`)

Add `AgentDroid` constant and full `AgentPresetInfo`:

```go
AgentDroid AgentPreset = "droid"
```

Preset fields (verified against Droid CLI `--help` and docs.factory.ai):

| Field | Value | Source |
|-------|-------|--------|
| Command | `droid` | `which droid` → native arm64 binary |
| Args | `["--skip-permissions-unsafe"]` | `droid exec --help` autonomy docs |
| ProcessNames | `["droid"]` | Native binary, not Node.js |
| SessionIDEnv | `""` | No env var observed; Droid stores sessions in `~/.factory/sessions/` |
| ResumeFlag | `--resume` | `droid --help` |
| ResumeStyle | `flag` | `droid --resume [sessionId]` |
| SupportsHooks | `true` | docs.factory.ai/reference/hooks-reference |
| PromptMode | `arg` | `droid "prompt"` works |
| ConfigDir | `.factory` | docs.factory.ai/cli/configuration/settings |
| HooksProvider | `droid` | Dedicated installer |
| HooksDir | `.factory` | Project-level config dir |
| HooksSettingsFile | `settings.json` | Same filename as Claude but different path |
| ReadyDelayMs | `5000` | TUI startup time estimate |
| InstructionsFile | `AGENTS.md` | docs.factory.ai/cli/configuration/agents-md |
| NonInteractive | `exec` subcommand, `--output-format json` | `droid exec --help` |

### 2. Droid Hook Installer (`internal/droid/`)

New package mirroring `internal/claude/config/`:

**`internal/droid/config/settings-autonomous.json`** — Hook template:

```json
{
  "hooks": {
    "SessionStart": [
      {
        "matcher": "",
        "hooks": [
          {
            "type": "command",
            "command": "gt prime --hook && gt mail check --inject"
          }
        ]
      }
    ],
    "PreCompact": [
      {
        "matcher": "",
        "hooks": [
          {
            "type": "command",
            "command": "gt prime --hook"
          }
        ]
      }
    ],
    "UserPromptSubmit": [
      {
        "matcher": "",
        "hooks": [
          {
            "type": "command",
            "command": "gt mail check --inject"
          }
        ]
      }
    ],
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "gt tap guard pr-workflow"
          }
        ]
      }
    ],
    "Stop": [
      {
        "matcher": "",
        "hooks": [
          {
            "type": "command",
            "command": "gt costs record"
          }
        ]
      }
    ]
  }
}
```

Hook events verified against Droid docs (docs.factory.ai/reference/hooks-reference):
`SessionStart`, `PreCompact`, `UserPromptSubmit`, `PreToolUse`, `PostToolUse`,
`Stop`, `SubagentStop`, `Notification`, `SessionEnd` — all present.

Format verified: same JSON structure as Claude (`hooks.EventName[].matcher/hooks[]`).

Key difference from Claude installer: hooks are written to `.factory/settings.json`
in the **working directory** (project-level), not a separate settings dir. Droid
does not have a `--settings` flag equivalent — it reads from `.factory/` in the
project root and `~/.factory/` at user level.

**`internal/droid/config/installer.go`** — Hook installer function:

```go
func EnsureSettingsForRoleAt(settingsDir, workDir, role, hooksDir, hooksFile string) error
```

Writes the settings template to `<workDir>/.factory/settings.json`, merging
with any existing project-level settings (preserving user hooks).

**Registration in `internal/runtime/runtime.go`:**

```go
config.RegisterHookInstaller("droid", droid.EnsureSettingsForRoleAt)
```

### 3. Example Preset (`examples/droid-preset.json`)

Standalone JSON for users who prefer external config over built-in:

```json
{
  "version": 1,
  "agents": {
    "droid": {
      "name": "droid",
      "command": "droid",
      "args": ["--skip-permissions-unsafe"],
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

### 4. Comprehensive Guide (`docs/guides/droid-integration.md`)

Fully self-contained — a Droid user goes from zero to multi-agent fleet.

**Outline:**

1. **What is Gas Town + Droid** — orientation, why use them together
2. **Prerequisites & Installation** — Gas Town, Droid, tmux, Dolt
3. **Setup** — town init, rig creation, activating Droid preset
4. **Your first Droid agent** — crew session, verify hooks, `gt prime`
5. **Planning work with Beads** — `bd create`, types, dependencies, epics, querying with `bd ready`/`bd blocked`
6. **Decomposing specs into tasks** — `mol-idea-to-plan` formula, 6-polecat PRD review, 6-polecat design, 5-polecat plan review, human gates, auto bead creation
7. **Wave-based dispatch** — `gt convoy stage`/`launch`, Kahn's algorithm, dependency-aware wave progression
8. **Polecats** — spawning workers, `gt sling`, the propulsion principle
9. **Convoys** — batch tracking, auto-convoy, stranded detection
10. **Formulas** — automated workflows, convoy/workflow/expansion types, `droid exec` integration
11. **Mixed agent fleets** — `role_agents`, multiple Droid presets with different models, Droid + Claude + GPT
12. **Scheduler** — `max_polecats`, deferred dispatch, circuit breaker
13. **Handoff & context cycling** — session continuity, `PreCompact` hooks, mail injection
14. **Dashboard & monitoring** — `gt dash`, activity feed, event symbols, problems view
15. **Health monitoring** — Witness patrol, Deacon/Boot chain, GUPP violations, stuck agent recovery
16. **Integration branches & merge queue** — epic delivery, Refinery, Bors-style batching
17. **Slash commands** — provisioning `/done`, `/handoff` into `.factory/commands/`
18. **Droid-specific features** — mission mode, custom droids, autonomy tiers (`--auto low/med/high`), plugins, spec mode
19. **Troubleshooting** — hooks not firing, process detection, resume failures, common errors

## Compatibility Research Summary

### Droid CLI Capabilities (verified)

| Capability | Droid Support | Verification |
|---|---|---|
| Autonomous mode | `--skip-permissions-unsafe` | `droid exec --help` |
| Hooks (settings.json) | 9 events, same JSON format | docs.factory.ai/reference/hooks-reference |
| Config directory | `.factory/` (project), `~/.factory/` (user) | docs.factory.ai/cli/configuration/settings |
| Session resume | `--resume [sessionId]` | `droid --help` |
| Non-interactive | `droid exec` subcommand | `droid exec --help` |
| Prompt as arg | `droid "prompt"` | `droid --help` |
| Process name | `droid` (native binary) | `file $(which droid)` → Mach-O arm64 |
| Instructions file | Reads `AGENTS.md` | docs.factory.ai/cli/configuration/agents-md |
| Plugins | `droid plugin install/list` | `droid plugin --help` |
| Custom subagents | `.factory/droids/` | docs.factory.ai/cli/configuration/custom-droids |
| Model selection | `-m <model>` per invocation | `droid exec --help` |
| Autonomy tiers | `--auto low/med/high` | `droid exec --help` |

### Hook Event Compatibility

| Gas Town Hook | Claude Event | Droid Event | Compatible |
|---|---|---|---|
| Context injection | SessionStart | SessionStart | Yes |
| Pre-compaction | PreCompact | PreCompact | Yes |
| Mail delivery | UserPromptSubmit | UserPromptSubmit | Yes |
| Tool guard | PreToolUse | PreToolUse | Yes |
| Cost tracking | Stop | Stop | Yes |

### Key Differences from Claude

| Aspect | Claude | Droid |
|---|---|---|
| Binary | Node.js (`node`) | Native (`droid`) |
| Config dir | `.claude/` | `.factory/` |
| Settings location | Separate `--settings` dir | Project `.factory/settings.json` |
| Permissions flag | `--dangerously-skip-permissions` | `--skip-permissions-unsafe` |
| Session storage | Env var `CLAUDE_SESSION_ID` | `~/.factory/sessions/` directory |
| Fork session | `--fork-session` supported | Not confirmed |

## Implementation Order

1. Verify Droid hook format against live CLI (write test hooks, confirm they fire)
2. Add `AgentDroid` preset to `internal/config/agents.go`
3. Create `internal/droid/config/` package with hook template and installer
4. Register hook installer in `internal/runtime/runtime.go`
5. Create `examples/droid-preset.json`
6. Write `docs/guides/droid-integration.md`
7. Test end-to-end: crew session → polecat spawn → convoy → formula
