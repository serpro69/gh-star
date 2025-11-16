## Project Overview & Goals

`gh-star` is a GitHub CLI extension for power users managing large collections (1000+) of starred repositories. It addresses the limitations of GitHub's web interface for users who treat stars as a curated knowledge base rather than casual bookmarks.

Target User: Developers with extensive star collections (1500-3000+) who need robust search, organization, and batch operations that the GitHub web UI doesn't provide.

Core Problems Solved:
1. Search & Discovery - GitHub's star search is limited and slow with large collections
2. Organization - Need flexible tagging, categorization, and notes beyond GitHub's basic lists
3. Batch Operations - Bulk starring/unstarring, cleanup, migrations
4. Local Control - Ability to backup, version control, and own your star data
5. CLI Workflow - Integration with terminal workflows, scripting, and Unix pipelines

Design Philosophy:
- Local-first - Primary data store is local SQLite, fast queries, offline access
- Pipeline-friendly - Outputs work with Unix tools (grep, fzf, jq, xargs)
- Hybrid interaction - Default to scriptable output, opt-in to interactive mode
- Git-compatible - Export to version-controllable formats (YAML/JSON)
- GitHub-aware - Sync with GitHub data, but local metadata stays local

## Architecture Overview

`gh-star` follows a local-first architecture with explicit sync points to GitHub's API.

Core Components:

1. CLI Interface (`gh star <command>`)
  - Built as a GitHub CLI extension using Go
  - Follows gh CLI patterns and conventions
  - Supports both human-friendly and machine-readable output
2. Local Database (SQLite with modernc.org/sqlite)
  - Pure-Go driver (no CGo) for easy cross-compilation
  - Primary data store at ~/.config/gh-star/stars.db
  - Full-text search using SQLite FTS5 extension
  - Relational schema for efficient querying
3. GitHub API Integration
  - Uses gh CLI for authentication (no separate auth needed)
  - Fetches starred repos, metadata, and Star Lists
  - Read-only operations (except star/unstar commands)
  - Respects rate limits with progress indicators
4. Export/Import System
  - YAML and JSON formats (user selectable)
  - Directory structure: one file per repo for clean git diffs
  - Explicit commands: `gh star export` and `gh star import`
  - Enables version control, backup, and sharing

Data Flow:
```
GitHub API → sync → SQLite DB → query/filter → CLI output
                       ↓
                    export → YAML/JSON files → git commit
                       ↑
                    import ← YAML/JSON files
```

## Data Model

SQLite Schema Design:

Table: `repositories`
- Stores core GitHub repository data
- Fields: `id` (GitHub ID), `full_name`, `owner`, `name`, `description`, `url`, `homepage`, `language`, `stars_count`, `forks_count`, `topics` (JSON array), `archived`, `starred_at` (timestamp)
- Primary key on `id`, indexes on `language`, `starred_at`, and `full_name`

Table: `tags`
- User-defined tags (both `gh:*` and custom tags)
- Fields: `id`, `name`, `is_github_list` (boolean flag for `gh:*` tags)
- Unique constraint on `name`

Table: `repository_tags`
- Many-to-many relationship between `repositories` and `tags`
- Fields: `repository_id`, `tag_id`, `created_at`
- Composite primary key, foreign keys to both tables

Table: `repository_metadata`
- User-added metadata per repository
- Fields: `repository_id`, `notes` (text), `custom_fields` (JSON for extensibility), `last_modified`
- One-to-one with `repositories`

Full-Text Search:
- FTS5 virtual table indexing: `full_name`, `description`, `topics`, and `notes`
- Enables fast queries like: "search for 'kubernetes docker' in any field"

Tag Namespacing:
- GitHub List "CLI Tools" → tag `gh:cli-tools`
- User tags have no prefix: `production`, `priority-1`, `needs-review`

## Core Features & Commands

Primary Commands:

Initialization & Sync:
- `gh star init` - Initial setup, create database, import all stars and GitHub Lists
- `gh star sync` - Fetch new stars, update metadata, reconcile gh:* tags from GitHub Lists
- `gh star sync --full` - Full refresh of all star data

Search & Discovery:
- `gh star list` - List all stars (supports filters, sorting)
- `gh star search <query>` - Full-text search across names, descriptions, topics, notes
- `gh star search --tag cli --tag go` - Filter by tags (AND logic)
- `gh star search --language Go --since 2024` - Filter by language, date ranges
- `gh star show <repo>` - Detailed view of a single repo with all metadata

Tagging & Organization:
- `gh star tag add <repo> <tags...>` - Add tags to a repository
- `gh star tag remove <repo> <tags...>` - Remove tags
- `gh star tags` - List all tags with repo counts
- `gh star note set <repo> "text"` - Add/update notes for a repo
- `gh star note show <repo>` - Display notes

Batch Operations:
- `gh star bulk-tag --query "language:Go" --tags backend,production` - Tag matching repos
- `gh star bulk-unstar --query "archived:true"` - Unstar repos matching criteria
- Pipe support: `gh star search docker | xargs -I {} gh star tag add {} container`

## GitHub Integration & Sync Strategy

One-Way Sync with Namespaced Tags:

GitHub Star Lists are synchronized as special `gh:*` prefixed tags. This creates a clear separation between GitHub-managed and user-managed organization.

Initial Import (`gh star init`):
1. Fetch all starred repositories via GitHub API
2. Fetch all Star Lists and their memberships
3. Create local tags for each list: "CLI Tools" → `gh:cli-tools`
4. Apply `gh:*` tags to repositories based on list membership
5. Store all data in local SQLite database

Ongoing Sync (`gh star sync`):
1. Fetch new starred repositories (since last sync)
2. Fetch current Star Lists memberships
3. Reconciliation process for `gh:*` tags:
  - Delete ALL existing `gh:*` tags from local database
  - Re-apply `gh:*` tags based on current GitHub Lists data
  - User tags are never touched - only `gh:*` namespace is managed
4. Update repository metadata (stars count, description changes, etc.)

Key Behaviors:
- Stars added on GitHub web → automatically imported on next sync
- Repos added to lists on GitHub → `gh:*` tags applied locally
- Repos removed from lists on GitHub → `gh:*` tags removed locally
- User tags (`production`, `priority-1`) remain completely independent
- No write-back to GitHub (except explicit star/unstar commands)

Clear Mental Model: "GitHub Lists control `gh:*` tags. Everything else is mine."

## Import/Export System

Export/Import for Version Control and Backup:

The export system transforms the SQLite database into human-readable, version-controllable files.

Export Structure:
```
~/my-stars/
├── stars/
│   ├── golang/
│   │   └── go.yaml (or .json)
│   ├── kubernetes/
│   │   └── kubernetes.yaml
│   └── ...
└── metadata.yaml  # Export timestamp, stats, config
```

Per-Repository File Format (YAML example):
```yaml
# Exported by gh-star on 2024-01-15
github_id: 23096959
full_name: golang/go
owner: golang
name: go
description: The Go programming language
url: https://github.com/golang/go
language: Go
topics: [compiler, go, programming-language]
starred_at: 2020-03-15T10:30:00Z

# User metadata
tags:
  - gh:programming-languages  # From GitHub List
  - backend                    # User tag
  - production                 # User tag
notes: |
  Core Go repository. Check releases for security updates.
  Used in production services.
```

Commands:
- `gh star export <path>` - Export to directory (default: YAML)
- `gh star export <path> --format json` - Export as JSON
- `gh star import <path>` - Import from directory (auto-detects format)
- `gh star import <path> --merge` - Merge with existing data instead of replacing

Version Control Workflow:
```bash
gh star export ~/my-stars
cd ~/my-stars && git add . && git commit -m "Update stars"
```

## Interactive Mode & Pipeline Integration

Hybrid Design Philosophy:

By default, commands output plain text or structured data for piping. Interactive mode is opt-in via `--interactive` or `-i` flag.

Default Pipeline-Friendly Output:
```bash
# Simple list output (one repo per line)
gh star list
# golang/go
# kubernetes/kubernetes
# ...

# Structured JSON for programmatic use
gh star list --json | jq '.[] | select(.language == "Go")'

# Tab-separated for Unix tools
gh star list --format tsv | grep docker | cut -f1
```

Interactive Mode (`--interactive flag`):

Uses `fzf` for fuzzy finding and interactive selection:

```bash
# Interactive search with preview
gh star search --interactive
# Opens fzf with:
# - Fuzzy search across all repos
# - Preview pane showing repo details, tags, notes
# - Multi-select support (TAB to select multiple)
# - Returns selected repos to stdout

# Interactive tagging workflow
gh star search "kubernetes" -i | xargs -I {} gh star tag add {} production

# Interactive browsing
gh star list -i --preview-command="gh star show {}"
```

`fzf` Integration Features:
- Custom preview showing: `description`, `language`, `stars`, `tags`, `notes`
- Multi-select for batch operations
- Keybindings: <key>CTRL-T</key> (tag), <key>CTRL-O</key> (open in browser), <key>CTRL-N</key> (add note)
- Falls back gracefully if `fzf` not installed (shows warning, uses simple list)

Future Enhancement: Optional rich TUI (like `lazygit`) for users who want fully integrated experience.

## Implementation Details

Technology Stack:

Language & Runtime:
- Go 1.21+ for CLI implementation
- Single static binary, cross-platform (Linux, macOS, Windows)
- GitHub CLI extension conventions and patterns

Key Dependencies:
- `modernc.org/sqlite` - Pure-Go SQLite driver (no CGo)
- `github.com/cli/go-gh` - GitHub CLI integration and auth
- `github.com/spf13/cobra` - CLI framework (standard for gh extensions)
- `gopkg.in/yaml.v3` - YAML parsing/generation
- `encoding/json` - Standard library JSON support

Database Implementation:
- SQLite with FTS5 extension enabled for full-text search
- Write-Ahead Logging (WAL) mode for better concurrency
- Pragmas: `journal_mode=WAL`, `synchronous=NORMAL`, `foreign_keys=ON`
- Connection pooling for CLI commands (single connection, no concurrency needed)

GitHub API:
- Uses gh api commands for authentication (inherits from gh CLI)
- Paginated fetching for large star collections (handles 3000+ stars)
- Rate limit handling with exponential backoff
- Progress indicators for long-running syncs

File System:
- Config/database location: `~/.config/gh-star/` (XDG Base Directory compliant)
- Database file: `~/.config/gh-star/stars.db`
- Config file: `~/.config/gh-star/config.yaml` (for user preferences)

Testing Strategy:
- Unit tests for core logic (tagging, search, filters)
- Integration tests with mock GitHub API responses
- CLI command tests using golden files for output validation
- Test database fixtures for reproducible tests
