---
name: conductor
description: "Multi-agent orchestration protocol for coordinating parallel Claude instances. Prevents file conflicts, enforces priority alignment, and provides real-time visibility across all running agents. Use /conductor to start tasks, claim files, check status, and release claims."
allowed-tools: Read, Write, Edit, Glob, Grep, Bash
---

# Conductor: Multi-Agent Orchestration Protocol

Conductor coordinates multiple Claude Code instances working in parallel. It prevents conflicts, enforces strategic alignment, and provides visibility into what each agent is doing.

## Quick Reference

```bash
/conductor start "task description" --project name --priority P1|P2|P3
/conductor status                    # Show all active instances
/conductor claim path/to/file.ts     # Lock files before editing
/conductor release                   # Release your file claims
/conductor done                      # Mark task complete, release all claims
/conductor align                     # Check task against north-stars
/conductor conflicts                 # Show any active conflicts
```

## Core Concepts

### The Problem

When running 3-5+ Claude instances simultaneously:
- **Conflicting changes**: Two agents edit the same file
- **Strategic drift**: Agents work on low-priority tasks
- **Lost visibility**: No idea what each instance is doing
- **Wasted effort**: Duplicate work or incompatible changes

### The Solution

A lightweight coordination protocol where:
1. Each agent **registers** when starting a task
2. Agents **claim files** before editing them
3. A **priority gate** rejects misaligned work
4. A **status dashboard** shows all activity

## How It Works

### Status File Location

Global coordination state lives at:
```
~/.conductor/
├── status.json      # Current state of all instances
├── history.jsonl    # Completed tasks log
└── conflicts.log    # Conflict history for learning
```

### Instance Lifecycle

```
┌─────────────────────────────────────────────────────────────┐
│                    INSTANCE LIFECYCLE                       │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   /conductor start "Fix auth bug" --project pitchello      │
│          │                                                  │
│          ▼                                                  │
│   ┌─────────────────┐                                       │
│   │ Priority Check  │──Fail──▶ ⚠️ Drift warning            │
│   │ (vs north-stars)│         (require override)            │
│   └────────┬────────┘                                       │
│            │ Pass                                           │
│            ▼                                                │
│   ┌─────────────────┐                                       │
│   │ Register in     │                                       │
│   │ status.json     │                                       │
│   └────────┬────────┘                                       │
│            │                                                │
│            ▼                                                │
│   /conductor claim src/auth.ts src/lib/session.ts          │
│          │                                                  │
│          ▼                                                  │
│   ┌─────────────────┐                                       │
│   │ Conflict Check  │──Conflict──▶ ⚠️ File locked          │
│   └────────┬────────┘              (coordinate)             │
│            │ Clear                                          │
│            ▼                                                │
│   ✅ Work on task (files are claimed)                      │
│            │                                                │
│            ▼                                                │
│   /conductor done                                           │
│          │                                                  │
│          ▼                                                  │
│   ┌─────────────────┐                                       │
│   │ Release claims  │                                       │
│   │ Log to history  │                                       │
│   │ Remove from     │                                       │
│   │ status.json     │                                       │
│   └─────────────────┘                                       │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## Commands Detail

### /conductor start

Register a new task with the conductor.

```bash
/conductor start "Implement user authentication" --project pitchello --priority P1
```

**What it does:**
1. Generates unique instance ID (e.g., `itm-a1b2c3`)
2. Checks priority against north-stars alignment
3. Registers in `~/.conductor/status.json`
4. Returns instance ID for reference

**Flags:**
- `--project` (required): Project name (pitchello, mission-control, digital-self, etc.)
- `--priority` (required): P1 (critical), P2 (important), P3 (nice-to-have)
- `--override` (optional): Skip priority gate with justification

### /conductor claim

Lock files before editing to prevent conflicts.

```bash
/conductor claim src/api/users.ts src/lib/auth.ts
```

**What it does:**
1. Checks if any file is already claimed
2. If clear, adds files to your instance's `files_claimed` array
3. If conflict, shows who has the file and suggests coordination

**Best practice:** Claim files at the start of work, not mid-task.

### /conductor status

Show dashboard of all active instances.

```bash
/conductor status
```

**Output:**
```
┌─────────────────────────────────────────────────────────────┐
│ CONDUCTOR STATUS - 2026-01-17 10:30                         │
├─────────────────────────────────────────────────────────────┤
│ ACTIVE INSTANCES (3)                                        │
├─────────────────────────────────────────────────────────────┤
│ [itm-a1b2] pitchello       │ P1 │ Fix auth bug      │ 45m  │
│            └─ src/api/users.ts, src/lib/auth.ts             │
│                                                             │
│ [itm-c3d4] mission-control │ P1 │ Lead scoring API  │ 30m  │
│            └─ app/api/agent/leads/*                         │
│                                                             │
│ [cur-e5f6] digital-self    │ P2 │ Content draft     │ 15m  │
│            └─ (no files claimed)                            │
├─────────────────────────────────────────────────────────────┤
│ CONFLICTS: None                                             │
│ DRIFT WARNINGS: 1 (cur-e5f6 working on P2 during P1 sprint) │
└─────────────────────────────────────────────────────────────┘
```

### /conductor done

Complete your task and release all claims.

```bash
/conductor done
```

**What it does:**
1. Releases all file claims
2. Logs task to `~/.conductor/history.jsonl`
3. Removes instance from `status.json`

### /conductor align

Check if your current/proposed task aligns with strategic priorities.

```bash
/conductor align "Add dark mode to settings"
```

**What it does:**
1. Reads `north-stars.md` from digital-self
2. Reads current sprint priorities (if defined)
3. Returns alignment assessment

### /conductor release

Release file claims without completing the task (e.g., when switching focus).

```bash
/conductor release src/api/users.ts
# or release all
/conductor release --all
```

### /conductor conflicts

Show detailed conflict information.

```bash
/conductor conflicts
```

## Integration Points

### With /daily-brief

At session start, daily-brief should:
1. Run `/conductor status` to show active instances
2. Flag any orphaned instances (started >24h ago)
3. Suggest cleanup if needed

### With /shutdown

At session end, shutdown should:
1. Run `/conductor done` to release this instance's claims
2. Log final status to history

### With File Edits

When editing files, agents should:
1. Check if file is claimed before editing
2. If not claimed, claim it first
3. If claimed by another, coordinate or wait

## Priority Levels

| Level | Description | Examples |
|-------|-------------|----------|
| P1 | Critical, sprint-blocking | Revenue features, production bugs |
| P2 | Important, planned work | Backlog items, improvements |
| P3 | Nice-to-have, opportunistic | Refactoring, documentation |

**Priority Gate Rule:** During focused sprints, P3 work requires explicit override.

## When to Use Conductor

**Always use when:**
- Running 2+ Claude instances simultaneously
- Working on shared codebases
- Tasks that will modify files

**Skip when:**
- Single instance, simple task
- Read-only research/exploration
- Content creation (no file conflicts possible)

## Troubleshooting

### Orphaned Instances

If an instance crashed without `/conductor done`:
```bash
/conductor cleanup --instance itm-a1b2c3
```

### Stale Claims

If files are claimed but instance is gone:
```bash
/conductor release --force src/api/users.ts
```

### Manual Status Reset

If status.json is corrupted:
```bash
/conductor reset
```
(Backs up current state to history, creates fresh status.json)

## See Also

- [protocol.md](./protocol.md) - Detailed protocol specification
- [schema.md](./schema.md) - JSON schema definitions
