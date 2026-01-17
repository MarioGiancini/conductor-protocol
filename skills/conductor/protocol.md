# Conductor Protocol Specification

This document defines the technical protocol for multi-agent coordination.

## Design Principles

1. **File-based coordination** - No external services required, just JSON files
2. **Pessimistic locking** - Claim files before editing, not after
3. **Fail-safe defaults** - If in doubt, block and ask for human input
4. **Minimal overhead** - Protocol adds seconds, not minutes, to workflows
5. **Human override** - Humans can always force actions with `--force`

## Instance Identification

### ID Format

```
{source}-{random6}
```

Where:
- `source`: `itm` (iTerm), `cur` (Cursor), `vsc` (VS Code), `web` (browser), `cow` (Cowork)
- `random6`: 6 character alphanumeric

Examples: `itm-a1b2c3`, `cur-x7y8z9`, `web-m4n5o6`

### ID Generation

```bash
# Bash implementation
generate_instance_id() {
  local source=$1
  local random=$(openssl rand -hex 3)
  echo "${source}-${random}"
}
```

### Source Detection

Detect source from environment:
```bash
detect_source() {
  if [[ -n "$CURSOR_SESSION" ]]; then
    echo "cur"
  elif [[ -n "$VSCODE_GIT_IPC_HANDLE" ]]; then
    echo "vsc"
  elif [[ "$TERM_PROGRAM" == "iTerm.app" ]]; then
    echo "itm"
  elif [[ -n "$CLAUDE_COWORK_SESSION" ]]; then
    echo "cow"
  else
    echo "web"  # Default assumption
  fi
}
```

## File Structures

### ~/.conductor/status.json

The primary coordination file. All instances read and write to this.

```json
{
  "version": "1.0",
  "updated_at": "2026-01-17T10:30:00Z",
  "instances": [
    {
      "id": "itm-a1b2c3",
      "source": "itm",
      "project": "pitchello",
      "task": "Fix authentication bug in user service",
      "priority": "P1",
      "files_claimed": [
        "src/api/users.ts",
        "src/lib/auth.ts"
      ],
      "started_at": "2026-01-17T09:45:00Z",
      "last_heartbeat": "2026-01-17T10:30:00Z",
      "working_directory": "/Users/mariogiancini/Developer/pitchello"
    }
  ],
  "global_claims": {
    "src/api/users.ts": "itm-a1b2c3",
    "src/lib/auth.ts": "itm-a1b2c3"
  },
  "drift_warnings": [
    {
      "instance_id": "cur-x7y8z9",
      "reason": "P3 task during P1 sprint focus",
      "acknowledged": false
    }
  ]
}
```

### ~/.conductor/history.jsonl

Append-only log of completed tasks. One JSON object per line.

```jsonl
{"id":"itm-a1b2c3","project":"pitchello","task":"Fix auth bug","priority":"P1","started_at":"2026-01-17T09:45:00Z","completed_at":"2026-01-17T10:30:00Z","files_modified":["src/api/users.ts","src/lib/auth.ts"],"outcome":"success"}
{"id":"cur-x7y8z9","project":"mission-control","task":"Add lead endpoint","priority":"P1","started_at":"2026-01-17T10:00:00Z","completed_at":"2026-01-17T10:45:00Z","files_modified":["app/api/agent/leads/route.ts"],"outcome":"success"}
```

### ~/.conductor/conflicts.log

Log of conflicts for pattern analysis.

```
2026-01-17T10:30:00Z | CONFLICT | itm-a1b2c3 tried to claim src/api/users.ts already held by cur-x7y8z9
2026-01-17T10:35:00Z | RESOLVED | cur-x7y8z9 released src/api/users.ts, itm-a1b2c3 claimed it
```

## Operations

### START Operation

**Purpose:** Register a new task with the conductor.

**Input:**
```json
{
  "operation": "start",
  "task": "Fix authentication bug",
  "project": "pitchello",
  "priority": "P1",
  "working_directory": "/Users/mariogiancini/Developer/pitchello"
}
```

**Steps:**
1. Generate instance ID
2. Run priority check (see Priority Gate below)
3. If priority check fails and no `--override`, return error
4. Add instance to `status.json`
5. Update `updated_at` timestamp
6. Return instance ID

**Output (success):**
```json
{
  "status": "ok",
  "instance_id": "itm-a1b2c3",
  "message": "Registered. Use /conductor claim to lock files before editing."
}
```

**Output (priority fail):**
```json
{
  "status": "blocked",
  "reason": "priority_gate",
  "message": "Task priority P3 doesn't align with current P1 sprint focus.",
  "options": [
    "Use --override 'justification' to proceed anyway",
    "Change to a P1 task instead"
  ]
}
```

### CLAIM Operation

**Purpose:** Lock files before editing.

**Input:**
```json
{
  "operation": "claim",
  "instance_id": "itm-a1b2c3",
  "files": [
    "src/api/users.ts",
    "src/lib/auth.ts"
  ]
}
```

**Steps:**
1. Validate instance exists in status.json
2. For each file:
   a. Check if in `global_claims`
   b. If claimed by another instance, add to conflicts list
   c. If unclaimed, add to both instance's `files_claimed` and `global_claims`
3. If any conflicts, return conflict response
4. If all clear, return success

**Output (success):**
```json
{
  "status": "ok",
  "claimed": ["src/api/users.ts", "src/lib/auth.ts"],
  "message": "Files claimed. Safe to edit."
}
```

**Output (conflict):**
```json
{
  "status": "conflict",
  "conflicts": [
    {
      "file": "src/api/users.ts",
      "held_by": "cur-x7y8z9",
      "holder_task": "Refactoring user service",
      "holder_started": "2026-01-17T09:30:00Z"
    }
  ],
  "claimed": ["src/lib/auth.ts"],
  "message": "1 conflict detected. Coordinate with cur-x7y8z9 or wait for release."
}
```

### STATUS Operation

**Purpose:** Get current state of all instances.

**Input:**
```json
{
  "operation": "status"
}
```

**Output:**
```json
{
  "status": "ok",
  "instances": [...],
  "conflicts": [],
  "drift_warnings": [...],
  "summary": {
    "active_count": 3,
    "total_files_claimed": 5,
    "oldest_instance_age_minutes": 45
  }
}
```

### DONE Operation

**Purpose:** Complete task and release all claims.

**Input:**
```json
{
  "operation": "done",
  "instance_id": "itm-a1b2c3",
  "outcome": "success",
  "files_modified": ["src/api/users.ts"]
}
```

**Steps:**
1. Remove all instance's files from `global_claims`
2. Append to `history.jsonl`
3. Remove instance from `status.json`
4. Update `updated_at`

### RELEASE Operation

**Purpose:** Release specific files or all claims without completing.

**Input:**
```json
{
  "operation": "release",
  "instance_id": "itm-a1b2c3",
  "files": ["src/api/users.ts"],  // or "all": true
  "reason": "Switching to higher priority task"
}
```

### HEARTBEAT Operation

**Purpose:** Update last_heartbeat to indicate instance is still active.

**Input:**
```json
{
  "operation": "heartbeat",
  "instance_id": "itm-a1b2c3"
}
```

**Note:** Consider running heartbeat every 5-10 minutes for long tasks.

### CLEANUP Operation

**Purpose:** Remove orphaned instances (no heartbeat in >1 hour).

**Input:**
```json
{
  "operation": "cleanup",
  "max_age_minutes": 60
}
```

**Steps:**
1. Find instances where `last_heartbeat` is older than threshold
2. For each orphan:
   a. Release all their claims
   b. Log to history with outcome "orphaned"
   c. Remove from status.json

## Priority Gate

The priority gate prevents strategic drift by checking task alignment.

### Priority Sources (in order)

1. **Sprint priorities** (if defined): `~/.conductor/sprint.json`
2. **North-stars**: `~/Developer/digital-self/2-areas/personal/north-stars.md`
3. **Project priorities**: Project-level `plans/prd.json` or similar

### Sprint File (Optional)

```json
{
  "sprint_name": "2026-W03",
  "focus": "P1",
  "priorities": [
    "Pitchello revenue features",
    "Mission Control lead scoring",
    "LP AI Quality Platform"
  ],
  "explicitly_deferred": [
    "Documentation improvements",
    "Refactoring"
  ]
}
```

### Gate Logic

```python
def priority_gate(task, priority, project):
    # P1 always passes
    if priority == "P1":
        return {"pass": True}

    # Check sprint focus
    sprint = load_sprint_file()
    if sprint and sprint["focus"] == "P1" and priority != "P1":
        return {
            "pass": False,
            "reason": f"Sprint focus is P1, task is {priority}",
            "suggestion": "Override with justification or defer task"
        }

    # Check north-stars alignment
    north_stars = load_north_stars()
    if not aligns_with_priorities(task, north_stars):
        return {
            "pass": False,
            "reason": "Task doesn't align with north-stars priorities",
            "current_priorities": north_stars["priority_hierarchy"]
        }

    return {"pass": True}
```

## Conflict Resolution

When a conflict occurs, the protocol suggests these resolution paths:

### 1. Wait

The simplest option. The other instance will eventually finish.

### 2. Coordinate

Instances can coordinate by:
- One instance releases the contested file
- Work is divided (one takes lines 1-100, other takes 101-200)
- Sequential execution (finish one task before starting other)

### 3. Force

Human override with `--force`:
```bash
/conductor claim src/api/users.ts --force "Talked to other instance, taking over"
```

Force operations are logged to `conflicts.log` for review.

## Concurrency Safety

### File Locking

The status.json file itself needs protection from concurrent writes.

**Approach: Atomic writes with lockfile**

```bash
conductor_write() {
  local lockfile="$HOME/.conductor/status.lock"

  # Acquire lock (wait up to 5 seconds)
  exec 200>"$lockfile"
  flock -w 5 200 || { echo "Failed to acquire lock"; return 1; }

  # Read current state
  local current=$(cat "$HOME/.conductor/status.json")

  # Apply operation
  local updated=$(echo "$current" | jq "$1")

  # Write atomically
  echo "$updated" > "$HOME/.conductor/status.json.tmp"
  mv "$HOME/.conductor/status.json.tmp" "$HOME/.conductor/status.json"

  # Release lock
  exec 200>&-
}
```

### Race Condition Handling

If two instances try to claim the same file simultaneously:
1. Lock acquisition serializes the operations
2. Second instance will see first's claim and get conflict response

## Extension Points

### Custom Priority Sources

Projects can define their own priority sources:
```json
// .conductor/config.json in project root
{
  "priority_source": "plans/prd.json",
  "priority_field": "tasks[].priority"
}
```

### Webhooks (Future)

For integration with external dashboards:
```json
{
  "webhooks": {
    "on_start": "https://...",
    "on_conflict": "https://...",
    "on_done": "https://..."
  }
}
```

### Mission Control Integration (Future)

Sync conductor state to Mission Control database for persistent dashboard.
