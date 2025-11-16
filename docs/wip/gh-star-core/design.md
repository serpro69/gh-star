# gh-star: Design Document

## Overview

**gh-star** is a GitHub CLI extension for power users managing large collections (1000+) of starred repositories. It provides robust search, organization, batch operations, and local control that GitHub's web interface doesn't offer.

### Target Audience

Developers with extensive star collections (1500-3000+) who treat stars as a curated knowledge base rather than casual bookmarks.

### Core Problems Solved

1. **Search & Discovery** - GitHub's star search is limited and slow with large collections
2. **Organization** - Need flexible tagging, categorization, and notes beyond GitHub's basic lists
3. **Batch Operations** - Bulk starring/unstarring, cleanup, migrations
4. **Local Control** - Ability to backup, version control, and own your star data
5. **CLI Workflow** - Integration with terminal workflows, scripting, and Unix pipelines

### Design Philosophy

- **Local-first**: Primary data store is local SQLite, fast queries, offline access
- **Pipeline-friendly**: Outputs work with Unix tools (grep, fzf, jq, xargs)
- **Hybrid interaction**: Default to scriptable output, opt-in to interactive mode
- **Git-compatible**: Export to version-controllable formats (YAML/JSON)
- **GitHub-aware**: Sync with GitHub data, but local metadata stays local

## Architecture

### High-Level Components

```
┌─────────────────┐
│   GitHub API    │
│  (Stars, Lists) │
└────────┬────────┘
         │ sync
         ▼
┌─────────────────┐      ┌──────────────┐
│  SQLite Database│◄─────┤  CLI Commands│
│  (Local Store)  │      │  (gh star)   │
└────────┬────────┘      └──────────────┘
         │ export/import
         ▼
┌─────────────────┐
│  YAML/JSON Files│
│  (Version Ctrl) │
└─────────────────┘
```

### Component Details

#### 1. CLI Interface

- Built as GitHub CLI extension using Go
- Entry point: `gh star <command>`
- Follows `gh` CLI patterns and conventions
- Supports both human-friendly and machine-readable output
- Uses `cobra` for command structure

#### 2. Local Database (SQLite)

- **Driver**: `modernc.org/sqlite` (pure-Go, no CGo)
- **Location**: `~/.config/gh-star/stars.db`
- **Features**:
  - Full-text search using FTS5 extension
  - Relational schema for efficient querying
  - WAL mode for better concurrency
  - ACID transactions for data integrity

#### 3. GitHub API Integration

- Uses `gh` CLI for authentication (no separate auth needed)
- Fetches starred repos, metadata, and Star Lists
- Read-only operations (except star/unstar commands)
- Handles rate limits with exponential backoff
- Progress indicators for long-running syncs

#### 4. Export/Import System

- **Formats**: YAML and JSON (user selectable)
- **Structure**: One file per repository for clean git diffs
- **Commands**: Explicit `gh star export` and `gh star import`
- **Purpose**: Version control, backup, sharing collections

## Data Model

### SQLite Schema

#### Table: `repositories`

Stores core GitHub repository data.

**Fields**:
- `id` INTEGER PRIMARY KEY (GitHub repository ID)
- `full_name` TEXT NOT NULL (owner/repo)
- `owner` TEXT NOT NULL
- `name` TEXT NOT NULL
- `description` TEXT
- `url` TEXT NOT NULL
- `homepage` TEXT
- `language` TEXT
- `stars_count` INTEGER
- `forks_count` INTEGER
- `topics` TEXT (JSON array of strings)
- `archived` BOOLEAN
- `starred_at` TEXT (ISO 8601 timestamp)
- `created_at` TEXT (repo creation date)
- `updated_at` TEXT (last GitHub update)
- `synced_at` TEXT (last local sync)

**Indexes**:
- `idx_repositories_language` on `language`
- `idx_repositories_starred_at` on `starred_at`
- `idx_repositories_full_name` on `full_name`

#### Table: `tags`

User-defined tags (both `gh:*` and custom tags).

**Fields**:
- `id` INTEGER PRIMARY KEY AUTOINCREMENT
- `name` TEXT UNIQUE NOT NULL
- `is_github_list` BOOLEAN (true for `gh:*` tags)
- `created_at` TEXT

**Purpose**: The `is_github_list` flag allows distinguishing between GitHub-sourced tags and user-created tags for sync reconciliation.

#### Table: `repository_tags`

Many-to-many relationship between repos and tags.

**Fields**:
- `repository_id` INTEGER NOT NULL
- `tag_id` INTEGER NOT NULL
- `created_at` TEXT
- PRIMARY KEY (`repository_id`, `tag_id`)
- FOREIGN KEY (`repository_id`) REFERENCES `repositories`(`id`) ON DELETE CASCADE
- FOREIGN KEY (`tag_id`) REFERENCES `tags`(`id`) ON DELETE CASCADE

**Indexes**:
- `idx_repository_tags_tag_id` on `tag_id` (for reverse lookups)

#### Table: `repository_metadata`

User-added metadata per repository.

**Fields**:
- `repository_id` INTEGER PRIMARY KEY
- `notes` TEXT (user's notes about the repository)
- `custom_fields` TEXT (JSON object for extensibility)
- `last_modified` TEXT
- FOREIGN KEY (`repository_id`) REFERENCES `repositories`(`id`) ON DELETE CASCADE

#### Full-Text Search Table

FTS5 virtual table for fast text search.

**Fields**:
- `full_name` TEXT
- `description` TEXT
- `topics` TEXT
- `notes` TEXT

**Configuration**:
- Tokenizer: `unicode61`
- Content table: synchronized with `repositories` and `repository_metadata`

### Tag Namespacing Convention

To maintain clear separation between GitHub-managed and user-managed tags:

- **GitHub Lists**: Converted to tags with `gh:` prefix
  - Example: GitHub List "CLI Tools" → tag `gh:cli-tools`
  - Naming: lowercase, spaces replaced with hyphens
- **User Tags**: No prefix
  - Examples: `production`, `priority-1`, `needs-review`

This convention enables:
1. Clear mental model for users
2. Simple sync reconciliation (delete/reapply `gh:*` tags)
3. No conflicts between GitHub and user tags

## GitHub Integration & Sync Strategy

### One-Way Sync with Namespaced Tags

GitHub Star Lists are synchronized as special `gh:*` prefixed tags, creating clear separation between GitHub-managed and user-managed organization.

### Initial Import (`gh star init`)

**Process**:
1. Create local database if it doesn't exist
2. Fetch all starred repositories via GitHub API (paginated)
3. Fetch all Star Lists and their memberships
4. For each Star List:
   - Create a tag with `gh:` prefix
   - Mark as `is_github_list = true`
5. Apply `gh:*` tags to repositories based on list membership
6. Store all data in local SQLite database

**API Calls**:
- `GET /user/starred` (paginated)
- `GET /users/{username}/lists` (list all Star Lists)
- `GET /users/{username}/lists/{list_id}/items` (for each list)

### Ongoing Sync (`gh star sync`)

**Process**:
1. Fetch new starred repositories since last sync
2. Update existing repository metadata (stars count, description, topics)
3. Fetch current Star Lists memberships
4. **Reconciliation of `gh:*` tags**:
   - Delete ALL tags where `is_github_list = true`
   - Re-fetch GitHub Lists and memberships
   - Re-create `gh:*` tags based on current GitHub state
   - Apply tags to repositories
5. Update `synced_at` timestamp

**Key Behaviors**:
- User tags (where `is_github_list = false`) are **never touched**
- Only `gh:*` namespace is managed by sync
- Full reconciliation ensures local state matches GitHub
- No diff/merge logic needed - delete and reapply

### Clear Mental Model

> **"GitHub Lists control `gh:*` tags. Everything else is mine."**

This design provides:
- Predictable sync behavior
- No conflict resolution needed
- Clear ownership of data
- Simple implementation

## Core Features & Commands

### Initialization & Sync

#### `gh star init`

Initialize gh-star, create database, import all stars and GitHub Lists.

**Behavior**:
- Creates `~/.config/gh-star/` directory
- Creates SQLite database
- Fetches all starred repos and lists
- Shows progress bar for large collections

**Flags**:
- `--skip-lists` - Don't import GitHub Star Lists
- `--force` - Reinitialize even if database exists (prompts for confirmation)

#### `gh star sync`

Fetch new stars and update metadata.

**Behavior**:
- Incremental sync (only new stars since last sync)
- Updates repository metadata for existing stars
- Reconciles `gh:*` tags from GitHub Lists
- Shows summary of changes

**Flags**:
- `--full` - Full refresh of all star data (slower)
- `--skip-lists` - Don't sync GitHub Lists

### Search & Discovery

#### `gh star list`

List all starred repositories.

**Output**: One repository per line (default: `owner/repo` format)

**Flags**:
- `--tag TAG` - Filter by tag (repeatable for AND logic)
- `--language LANG` - Filter by programming language
- `--since DATE` - Stars added after date
- `--until DATE` - Stars added before date
- `--archived` - Include/exclude archived repos
- `--sort FIELD` - Sort by: `starred`, `name`, `stars`, `updated`
- `--limit N` - Limit results
- `--json` - Output as JSON array
- `--format FORMAT` - Output format: `simple`, `tsv`, `json`

**Examples**:
```bash
gh star list
gh star list --tag production --language Go
gh star list --since 2024-01-01 --json
```

#### `gh star search <query>`

Full-text search across repository names, descriptions, topics, and notes.

**Behavior**:
- Uses FTS5 for fast full-text search
- Searches across: `full_name`, `description`, `topics`, `notes`
- Supports FTS5 query syntax (phrases, boolean operators)

**Flags**:
- Same filter flags as `list` (can combine search with filters)
- `--interactive` / `-i` - Open fzf for interactive selection

**Examples**:
```bash
gh star search kubernetes
gh star search "docker compose"
gh star search kubernetes --language Go --tag production
```

#### `gh star show <repo>`

Show detailed information about a repository.

**Input**: Repository in format `owner/repo` or GitHub URL

**Output**:
- Repository metadata (description, language, stars, topics)
- User tags (separated: `gh:*` tags and user tags)
- User notes
- GitHub Star Lists membership (if any)
- Starred date

**Flags**:
- `--json` - Output as JSON object

### Tagging & Organization

#### `gh star tag add <repo> <tags...>`

Add one or more tags to a repository.

**Behavior**:
- Creates tags if they don't exist
- Ignores duplicate tags (idempotent)
- Validates tag names (no `gh:` prefix allowed for user tags)

**Examples**:
```bash
gh star tag add golang/go production backend
gh star tag add k8s/kubernetes infrastructure priority-1
```

#### `gh star tag remove <repo> <tags...>`

Remove tags from a repository.

**Behavior**:
- Removes tag associations
- Does NOT delete tag definitions (tags remain for other repos)
- Silently ignores non-existent tags

#### `gh star tags`

List all tags with repository counts.

**Output**: Tag name and count of repos with that tag

**Flags**:
- `--user-only` - Show only user tags (exclude `gh:*`)
- `--github-only` - Show only `gh:*` tags
- `--sort count|name` - Sort by count or alphabetically

#### `gh star note set <repo> <text>`

Add or update notes for a repository.

**Input**:
- `<text>` can be provided as argument or read from stdin if `-`

**Examples**:
```bash
gh star note set golang/go "Core Go repository. Check for security updates."
echo "Production service dependency" | gh star note set myorg/myrepo -
```

#### `gh star note show <repo>`

Display notes for a repository.

**Output**: Notes text, or empty if no notes exist

### Batch Operations

#### `gh star bulk-tag`

Add tags to multiple repositories matching a query.

**Behavior**:
- Combines search/filter logic with tag addition
- Shows confirmation prompt with count before applying

**Flags**:
- All filter flags from `list` command
- `--tags TAGS` - Comma-separated tags to add
- `--yes` / `-y` - Skip confirmation prompt

**Examples**:
```bash
gh star bulk-tag --language Go --tags backend,production
gh star bulk-tag --archived --tags legacy --yes
```

#### `gh star bulk-unstar`

Unstar repositories matching a query.

**Behavior**:
- Removes stars from GitHub (destructive operation)
- Shows confirmation prompt with list of repos
- Removes from local database after successful unstar

**Flags**:
- All filter flags from `list` command
- `--yes` / `-y` - Skip confirmation (dangerous)

**Examples**:
```bash
gh star bulk-unstar --archived --yes
```

### Import/Export

#### `gh star export <path>`

Export star collection to files for version control.

**Behavior**:
- Creates directory structure: `<path>/stars/owner/repo.yaml`
- One file per repository for clean git diffs
- Includes all metadata: GitHub data + user tags + notes
- Creates `metadata.yaml` with export info

**Flags**:
- `--format FORMAT` - Output format: `yaml` (default) or `json`
- `--filter` - Use same filter flags as `list` to export subset

**File Structure**:
```
<path>/
├── stars/
│   ├── golang/
│   │   └── go.yaml
│   ├── kubernetes/
│   │   └── kubernetes.yaml
│   └── ...
└── metadata.yaml
```

**Example**:
```bash
gh star export ~/my-stars
gh star export ~/my-stars --format json
gh star export ~/my-stars --tag production
```

#### `gh star import <path>`

Import star collection from exported files.

**Behavior**:
- Auto-detects format (YAML or JSON)
- **Default: Replace** - Clears database and imports (prompts for confirmation)
- **Merge mode**: Merges with existing data (repos by ID, tags by name)

**Flags**:
- `--merge` - Merge with existing data instead of replacing
- `--yes` / `-y` - Skip confirmation prompt

**Examples**:
```bash
gh star import ~/my-stars
gh star import ~/my-stars --merge
```

## Interactive Mode & Pipeline Integration

### Design Philosophy

**Default behavior**: Pipeline-friendly output for Unix tools
**Opt-in**: Interactive mode via `--interactive` / `-i` flag

### Pipeline-Friendly Output (Default)

Commands output plain text or structured data suitable for piping.

**Output Formats**:

1. **Simple** (default): One repo per line (`owner/repo`)
2. **TSV**: Tab-separated values for `cut`, `awk`, `sort`
3. **JSON**: Structured data for `jq` and programmatic use

**Examples**:
```bash
# Pipe to grep
gh star list | grep kubernetes

# Use with jq
gh star list --json | jq '.[] | select(.language == "Go")'

# Tab-separated for Unix tools
gh star list --format tsv | cut -f1,3 | sort

# Pipe to xargs for batch operations
gh star search docker | xargs -I {} gh star tag add {} container
```

### Interactive Mode (`--interactive` flag)

Uses **fzf** for fuzzy finding and interactive selection.

**Behavior**:
- Opens fzf with repository list
- Fuzzy search across repo names
- Preview pane shows: description, language, stars, tags, notes
- Multi-select support (TAB to select multiple)
- Selected repos written to stdout (can pipe to other commands)

**fzf Integration**:
- Custom preview command: `gh star show {}`
- Key bindings:
  - `TAB` - Select/deselect
  - `CTRL-A` - Select all
  - `ENTER` - Confirm selection
- Graceful fallback if fzf not installed (warning + simple list)

**Examples**:
```bash
# Interactive search with preview
gh star search --interactive

# Interactive tagging workflow
gh star search kubernetes -i | xargs -I {} gh star tag add {} k8s

# Interactive selection for bulk operations
gh star list --tag legacy -i | xargs gh star bulk-unstar
```

### Preview Command

The interactive preview shows:
- Repository full name and URL
- Description
- Language, stars, forks
- Topics
- All tags (grouped: `gh:*` tags | user tags)
- User notes
- Starred date

## Export/Import File Format

### Directory Structure

```
<export-path>/
├── metadata.yaml
└── stars/
    ├── golang/
    │   ├── go.yaml
    │   └── groupcache.yaml
    ├── kubernetes/
    │   └── kubernetes.yaml
    └── ...
```

### Metadata File (`metadata.yaml`)

```yaml
# gh-star export metadata
export_version: "1"
exported_at: "2024-01-15T10:30:00Z"
repository_count: 3247
tag_count: 156
format: yaml
```

### Repository File Format (YAML)

```yaml
# Exported by gh-star
# Repository: golang/go

# GitHub metadata
github_id: 23096959
full_name: golang/go
owner: golang
name: go
description: The Go programming language
url: https://github.com/golang/go
homepage: https://golang.org
language: Go
stars_count: 115234
forks_count: 16892
topics:
  - compiler
  - go
  - programming-language
archived: false
starred_at: "2020-03-15T10:30:00Z"
created_at: "2014-08-19T04:33:40Z"
updated_at: "2024-01-15T08:22:14Z"

# User metadata
tags:
  # From GitHub Lists
  - gh:programming-languages
  - gh:backend-tech
  # User tags
  - backend
  - production
  - compiler-design
notes: |
  Core Go repository. Check releases for security updates.
  Used in production services.
  Relevant issues: #12345, #67890

custom_fields:
  priority: high
  team: backend
  last_reviewed: "2024-01-10"
```

### Repository File Format (JSON)

```json
{
  "github_id": 23096959,
  "full_name": "golang/go",
  "owner": "golang",
  "name": "go",
  "description": "The Go programming language",
  "url": "https://github.com/golang/go",
  "homepage": "https://golang.org",
  "language": "Go",
  "stars_count": 115234,
  "forks_count": 16892,
  "topics": ["compiler", "go", "programming-language"],
  "archived": false,
  "starred_at": "2020-03-15T10:30:00Z",
  "created_at": "2014-08-19T04:33:40Z",
  "updated_at": "2024-01-15T08:22:14Z",
  "tags": [
    "gh:programming-languages",
    "gh:backend-tech",
    "backend",
    "production",
    "compiler-design"
  ],
  "notes": "Core Go repository. Check releases for security updates.\nUsed in production services.\nRelevant issues: #12345, #67890",
  "custom_fields": {
    "priority": "high",
    "team": "backend",
    "last_reviewed": "2024-01-10"
  }
}
```

## Implementation Details

### Technology Stack

**Language**: Go 1.21+

**Key Dependencies**:
- `modernc.org/sqlite` - Pure-Go SQLite driver
- `github.com/cli/go-gh` - GitHub CLI integration
- `github.com/spf13/cobra` - CLI framework
- `gopkg.in/yaml.v3` - YAML support
- Standard library `encoding/json` for JSON

### Database Configuration

**SQLite Pragmas**:
```sql
PRAGMA journal_mode = WAL;
PRAGMA synchronous = NORMAL;
PRAGMA foreign_keys = ON;
PRAGMA temp_store = MEMORY;
```

**Connection Management**:
- Single connection per CLI invocation (no pooling needed)
- Transactions for batch operations
- Automatic database migration on schema changes

### GitHub API Integration

**Authentication**:
- Uses `gh api` commands (inherits authentication from gh CLI)
- No separate token management needed

**Rate Limiting**:
- Check rate limit before operations: `GET /rate_limit`
- Exponential backoff on 429 responses
- Show progress and ETA for large syncs

**Pagination**:
- Handle pagination for starred repos (100 per page)
- Configurable page size for performance tuning

### File System Layout

**Config Directory**: `~/.config/gh-star/` (follows XDG Base Directory spec)

**Files**:
- `stars.db` - SQLite database
- `config.yaml` - User preferences (future: custom fields, aliases, etc.)

### Error Handling

**Principles**:
- Return errors, don't panic
- Provide actionable error messages
- Distinguish between user errors and system errors
- Proper exit codes for scripting

**Error Types**:
- Network errors (GitHub API)
- Database errors (corruption, permissions)
- Validation errors (invalid repo names, tags)
- User errors (repo not found, already starred)

### Testing Strategy

**Unit Tests**:
- Database operations (CRUD for repos, tags, metadata)
- Tag namespace logic
- Search and filter functions
- Export/import serialization

**Integration Tests**:
- Full command execution with test database
- Mock GitHub API responses
- Golden files for output validation
- Cross-platform path handling

**Test Utilities**:
- Fixture database with sample data
- Mock GitHub API server
- Helper functions for test assertions

## Future Enhancements (Out of Scope for v1.0)

These features are explicitly deferred to keep initial implementation focused:

1. **Analytics & Insights**
   - Trending patterns in starred repos
   - Activity tracking (recent commits, releases)
   - Collection statistics and breakdowns
   - Recommendations based on starred repos

2. **Rich TUI**
   - Full terminal UI (like `lazygit`)
   - Panel-based navigation
   - Built-in editor for notes

3. **Advanced Search**
   - Regex support
   - Dependency graph queries
   - Similar repo discovery

4. **Collaboration**
   - Share collections with others
   - Merge collections from multiple users
   - Team star management

5. **Sync Enhancements**
   - Two-way sync with GitHub Lists (write-back)
   - Conflict resolution for competing edits
   - Webhook support for real-time updates

6. **Custom Fields**
   - User-defined schema for custom_fields
   - Validation and constraints
   - Search/filter on custom fields
