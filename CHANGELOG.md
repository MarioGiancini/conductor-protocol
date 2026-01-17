# Changelog

All notable changes to the Conductor Protocol plugin are documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [1.0.0] - 2026-01-17

### Added
- **Initial release** of Conductor Protocol plugin
- **/conductor command** with subcommands:
  - `start` - Register a new task with project and priority
  - `status` - Show dashboard of all active instances
  - `claim` - Lock files before editing
  - `release` - Release file claims
  - `done` - Complete task and release all claims
  - `align` - Check task against north-stars
  - `conflicts` - Show active conflicts
  - `cleanup` - Remove orphaned instances
  - `reset` - Reset status file
- **Conductor skill** with full documentation:
  - SKILL.md - Quick reference and overview
  - protocol.md - Detailed protocol specification
  - schema.md - JSON schema definitions
- **State management** via `~/.conductor/`:
  - status.json - Current instance state
  - history.jsonl - Completed tasks log
  - sprint.json - Optional sprint focus configuration
- **Conflict prevention** via file claiming system
- **Priority gate** for strategic alignment
- **Instance identification** by source (iTerm, Cursor, VS Code, browser, Cowork)

### Architecture
- File-based coordination (no external services required)
- Pessimistic locking (claim before edit)
- Human override capability (`--force` flag)
- Minimal overhead design

## Version History Summary

| Version | Date | Highlights |
|---------|------|------------|
| 1.0.0 | 2026-01-17 | Initial release |
