# Conductor JSON Schemas

This document defines the JSON schemas for all conductor data structures.

## status.json Schema

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "$id": "conductor-status",
  "title": "Conductor Status",
  "description": "Current state of all active Claude instances",
  "type": "object",
  "required": ["version", "updated_at", "instances", "global_claims"],
  "properties": {
    "version": {
      "type": "string",
      "const": "1.0"
    },
    "updated_at": {
      "type": "string",
      "format": "date-time",
      "description": "ISO 8601 timestamp of last update"
    },
    "instances": {
      "type": "array",
      "items": { "$ref": "#/$defs/instance" }
    },
    "global_claims": {
      "type": "object",
      "description": "Map of file path to instance ID that holds the claim",
      "additionalProperties": {
        "type": "string",
        "pattern": "^[a-z]{3}-[a-f0-9]{6}$"
      }
    },
    "drift_warnings": {
      "type": "array",
      "items": { "$ref": "#/$defs/drift_warning" }
    }
  },
  "$defs": {
    "instance": {
      "type": "object",
      "required": ["id", "source", "project", "task", "priority", "started_at", "last_heartbeat", "working_directory"],
      "properties": {
        "id": {
          "type": "string",
          "pattern": "^[a-z]{3}-[a-f0-9]{6}$",
          "description": "Unique instance identifier (e.g., itm-a1b2c3)"
        },
        "source": {
          "type": "string",
          "enum": ["itm", "cur", "vsc", "web", "cow"],
          "description": "Source application (iTerm, Cursor, VS Code, browser, Cowork)"
        },
        "project": {
          "type": "string",
          "description": "Project name (e.g., pitchello, mission-control)"
        },
        "task": {
          "type": "string",
          "description": "Human-readable task description"
        },
        "priority": {
          "type": "string",
          "enum": ["P1", "P2", "P3"],
          "description": "Task priority level"
        },
        "files_claimed": {
          "type": "array",
          "items": { "type": "string" },
          "description": "List of file paths this instance has claimed"
        },
        "started_at": {
          "type": "string",
          "format": "date-time"
        },
        "last_heartbeat": {
          "type": "string",
          "format": "date-time"
        },
        "working_directory": {
          "type": "string",
          "description": "Absolute path to working directory"
        },
        "override_reason": {
          "type": "string",
          "description": "If priority gate was overridden, the justification"
        }
      }
    },
    "drift_warning": {
      "type": "object",
      "required": ["instance_id", "reason", "acknowledged"],
      "properties": {
        "instance_id": {
          "type": "string",
          "pattern": "^[a-z]{3}-[a-f0-9]{6}$"
        },
        "reason": {
          "type": "string"
        },
        "acknowledged": {
          "type": "boolean"
        },
        "acknowledged_at": {
          "type": "string",
          "format": "date-time"
        }
      }
    }
  }
}
```

## history.jsonl Entry Schema

Each line in history.jsonl follows this schema:

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "$id": "conductor-history-entry",
  "title": "Conductor History Entry",
  "description": "Record of a completed task",
  "type": "object",
  "required": ["id", "project", "task", "priority", "started_at", "completed_at", "outcome"],
  "properties": {
    "id": {
      "type": "string",
      "pattern": "^[a-z]{3}-[a-f0-9]{6}$"
    },
    "project": {
      "type": "string"
    },
    "task": {
      "type": "string"
    },
    "priority": {
      "type": "string",
      "enum": ["P1", "P2", "P3"]
    },
    "started_at": {
      "type": "string",
      "format": "date-time"
    },
    "completed_at": {
      "type": "string",
      "format": "date-time"
    },
    "duration_minutes": {
      "type": "integer",
      "description": "Calculated from started_at to completed_at"
    },
    "files_modified": {
      "type": "array",
      "items": { "type": "string" },
      "description": "Files that were actually modified during the task"
    },
    "outcome": {
      "type": "string",
      "enum": ["success", "abandoned", "orphaned", "conflict"],
      "description": "How the task ended"
    },
    "notes": {
      "type": "string",
      "description": "Optional notes about the task completion"
    }
  }
}
```

## sprint.json Schema (Optional)

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "$id": "conductor-sprint",
  "title": "Conductor Sprint",
  "description": "Current sprint focus and priorities",
  "type": "object",
  "required": ["sprint_name", "focus"],
  "properties": {
    "sprint_name": {
      "type": "string",
      "description": "Sprint identifier (e.g., 2026-W03)"
    },
    "focus": {
      "type": "string",
      "enum": ["P1", "P2", "P3"],
      "description": "Minimum priority level for this sprint"
    },
    "start_date": {
      "type": "string",
      "format": "date"
    },
    "end_date": {
      "type": "string",
      "format": "date"
    },
    "priorities": {
      "type": "array",
      "items": { "type": "string" },
      "description": "List of priority areas for this sprint"
    },
    "explicitly_deferred": {
      "type": "array",
      "items": { "type": "string" },
      "description": "Work explicitly deferred to later sprints"
    }
  }
}
```

## Example Files

### Empty status.json (Initial State)

```json
{
  "version": "1.0",
  "updated_at": "2026-01-17T00:00:00Z",
  "instances": [],
  "global_claims": {},
  "drift_warnings": []
}
```

### Active status.json (3 Instances)

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
    },
    {
      "id": "cur-d4e5f6",
      "source": "cur",
      "project": "mission-control",
      "task": "Add lead scoring API endpoint",
      "priority": "P1",
      "files_claimed": [
        "app/api/agent/leads/score/route.ts",
        "lib/scoring.ts"
      ],
      "started_at": "2026-01-17T10:00:00Z",
      "last_heartbeat": "2026-01-17T10:28:00Z",
      "working_directory": "/Users/mariogiancini/Developer/digital-self/1-projects/mission-control"
    },
    {
      "id": "web-g7h8i9",
      "source": "web",
      "project": "self-engineer",
      "task": "Draft article on AI agent coordination",
      "priority": "P2",
      "files_claimed": [],
      "started_at": "2026-01-17T10:15:00Z",
      "last_heartbeat": "2026-01-17T10:29:00Z",
      "working_directory": "/Users/mariogiancini/Developer/digital-self"
    }
  ],
  "global_claims": {
    "src/api/users.ts": "itm-a1b2c3",
    "src/lib/auth.ts": "itm-a1b2c3",
    "app/api/agent/leads/score/route.ts": "cur-d4e5f6",
    "lib/scoring.ts": "cur-d4e5f6"
  },
  "drift_warnings": [
    {
      "instance_id": "web-g7h8i9",
      "reason": "P2 task during P1 sprint focus",
      "acknowledged": true,
      "acknowledged_at": "2026-01-17T10:15:30Z"
    }
  ]
}
```

### Sample history.jsonl

```jsonl
{"id":"itm-x1y2z3","project":"pitchello","task":"Fix TypeScript errors in API","priority":"P1","started_at":"2026-01-17T08:00:00Z","completed_at":"2026-01-17T09:30:00Z","duration_minutes":90,"files_modified":["src/api/users.ts","src/api/projects.ts","src/types/index.ts"],"outcome":"success"}
{"id":"cur-a4b5c6","project":"mission-control","task":"Add content pipeline status endpoint","priority":"P1","started_at":"2026-01-17T08:30:00Z","completed_at":"2026-01-17T09:15:00Z","duration_minutes":45,"files_modified":["app/api/agent/content/status/route.ts"],"outcome":"success"}
{"id":"itm-d7e8f9","project":"digital-self","task":"Update daily-brief command","priority":"P2","started_at":"2026-01-16T14:00:00Z","completed_at":"2026-01-16T14:30:00Z","duration_minutes":30,"files_modified":[".claude/commands/daily-brief.md"],"outcome":"success","notes":"Added conductor status integration"}
```

## Validation

### TypeScript Types (Reference)

```typescript
type InstanceSource = 'itm' | 'cur' | 'vsc' | 'web' | 'cow';
type Priority = 'P1' | 'P2' | 'P3';
type Outcome = 'success' | 'abandoned' | 'orphaned' | 'conflict';

interface Instance {
  id: string;                    // Pattern: /^[a-z]{3}-[a-f0-9]{6}$/
  source: InstanceSource;
  project: string;
  task: string;
  priority: Priority;
  files_claimed: string[];
  started_at: string;            // ISO 8601
  last_heartbeat: string;        // ISO 8601
  working_directory: string;
  override_reason?: string;
}

interface DriftWarning {
  instance_id: string;
  reason: string;
  acknowledged: boolean;
  acknowledged_at?: string;
}

interface ConductorStatus {
  version: '1.0';
  updated_at: string;
  instances: Instance[];
  global_claims: Record<string, string>;  // file path -> instance ID
  drift_warnings: DriftWarning[];
}

interface HistoryEntry {
  id: string;
  project: string;
  task: string;
  priority: Priority;
  started_at: string;
  completed_at: string;
  duration_minutes?: number;
  files_modified?: string[];
  outcome: Outcome;
  notes?: string;
}
```

### jq Validation Commands

```bash
# Validate status.json structure
jq -e '.version == "1.0" and (.instances | type) == "array"' ~/.conductor/status.json

# Check for orphaned instances (no heartbeat in 60 min)
jq --arg cutoff "$(date -v-60M -u +%Y-%m-%dT%H:%M:%SZ)" \
  '.instances[] | select(.last_heartbeat < $cutoff)' ~/.conductor/status.json

# List all claimed files
jq '.global_claims | keys[]' ~/.conductor/status.json

# Find conflicts (same file claimed by multiple - shouldn't happen)
jq '.global_claims | to_entries | group_by(.value) | map(select(length > 1))' ~/.conductor/status.json
```
