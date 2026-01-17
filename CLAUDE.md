# Conductor Protocol - Plugin Maintenance Guide

This file contains guidelines for maintaining the Conductor Protocol plugin.

## Version Management

Follow semantic versioning (MAJOR.MINOR.PATCH):
- **MAJOR**: Breaking changes to protocol or command interface
- **MINOR**: New features, backward compatible
- **PATCH**: Bug fixes, documentation updates

### Version Bump Checklist

When updating the plugin:
1. Update version in `.claude-plugin/plugin.json`
2. Add entry to `CHANGELOG.md` with date and changes
3. Update version history table in CHANGELOG.md
4. Commit with message format: `type: description`

### Commit Message Types
- `feat:` - New features
- `fix:` - Bug fixes
- `docs:` - Documentation changes
- `refactor:` - Code refactoring
- `chore:` - Maintenance tasks

## File Structure

```
conductor-protocol/
├── .claude-plugin/
│   ├── plugin.json           # Plugin metadata (version here)
│   └── marketplace.json      # Marketplace config
├── skills/
│   └── conductor/
│       ├── SKILL.md          # Main skill documentation
│       ├── protocol.md       # Protocol specification
│       └── schema.md         # JSON schemas
├── commands/
│   └── conductor.md          # /conductor command
├── templates/
│   ├── status-template.json  # Initial status.json
│   └── sprint-template.json  # Sprint config example
├── README.md                 # User-facing documentation
├── CHANGELOG.md              # Version history
├── CLAUDE.md                 # This file
└── LICENSE                   # MIT license
```

## Key Design Decisions

### Why File-Based Coordination?
- No external services required
- Works offline
- Simple to understand and debug
- Easy to manually inspect/fix state

### Why Pessimistic Locking?
- Prevents conflicts before they happen
- Clear ownership of files
- Human can always override with `--force`

### Why User-Level State (~/.conductor/)?
- Single coordination point across all projects
- State persists across sessions
- Easy to backup/restore

## Testing Changes

Before releasing:
1. Test `/conductor start` with various priorities
2. Test `/conductor claim` with conflict scenarios
3. Test `/conductor status` output formatting
4. Verify `~/.conductor/status.json` structure matches schema
5. Test orphan cleanup with old heartbeats

## Integration Points

### Digital-Self Integration
- Can integrate with `/daily-brief` to show status at session start
- Can integrate with `/shutdown` to release claims at session end
- Reads `north-stars.md` for priority alignment

### Future Enhancements
- [ ] Automatic heartbeat via hook
- [ ] Mission Control dashboard sync
- [ ] Pre-edit hook for automatic claim checking
- [ ] Webhook support for external dashboards
