# Conductor Protocol

A Claude Code plugin for multi-agent orchestration. Coordinate parallel Claude instances to prevent file conflicts, enforce priority alignment, and maintain visibility across all running agents.

## The Problem

When running 3-5+ Claude instances simultaneously:
- **Conflicting changes**: Two agents edit the same file
- **Strategic drift**: Agents work on low-priority tasks
- **Lost visibility**: No idea what each instance is doing
- **Wasted effort**: Duplicate work or incompatible changes

## The Solution

A lightweight coordination protocol where:
1. Each agent **registers** when starting a task
2. Agents **claim files** before editing them
3. A **priority gate** rejects misaligned work
4. A **status dashboard** shows all activity

## Installation

### Plugin Install

```bash
# Add the marketplace
claude plugin marketplace add MarioGiancini/conductor-protocol

# Install the plugin
claude plugin install conductor-protocol
```

### Updating

```bash
# Update marketplace cache first
claude plugin marketplace update conductor-protocol

# Update the plugin
claude plugin update conductor-protocol@conductor-protocol
```

## Quick Start

```bash
# 1. Start a task (registers you in the conductor)
/conductor start "Fix auth bug" --project pitchello --priority P1

# 2. Claim files before editing
/conductor claim src/api/users.ts src/lib/auth.ts

# 3. Do your work...

# 4. When done, release claims and log completion
/conductor done

# Anytime: Check what all instances are doing
/conductor status
```

## Commands

| Command | Description |
|---------|-------------|
| `/conductor start "task" --project --priority` | Register a new task |
| `/conductor status` | Show all active instances |
| `/conductor claim <files>` | Lock files before editing |
| `/conductor release [files]` | Release file claims |
| `/conductor done` | Complete task and release all claims |
| `/conductor align "task"` | Check task against north-stars |
| `/conductor conflicts` | Show active conflicts |
| `/conductor cleanup` | Remove orphaned instances |

## How It Works

### Instance Lifecycle

```
/conductor start "Fix auth bug" --project pitchello --priority P1
         │
         ▼
  ┌─────────────────┐
  │ Priority Check  │──Fail──▶ ⚠️ Drift warning
  └────────┬────────┘
           │ Pass
           ▼
/conductor claim src/auth.ts
         │
         ▼
  ┌─────────────────┐
  │ Conflict Check  │──Conflict──▶ ⚠️ File locked
  └────────┬────────┘
           │ Clear
           ▼
  ✅ Work on task
           │
           ▼
/conductor done
         │
         ▼
  ✅ Claims released, logged to history
```

### Status Dashboard

```
/conductor status

┌─────────────────────────────────────────────────────────────┐
│ CONDUCTOR STATUS - 2026-01-17 10:30                         │
├─────────────────────────────────────────────────────────────┤
│ ACTIVE INSTANCES (3)                                        │
├─────────────────────────────────────────────────────────────┤
│ [itm-a1b2] pitchello       │ P1 │ Fix auth bug      │ 45m  │
│            └─ src/api/users.ts, src/lib/auth.ts             │
│                                                             │
│ [cur-c3d4] mission-control │ P1 │ Lead scoring API  │ 30m  │
│            └─ app/api/agent/leads/*                         │
│                                                             │
│ [web-e5f6] digital-self    │ P2 │ Content draft     │ 15m  │
│            └─ (no files claimed)                            │
├─────────────────────────────────────────────────────────────┤
│ CONFLICTS: None                                             │
│ DRIFT WARNINGS: 1 (web-e5f6 working on P2 during P1 sprint) │
└─────────────────────────────────────────────────────────────┘
```

## State Files

All state lives in `~/.conductor/`:

| File | Purpose |
|------|---------|
| `status.json` | Current instance state and file claims |
| `history.jsonl` | Completed tasks log |
| `sprint.json` | Optional sprint focus configuration |
| `conflicts.log` | Conflict history for pattern analysis |

## Priority Levels

| Level | Description | Examples |
|-------|-------------|----------|
| P1 | Critical, sprint-blocking | Revenue features, production bugs |
| P2 | Important, planned work | Backlog items, improvements |
| P3 | Nice-to-have, opportunistic | Refactoring, documentation |

**Priority Gate Rule:** During focused sprints, P3 work requires explicit override.

## Integration

### With Daily Workflows

- `/daily-brief` can show conductor status at session start
- `/shutdown` can run `/conductor done` to release claims

### With Strategic Alignment

- `/conductor align` checks tasks against north-stars.md
- Sprint focus can be configured via `~/.conductor/sprint.json`

## Philosophy

> "Progress should persist. Failures should evaporate. Conflicts should be prevented."

- **File claiming** prevents merge conflicts before they happen
- **Priority gates** keep work aligned with strategic goals
- **Visibility** lets you orchestrate multiple agents confidently

## Requirements

- Claude Code CLI
- `jq` (for JSON parsing)

## Credits

- Inspired by [Boris Cherny's 15-parallel-instance workflow](https://x.com/aakashgupta/status/2012396910221693216)
- Built for the [digital-self](https://github.com/MarioGiancini) multi-agent architecture

## License

MIT
