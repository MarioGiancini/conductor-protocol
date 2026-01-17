---
description: Multi-agent orchestration - start tasks, claim files, check status, prevent conflicts
allowed-tools: Read, Write, Edit, Bash, Glob, Grep
argument-hint: <subcommand> [args]
---

# /conductor Command

Coordinate multiple Claude instances to prevent conflicts and maintain strategic alignment.

## Subcommands

Parse $ARGUMENTS to determine which subcommand to run:

- `start "task" --project name --priority P1|P2|P3` - Register a new task
- `status` - Show all active instances
- `claim path/to/file.ts [more files...]` - Lock files before editing
- `release [file] | --all` - Release file claims
- `done` - Complete task and release all claims
- `align "task description"` - Check task against north-stars
- `conflicts` - Show active conflicts
- `cleanup` - Remove orphaned instances
- `reset` - Reset status file (backup current state first)

## Setup Check

First, ensure the conductor directory and files exist:

```bash
# Check if status.json exists
if [[ ! -f ~/.conductor/status.json ]]; then
  # Create initial empty state
  echo '{
  "version": "1.0",
  "updated_at": "'$(date -u +%Y-%m-%dT%H:%M:%SZ)'",
  "instances": [],
  "global_claims": {},
  "drift_warnings": []
}' > ~/.conductor/status.json
fi
```

## Subcommand: start

When $ARGUMENTS starts with "start":

1. Parse arguments:
   - Task description (quoted string after "start")
   - `--project` flag value
   - `--priority` flag value (P1, P2, or P3)
   - `--override` flag value (optional justification)

2. Detect source environment:
   - Check `$CURSOR_SESSION` for Cursor
   - Check `$TERM_PROGRAM` for iTerm
   - Default to "web" if unknown

3. Generate instance ID:
   ```bash
   source_prefix="itm"  # or cur, vsc, web, cow
   random_suffix=$(openssl rand -hex 3)
   instance_id="${source_prefix}-${random_suffix}"
   ```

4. Priority gate check:
   - If priority is P1, pass automatically
   - If priority is P2 or P3, check `~/.conductor/sprint.json` if it exists
   - If sprint focus is P1 and task is lower priority, require `--override`

5. Add instance to status.json using jq:
   ```bash
   jq --arg id "$instance_id" \
      --arg source "$source" \
      --arg project "$project" \
      --arg task "$task" \
      --arg priority "$priority" \
      --arg started "$(date -u +%Y-%m-%dT%H:%M:%SZ)" \
      --arg heartbeat "$(date -u +%Y-%m-%dT%H:%M:%SZ)" \
      --arg workdir "$(pwd)" \
      '.instances += [{
        id: $id,
        source: $source,
        project: $project,
        task: $task,
        priority: $priority,
        files_claimed: [],
        started_at: $started,
        last_heartbeat: $heartbeat,
        working_directory: $workdir
      }] | .updated_at = $heartbeat' ~/.conductor/status.json > ~/.conductor/status.json.tmp \
      && mv ~/.conductor/status.json.tmp ~/.conductor/status.json
   ```

6. Store instance ID for this session (for subsequent commands):
   - Write to `~/.conductor/current_instance`
   - Or announce it clearly so the agent remembers it

7. Output:
   ```
   ✅ Registered as [instance_id]
   Project: [project]
   Task: [task]
   Priority: [priority]

   Next: Use `/conductor claim <files>` to lock files before editing.
   ```

## Subcommand: status

When $ARGUMENTS is "status" or empty:

1. Read ~/.conductor/status.json
2. Format as dashboard:

```
┌─────────────────────────────────────────────────────────────┐
│ CONDUCTOR STATUS - [timestamp]                              │
├─────────────────────────────────────────────────────────────┤
│ ACTIVE INSTANCES ([count])                                  │
├─────────────────────────────────────────────────────────────┤
│ [id] [project]      │ [priority] │ [task]          │ [age] │
│      └─ [claimed files list]                                │
│                                                             │
│ [repeat for each instance]                                  │
├─────────────────────────────────────────────────────────────┤
│ CONFLICTS: [count or "None"]                                │
│ DRIFT WARNINGS: [count with details]                        │
└─────────────────────────────────────────────────────────────┘
```

3. Calculate age from `started_at` to now
4. Highlight any orphaned instances (heartbeat > 60 min ago)

## Subcommand: claim

When $ARGUMENTS starts with "claim":

1. Get current instance ID from `~/.conductor/current_instance` or ask user
2. Parse file paths from arguments
3. For each file:
   - Check if in `global_claims`
   - If claimed by another, report conflict
   - If unclaimed, add to claims

4. Update status.json:
   ```bash
   jq --arg id "$instance_id" \
      --arg file "$filepath" \
      '.instances |= map(if .id == $id then .files_claimed += [$file] else . end)
       | .global_claims[$file] = $id
       | .updated_at = (now | todate)' ~/.conductor/status.json
   ```

5. Output:
   - Success: "✅ Claimed: [files]. Safe to edit."
   - Conflict: "⚠️ CONFLICT: [file] is claimed by [instance] ([task]). Coordinate or wait."

## Subcommand: done

When $ARGUMENTS is "done":

1. Get current instance ID
2. Get list of files that were claimed
3. Remove instance from status.json:
   ```bash
   jq --arg id "$instance_id" \
      'del(.instances[] | select(.id == $id))
       | .global_claims |= with_entries(select(.value != $id))
       | .updated_at = (now | todate)' ~/.conductor/status.json
   ```

4. Append to history.jsonl:
   ```bash
   echo '{"id":"'$instance_id'","project":"'$project'","task":"'$task'",...,"outcome":"success"}' >> ~/.conductor/history.jsonl
   ```

5. Clear current_instance file
6. Output: "✅ Task complete. All claims released."

## Subcommand: release

When $ARGUMENTS starts with "release":

1. If `--all`, release all files for current instance
2. Otherwise, release specific files listed
3. Update status.json to remove from both instance's files_claimed and global_claims
4. Output: "✅ Released: [files]"

## Subcommand: align

When $ARGUMENTS starts with "align":

1. Parse task description
2. Read north-stars from `~/Developer/digital-self/2-areas/personal/north-stars.md`
3. Check alignment with:
   - Priority hierarchy (Pitchello > LP > Self Engineer > Personal)
   - Current quarterly goals
   - Anti-goals (what we're NOT optimizing for)

4. Output assessment:
   ```
   ALIGNMENT CHECK: "[task]"

   ✅ Aligns with: [north-star priority]
   ⚠️ Concern: [any misalignment]

   Recommendation: [proceed/defer/reconsider]
   ```

## Subcommand: conflicts

When $ARGUMENTS is "conflicts":

1. Scan global_claims for any issues
2. Check for orphaned claims (instance gone but claims remain)
3. Report any active conflicts
4. Output detailed conflict information

## Subcommand: cleanup

When $ARGUMENTS is "cleanup":

1. Find instances with last_heartbeat > 60 minutes ago
2. For each orphan:
   - Release their claims
   - Log to history with outcome "orphaned"
   - Remove from status.json
3. Output: "✅ Cleaned up [count] orphaned instances"

## Subcommand: reset

When $ARGUMENTS is "reset":

1. Backup current status.json to history
2. Create fresh empty status.json
3. Output: "✅ Status reset. Previous state backed up to history."

## State File Location

All state lives in `~/.conductor/`:
- `status.json` - Current instance state
- `history.jsonl` - Completed task log
- `current_instance` - This session's instance ID
- `sprint.json` - Optional sprint focus configuration

## Error Handling

- If status.json is corrupted, offer to reset
- If instance ID not found, prompt to run `/conductor start` first
- If file claims fail, suggest `--force` option for human override

## See Also

Read the full skill documentation at `~/.claude/skills/conductor/SKILL.md` for:
- Complete protocol specification
- JSON schemas
- Integration patterns
