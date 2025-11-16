# gh-star: Implementation Plan

## Overview

This document provides a comprehensive, step-by-step implementation plan for building gh-star. The plan is organized into phases, with each phase containing bite-sized tasks. Tasks are ordered to minimize dependencies and enable incremental testing.

**Target Audience**: Experienced Go developer, first-time contributor to this codebase.

**Principles**:
- Follow TDD (Test-Driven Development) where practical
- Atomic, self-contained commits for each task
- DRY (Don't Repeat Yourself)
- YAGNI (You Aren't Gonna Need It)

## Project Structure

```
gh-star/
├── main.go                      # CLI entry point
├── go.mod                       # Go module definition
├── go.sum                       # Dependency checksums
├── cmd/                         # Command implementations
│   ├── root.go                  # Root command setup
│   ├── init.go                  # init command
│   ├── sync.go                  # sync command
│   ├── list.go                  # list command
│   ├── search.go                # search command
│   ├── show.go                  # show command
│   ├── tag.go                   # tag add/remove/list commands
│   ├── note.go                  # note set/show commands
│   ├── bulk.go                  # bulk operations
│   ├── export.go                # export command
│   └── import.go                # import command
├── internal/                    # Internal packages
│   ├── config/                  # Configuration management
│   │   ├── config.go
│   │   └── paths.go
│   ├── db/                      # Database layer
│   │   ├── db.go                # Connection management
│   │   ├── schema.go            # Schema and migrations
│   │   ├── repository.go        # Repository CRUD
│   │   ├── tag.go               # Tag CRUD
│   │   ├── metadata.go          # Metadata CRUD
│   │   └── search.go            # Search and filter
│   ├── github/                  # GitHub API integration
│   │   ├── client.go            # API client wrapper
│   │   ├── stars.go             # Fetch starred repos
│   │   └── lists.go             # Fetch Star Lists
│   ├── export/                  # Export/import functionality
│   │   ├── export.go            # Export logic
│   │   ├── import.go            # Import logic
│   │   └── format.go            # YAML/JSON serialization
│   ├── interactive/             # Interactive mode
│   │   └── fzf.go               # fzf integration
│   └── output/                  # Output formatting
│       ├── formatter.go         # Format interface
│       ├── simple.go            # Simple text output
│       ├── json.go              # JSON output
│       └── tsv.go               # TSV output
├── testdata/                    # Test fixtures
│   ├── fixtures/                # Sample databases
│   ├── mocks/                   # Mock GitHub API responses
│   └── golden/                  # Golden file outputs
└── docs/                        # Documentation
    └── wip/gh-star-core/        # This design doc
```

## Phase 1: Project Setup & Foundation

### Task 1.1: Initialize Go Module and Project Structure

**Goal**: Set up Go module, directory structure, and basic tooling.

**Steps**:
1. Initialize Go module: `go mod init github.com/serpro69/gh-star`
2. Create directory structure as outlined above
3. Create `.gitignore` for Go projects (binaries, `vendor/`, IDE files)
4. Set Go version to 1.21+ in `go.mod`

**Files to Create**:
- `go.mod`
- `.gitignore` (update existing if present)
- All directories in structure

**Testing**: `go mod verify` should succeed.

---

### Task 1.2: Set Up CLI Framework with Cobra

**Goal**: Create basic CLI structure with root command and version.

**Dependencies**:
```bash
go get github.com/spf13/cobra@latest
go get github.com/cli/go-gh@latest
```

**Files to Create/Modify**:
- `main.go` - Entry point that calls root command
- `cmd/root.go` - Root command setup

**What to Implement**:
- Root command with description
- Version flag that prints version info
- Help text and usage
- Global flags for common options (e.g., `--verbose`)

**Testing**:
- `go run main.go --version` should print version
- `go run main.go --help` should show usage
- Build binary: `go build -o gh-star`

---

### Task 1.3: Configuration Management

**Goal**: Implement config directory and paths following XDG spec.

**Files to Create**:
- `internal/config/config.go` - Config struct and load/save functions
- `internal/config/paths.go` - Path resolution (XDG-compliant)

**What to Implement**:
- `GetConfigDir()` - Returns `~/.config/gh-star/` (with XDG override)
- `GetDatabasePath()` - Returns path to `stars.db`
- `GetConfigPath()` - Returns path to `config.yaml`
- `EnsureConfigDir()` - Creates config directory if missing
- Basic config struct (can be empty for now, will extend later)

**Key Considerations**:
- Windows support (`%APPDATA%/gh-star/`)
- macOS support (`~/Library/Application Support/gh-star/` if not XDG)
- Linux support (XDG Base Directory spec)
- Handle `~` expansion and absolute paths

**Testing**:
- Unit tests for path functions on different platforms
- Test directory creation

---

## Phase 2: Database Layer

### Task 2.1: Database Schema and Migrations

**Goal**: Define SQLite schema and implement migration logic.

**Dependencies**:
```bash
go get modernc.org/sqlite@latest
```

**Files to Create**:
- `internal/db/schema.go` - Schema definition and migrations
- `internal/db/db.go` - Database connection management

**What to Implement in `schema.go`**:
- Schema version tracking table: `schema_version (version INTEGER PRIMARY KEY)`
- SQL DDL for all tables:
  - `repositories`
  - `tags`
  - `repository_tags`
  - `repository_metadata`
  - FTS5 virtual table
- Migration functions:
  - `GetCurrentSchemaVersion(db)` - Returns current version
  - `Migrate(db)` - Applies migrations up to latest version
  - `CreateSchema(db)` - Creates all tables (version 1)
- Pragmas setup function: `ConfigureDatabase(db)`

**What to Implement in `db.go`**:
- `Open(dbPath string) (*sql.DB, error)` - Opens database with pragmas
- `Close(db *sql.DB) error` - Closes database
- Apply pragmas on open: `journal_mode=WAL`, `foreign_keys=ON`, etc.

**Key Considerations**:
- Use `modernc.org/sqlite` driver (pure Go, no CGo)
- Enable foreign keys and WAL mode
- FTS5 table should sync with repositories and repository_metadata
- Add proper indexes as defined in design

**Testing**:
- Test database creation in temporary directory
- Verify schema version after migration
- Test pragma settings
- Verify foreign key constraints work

---

### Task 2.2: Repository CRUD Operations

**Goal**: Implement database operations for repositories table.

**Files to Create**:
- `internal/db/repository.go` - Repository struct and CRUD functions

**What to Implement**:
- `Repository` struct matching database schema
- `CreateRepository(db, repo) error` - Insert repository
- `GetRepository(db, id) (*Repository, error)` - Get by GitHub ID
- `GetRepositoryByName(db, fullName) (*Repository, error)` - Get by owner/repo
- `ListRepositories(db, filters) ([]*Repository, error)` - List with filters
- `UpdateRepository(db, repo) error` - Update existing repository
- `DeleteRepository(db, id) error` - Delete repository
- `UpsertRepository(db, repo) error` - Insert or update

**Filter Types**:
- `RepositoryFilter` struct with fields: `Language`, `Tags`, `Since`, `Until`, `Archived`, `Limit`, `Sort`
- Build WHERE clauses dynamically based on filter

**Key Considerations**:
- Handle JSON fields (`topics` array) properly
- Use prepared statements
- Return `sql.ErrNoRows` as appropriate
- Atomic upsert operations

**Testing**:
- Test CRUD operations on test database
- Test filtering by various criteria
- Test pagination and sorting
- Test JSON field serialization/deserialization

---

### Task 2.3: Tag CRUD Operations

**Goal**: Implement database operations for tags and repository_tags.

**Files to Create**:
- `internal/db/tag.go` - Tag struct and CRUD functions

**What to Implement**:
- `Tag` struct matching database schema
- `CreateTag(db, name, isGitHubList) error` - Insert tag
- `GetOrCreateTag(db, name, isGitHubList) (tagID, error)` - Get or create
- `ListTags(db) ([]*Tag, error)` - List all tags with counts
- `DeleteTag(db, tagID) error` - Delete tag (cascades to repository_tags)
- `AddTagToRepository(db, repoID, tagID) error` - Add association
- `RemoveTagFromRepository(db, repoID, tagID) error` - Remove association
- `GetTagsForRepository(db, repoID) ([]*Tag, error)` - Get repo's tags
- `GetRepositoriesForTag(db, tagID) ([]*Repository, error)` - Get tagged repos
- `DeleteGitHubListTags(db) error` - Delete all `is_github_list = true` tags

**Key Considerations**:
- Tag names should be unique (handled by schema constraint)
- `is_github_list` flag for namespaced tags
- Cascade deletes via foreign keys
- Idempotent operations (adding existing tag is no-op)

**Testing**:
- Test tag creation and lookup
- Test tag-repository associations
- Test cascade deletes
- Test GitHub list tag cleanup

---

### Task 2.4: Metadata CRUD Operations

**Goal**: Implement database operations for repository_metadata.

**Files to Create**:
- `internal/db/metadata.go` - Metadata operations

**What to Implement**:
- `RepositoryMetadata` struct
- `GetMetadata(db, repoID) (*RepositoryMetadata, error)` - Get metadata
- `SetNotes(db, repoID, notes) error` - Set notes for repository
- `SetCustomFields(db, repoID, fields) error` - Set custom JSON fields
- `UpdateMetadata(db, repoID, metadata) error` - Update metadata

**Key Considerations**:
- Handle JSON serialization for `custom_fields`
- Update `last_modified` timestamp
- Sync FTS5 table when notes change

**Testing**:
- Test notes CRUD
- Test custom fields with various JSON structures
- Test FTS5 synchronization

---

### Task 2.5: Search and Filter Implementation

**Goal**: Implement full-text search and advanced filtering.

**Files to Create**:
- `internal/db/search.go` - Search and filter functions

**What to Implement**:
- `SearchRepositories(db, query, filters) ([]*Repository, error)` - FTS5 search
- Build FTS5 query from search string
- Combine FTS5 search with filter conditions
- Support sorting and pagination

**Search Features**:
- Full-text search across: full_name, description, topics, notes
- FTS5 query syntax support (phrases, boolean operators)
- Ranking by relevance

**Key Considerations**:
- Properly escape FTS5 special characters
- Combine FTS5 with JOIN to tags for tag filtering
- Return repositories with highlighted snippets (optional for v1)

**Testing**:
- Test search across different fields
- Test phrase queries
- Test combination of search + filters
- Test relevance ranking

---

## Phase 3: GitHub API Integration

### Task 3.1: GitHub API Client Wrapper

**Goal**: Create wrapper around `gh api` for making GitHub API calls.

**Files to Create**:
- `internal/github/client.go` - API client and common functions

**What to Implement**:
- `Client` struct with rate limit tracking
- `NewClient() (*Client, error)` - Create client using `gh api`
- `APIGet(path string, result interface{}) error` - Generic GET request
- `CheckRateLimit() (remaining, limit, resetTime)` - Check rate limit status
- `WaitForRateLimit()` - Wait if rate limit exhausted
- Error handling for API responses

**Key Considerations**:
- Use `github.com/cli/go-gh` for authentication
- Parse rate limit headers from responses
- Exponential backoff on 429 status
- Progress indicators for long operations

**Testing**:
- Mock GitHub API responses
- Test rate limit handling
- Test error responses (404, 403, 500)

---

### Task 3.2: Fetch Starred Repositories

**Goal**: Implement fetching of starred repositories from GitHub.

**Files to Create**:
- `internal/github/stars.go` - Stars API functions

**What to Implement**:
- `FetchStarredRepos(client, since) ([]*Repository, error)` - Fetch all stars
- Pagination handling (100 repos per page)
- Parse GitHub API response to `Repository` struct
- Support incremental sync with `since` parameter
- Progress callback for UI updates

**API Endpoint**: `GET /user/starred`

**Query Parameters**:
- `per_page=100`
- `page=N`
- `since=TIMESTAMP` (for incremental sync)

**Key Considerations**:
- Map GitHub API response fields to `Repository` struct
- Parse `topics` array
- Handle pagination with Link header
- Convert timestamp formats (ISO 8601)

**Testing**:
- Test with mock paginated responses
- Test incremental sync with `since` parameter
- Test rate limit handling during pagination

---

### Task 3.3: Fetch GitHub Star Lists

**Goal**: Implement fetching of Star Lists and memberships.

**Files to Create**:
- `internal/github/lists.go` - Star Lists API functions

**What to Implement**:
- `FetchStarLists(client) ([]*StarList, error)` - Fetch all Star Lists
- `FetchListItems(client, listID) ([]int64, error)` - Get repo IDs in list
- Map list names to tag names (lowercase, hyphenate spaces)
- `StarList` struct with ID and name

**API Endpoints**:
- `GET /users/{username}/lists` - List all Star Lists
- `GET /users/{username}/lists/{list_id}/items` - List items in a list

**Key Considerations**:
- Need to get authenticated username first
- Pagination for list items
- Convert list names to tag format (`gh:list-name`)
- Handle lists with no items

**Testing**:
- Test with mock list responses
- Test name conversion logic
- Test empty lists

---

## Phase 4: Core Commands (Part 1: Setup & Sync)

### Task 4.1: Implement `init` Command

**Goal**: Initialize gh-star, create database, import stars and lists.

**Files to Create**:
- `cmd/init.go` - Init command implementation

**What to Implement**:
- Check if database already exists (prompt for `--force`)
- Create config directory
- Create and initialize database
- Fetch all starred repos from GitHub
- Fetch all Star Lists and memberships
- Import repos into database
- Create `gh:*` tags for lists
- Show progress bar for large operations
- Print summary when complete

**Flags**:
- `--skip-lists` - Don't import GitHub Star Lists
- `--force` - Reinitialize even if database exists

**Progress Indicators**:
- Show spinner/progress during GitHub API calls
- Show count of repos imported
- Show count of lists imported

**Key Considerations**:
- Handle existing database gracefully
- Transaction for bulk inserts
- Error recovery (don't corrupt database on failure)
- Clear user feedback throughout process

**Testing**:
- Test first-time init
- Test re-init with `--force`
- Test `--skip-lists` flag
- Test error handling (network failures, API errors)

---

### Task 4.2: Implement `sync` Command

**Goal**: Incremental sync of stars and lists.

**Files to Create**:
- `cmd/sync.go` - Sync command implementation

**What to Implement**:
- Check last sync time from database metadata
- Fetch new stars since last sync
- Update existing repos (stars count, description, etc.)
- Fetch current Star Lists memberships
- Reconcile `gh:*` tags:
  - Delete all `is_github_list = true` tags
  - Re-fetch lists and re-create tags
  - Re-apply to repositories
- Update last sync timestamp
- Show summary of changes

**Flags**:
- `--full` - Full refresh (re-fetch all repos)
- `--skip-lists` - Don't sync Star Lists

**Summary Output**:
- Number of new repos added
- Number of repos updated
- Number of `gh:*` tags changed
- Last sync time

**Key Considerations**:
- Use transactions for atomic updates
- Handle repos that were unstarred (remove from DB)
- Preserve user tags and metadata during sync
- Error recovery (rollback on failure)

**Testing**:
- Test incremental sync with new stars
- Test full refresh
- Test tag reconciliation
- Test error handling and rollback

---

## Phase 5: Core Commands (Part 2: Search & Display)

### Task 5.1: Implement `list` Command

**Goal**: List repositories with filtering and sorting.

**Files to Create**:
- `cmd/list.go` - List command implementation
- `internal/output/formatter.go` - Output formatter interface
- `internal/output/simple.go` - Simple text formatter
- `internal/output/json.go` - JSON formatter
- `internal/output/tsv.go` - TSV formatter

**What to Implement**:
- Parse filter flags into `RepositoryFilter` struct
- Query database with filters
- Format output based on `--format` flag
- Default: simple output (one `owner/repo` per line)

**Flags**:
- `--tag TAG` (repeatable) - Filter by tags (AND logic)
- `--language LANG` - Filter by language
- `--since DATE` - Starred after date
- `--until DATE` - Starred before date
- `--archived` - Include/exclude archived
- `--sort FIELD` - Sort by: `starred`, `name`, `stars`, `updated`
- `--limit N` - Limit results
- `--format FORMAT` - Output format: `simple`, `tsv`, `json`

**Output Formats**:
- **Simple**: `owner/repo` (one per line)
- **TSV**: Tab-separated: `full_name\tlanguage\tstars_count\tstarred_at`
- **JSON**: Array of repository objects

**Key Considerations**:
- Date parsing (support various formats)
- Multiple tag filters (AND logic)
- Proper sorting and pagination
- Handle empty results gracefully

**Testing**:
- Test each filter flag independently
- Test combinations of filters
- Test each output format
- Test sorting and limiting

---

### Task 5.2: Implement `search` Command

**Goal**: Full-text search with filters.

**Files to Create**:
- `cmd/search.go` - Search command implementation

**What to Implement**:
- Parse search query and filters
- Call `SearchRepositories` from db layer
- Format output (reuse output formatters from `list`)
- Support all filter flags from `list` command

**Arguments**:
- `<query>` - Search query (FTS5 syntax)

**Flags**:
- All filter flags from `list` command
- `--interactive` / `-i` - Launch fzf (deferred to later task)

**Search Behavior**:
- Search across: full_name, description, topics, notes
- Support FTS5 syntax (phrases, boolean operators)
- Return results sorted by relevance

**Key Considerations**:
- Escape FTS5 special characters in user input
- Clear error messages for invalid FTS5 queries
- Performance with large result sets

**Testing**:
- Test simple keyword search
- Test phrase search ("exact phrase")
- Test boolean operators (AND, OR, NOT)
- Test combination with filters

---

### Task 5.3: Implement `show` Command

**Goal**: Display detailed information about a single repository.

**Files to Create**:
- `cmd/show.go` - Show command implementation

**What to Implement**:
- Parse repository identifier (accept `owner/repo` or GitHub URL)
- Fetch repository from database
- Fetch tags for repository
- Fetch metadata (notes, custom fields)
- Format detailed output

**Arguments**:
- `<repo>` - Repository in format `owner/repo` or GitHub URL

**Flags**:
- `--json` - Output as JSON object

**Output (Human-Readable)**:
```
golang/go
The Go programming language

Language: Go
Stars: 115,234 | Forks: 16,892
Starred: 2020-03-15
Topics: compiler, go, programming-language

GitHub Lists:
  gh:programming-languages
  gh:backend-tech

User Tags:
  backend, production, compiler-design

Notes:
  Core Go repository. Check releases for security updates.
  Used in production services.

URL: https://github.com/golang/go
```

**Key Considerations**:
- Handle repo not found error
- Distinguish GitHub list tags from user tags in output
- Format dates in human-readable format
- Parse GitHub URLs to extract `owner/repo`

**Testing**:
- Test with valid repository
- Test with non-existent repository
- Test URL parsing
- Test JSON output format

---

## Phase 6: Core Commands (Part 3: Tagging & Notes)

### Task 6.1: Implement `tag add` Command

**Goal**: Add tags to a repository.

**Files to Create**:
- `cmd/tag.go` - Tag subcommands (add, remove, list)

**What to Implement for `tag add`**:
- Parse repository and tag names from arguments
- Validate tags (no `gh:` prefix allowed)
- Get or create tags
- Add tag associations to repository
- Handle repo not found error

**Usage**: `gh star tag add <repo> <tag1> [tag2] [tag3]...`

**Key Considerations**:
- Idempotent (adding existing tag is no-op)
- Validate tag names (alphanumeric, hyphens, underscores)
- Reject `gh:` prefix (reserved for GitHub lists)
- Multiple tags in single command

**Testing**:
- Test adding single tag
- Test adding multiple tags
- Test idempotency
- Test validation (reject `gh:` tags)

---

### Task 6.2: Implement `tag remove` Command

**Goal**: Remove tags from a repository.

**What to Implement for `tag remove`**:
- Parse repository and tag names
- Remove tag associations
- Silently ignore non-existent tags

**Usage**: `gh star tag remove <repo> <tag1> [tag2]...`

**Key Considerations**:
- Don't delete tag definition (keep for other repos)
- Handle non-existent tags gracefully
- Multiple tags in single command

**Testing**:
- Test removing existing tags
- Test removing non-existent tags
- Test removing multiple tags

---

### Task 6.3: Implement `tags` Command (List All Tags)

**Goal**: List all tags with repository counts.

**What to Implement for `tags` (or `tag list`)**:
- Query all tags with counts
- Sort by name or count
- Filter by tag type

**Usage**: `gh star tags` or `gh star tag list`

**Flags**:
- `--user-only` - Show only user tags
- `--github-only` - Show only `gh:*` tags
- `--sort count|name` - Sort order

**Output Format**:
```
gh:backend-tech (45)
gh:programming-languages (23)
backend (67)
production (34)
priority-1 (12)
```

**Testing**:
- Test listing all tags
- Test filtering by type
- Test sorting

---

### Task 6.4: Implement `note set` Command

**Goal**: Set notes for a repository.

**Files to Create**:
- `cmd/note.go` - Note subcommands (set, show)

**What to Implement for `note set`**:
- Parse repository and note text
- Support reading from stdin if text is `-`
- Update repository metadata with notes
- Update `last_modified` timestamp

**Usage**:
- `gh star note set <repo> "note text"`
- `echo "note text" | gh star note set <repo> -`

**Key Considerations**:
- Multi-line notes support
- Overwrite existing notes (not append)
- Handle empty notes (clear notes)

**Testing**:
- Test setting notes
- Test reading from stdin
- Test overwriting notes
- Test multi-line notes

---

### Task 6.5: Implement `note show` Command

**Goal**: Display notes for a repository.

**What to Implement for `note show`**:
- Parse repository identifier
- Fetch notes from metadata
- Display notes or message if no notes exist

**Usage**: `gh star note show <repo>`

**Output**:
- If notes exist: Print notes text
- If no notes: Print "(no notes)"

**Testing**:
- Test showing existing notes
- Test showing repo with no notes

---

## Phase 7: Batch Operations

### Task 7.1: Implement `bulk-tag` Command

**Goal**: Add tags to multiple repositories matching filters.

**Files to Create**:
- `cmd/bulk.go` - Bulk operations (bulk-tag, bulk-unstar)

**What to Implement for `bulk-tag`**:
- Parse filter flags (reuse from `list` command)
- Parse tags to add
- Query repositories matching filters
- Show confirmation prompt with count
- Add tags to all matching repos in transaction
- Show summary

**Usage**: `gh star bulk-tag --language Go --tags backend,production`

**Flags**:
- All filter flags from `list`
- `--tags TAGS` - Comma-separated tags to add
- `--yes` / `-y` - Skip confirmation

**Confirmation Prompt**:
```
About to add tags [backend, production] to 156 repositories.
Continue? [y/N]
```

**Key Considerations**:
- Transaction for atomicity
- Progress indicator for large operations
- Validate tags before prompting
- Rollback on error

**Testing**:
- Test with filters
- Test confirmation prompt
- Test `--yes` flag
- Test transaction rollback on error

---

### Task 7.2: Implement `bulk-unstar` Command

**Goal**: Unstar repositories matching filters (destructive operation).

**What to Implement for `bulk-unstar`**:
- Parse filter flags
- Query repositories matching filters
- Show list of repos to unstar with confirmation
- Call GitHub API to unstar each repo
- Remove from local database after successful unstar
- Show summary with successes/failures

**Usage**: `gh star bulk-unstar --archived --yes`

**Flags**:
- All filter flags from `list`
- `--yes` / `-y` - Skip confirmation

**Confirmation Prompt**:
```
About to UNSTAR the following repositories:
  owner1/repo1
  owner2/repo2
  ... (showing first 10, 156 total)

This action is destructive and will remove stars from GitHub.
Continue? [y/N]
```

**Key Considerations**:
- Very clear confirmation (show repo list)
- Handle API failures gracefully (continue on error)
- Don't remove from DB if GitHub API fails
- Rate limit handling
- Progress indicator

**Testing**:
- Test with mock GitHub API
- Test confirmation prompt
- Test error handling (API failures)
- Test partial success (some unstar, some fail)

---

## Phase 8: Export/Import

### Task 8.1: Export Serialization

**Goal**: Implement serialization to YAML and JSON formats.

**Files to Create**:
- `internal/export/format.go` - Serialization functions
- `internal/export/export.go` - Export logic

**What to Implement in `format.go`**:
- `ExportedRepository` struct (for YAML/JSON representation)
- `SerializeRepository(repo, tags, metadata) (*ExportedRepository, error)`
- `WriteYAML(repo, filePath) error` - Write repo to YAML file
- `WriteJSON(repo, filePath) error` - Write repo to JSON file
- `WriteMetadataFile(path, metadata) error` - Write metadata.yaml/json

**Export Metadata Struct**:
- `ExportVersion` - Format version ("1")
- `ExportedAt` - Timestamp
- `RepositoryCount` - Total repos exported
- `TagCount` - Total unique tags
- `Format` - "yaml" or "json"

**Key Considerations**:
- YAML comments for readability
- Consistent field ordering
- Proper indentation
- Handle special characters in notes

**Testing**:
- Test YAML serialization
- Test JSON serialization
- Test special characters and multi-line strings
- Test roundtrip (serialize + deserialize)

---

### Task 8.2: Implement `export` Command

**Goal**: Export star collection to directory.

**Files to Create**:
- `cmd/export.go` - Export command implementation

**What to Implement**:
- Parse target directory path
- Query repositories (with optional filters)
- Create directory structure: `<path>/stars/owner/`
- Serialize each repo to file
- Write metadata file
- Show progress bar

**Usage**: `gh star export ~/my-stars`

**Flags**:
- `--format FORMAT` - Output format: `yaml` (default) or `json`
- All filter flags from `list` (to export subset)

**Directory Structure**:
```
~/my-stars/
├── metadata.yaml
└── stars/
    ├── golang/
    │   └── go.yaml
    └── kubernetes/
        └── kubernetes.yaml
```

**Key Considerations**:
- Create directories as needed
- Handle existing files (overwrite or prompt?)
- Progress indicator for large exports
- Atomic write (temp file + rename)

**Testing**:
- Test full export
- Test filtered export
- Test both formats
- Test directory creation
- Test overwriting existing export

---

### Task 8.3: Import Deserialization

**Goal**: Implement deserialization from YAML and JSON formats.

**Files to Create**:
- `internal/export/import.go` - Import logic

**What to Implement**:
- `ReadYAML(filePath) (*ExportedRepository, error)` - Read YAML file
- `ReadJSON(filePath) (*ExportedRepository, error)` - Read JSON file
- `AutoDetectFormat(filePath) (format, error)` - Detect YAML vs JSON
- `DeserializeRepository(exported) (repo, tags, metadata, error)`
- Walk directory structure and collect all files

**Key Considerations**:
- Auto-detect format from file extension or content
- Validate required fields
- Handle missing optional fields gracefully
- Report files that fail to parse (continue with others)

**Testing**:
- Test YAML deserialization
- Test JSON deserialization
- Test auto-detection
- Test malformed files
- Test missing fields

---

### Task 8.4: Implement `import` Command

**Goal**: Import star collection from directory.

**Files to Create**:
- `cmd/import.go` - Import command implementation

**What to Implement**:
- Parse source directory path
- Read metadata file
- Walk directory and deserialize repos
- **Replace mode** (default):
  - Show confirmation prompt
  - Clear database
  - Import all repos
- **Merge mode** (`--merge` flag):
  - Import repos by ID (upsert)
  - Merge tags (union)
  - Preserve existing user metadata
- Show progress bar
- Show summary

**Usage**: `gh star import ~/my-stars`

**Flags**:
- `--merge` - Merge with existing data
- `--yes` / `-y` - Skip confirmation

**Confirmation Prompt (Replace Mode)**:
```
About to REPLACE local database with import.
Current database has 3,247 repositories.
Import contains 3,189 repositories.

This will DELETE all local data. Continue? [y/N]
```

**Key Considerations**:
- Very clear warning for replace mode
- Transaction for atomicity
- Error recovery (rollback on failure)
- Handle corrupted files (skip, report)
- Merge mode tag handling (union, not replace)

**Testing**:
- Test replace mode
- Test merge mode
- Test confirmation prompt
- Test error handling
- Test transaction rollback

---

## Phase 9: Interactive Mode

### Task 9.1: fzf Integration

**Goal**: Implement interactive mode with fzf for search and list commands.

**Files to Create**:
- `internal/interactive/fzf.go` - fzf integration

**What to Implement**:
- `IsFzfAvailable() bool` - Check if fzf is installed
- `LaunchFzf(repos, previewCmd) ([]string, error)` - Launch fzf with repos
- Build fzf options:
  - Preview command
  - Multi-select mode
  - Key bindings
- Parse fzf output (selected repos)

**fzf Configuration**:
- Preview command: `gh star show {}`
- Multi-select with TAB
- Return selected items to stdout

**Key Considerations**:
- Graceful fallback if fzf not installed (warning + simple list)
- Handle empty selection (user canceled)
- Pass environment variables to fzf
- Customize fzf appearance (colors, layout)

**Testing**:
- Test with mock fzf (simulate selection)
- Test fallback when fzf not available
- Test multi-select
- Test cancellation

---

### Task 9.2: Add Interactive Flag to Commands

**Goal**: Add `--interactive` flag to `list` and `search` commands.

**Files to Modify**:
- `cmd/list.go`
- `cmd/search.go`

**What to Implement**:
- Add `--interactive` / `-i` flag
- When flag is set:
  - Check if fzf is available
  - Query repositories normally
  - Launch fzf with results
  - Output selected repos to stdout (for piping)

**Behavior**:
```bash
# Interactive search, then pipe to tag command
gh star search kubernetes -i | xargs -I {} gh star tag add {} k8s
```

**Testing**:
- Test interactive mode in both commands
- Test piping output to other commands
- Test fallback behavior

---

## Phase 10: Polish & Documentation

### Task 10.1: Error Messages and User Feedback

**Goal**: Improve error messages and user feedback throughout CLI.

**What to Improve**:
- Consistent error message format
- Actionable error messages (tell user how to fix)
- Progress indicators for long operations
- Success messages with summaries
- Proper exit codes

**Error Message Examples**:
- Bad: "Database error"
- Good: "Failed to open database at ~/.config/gh-star/stars.db: permission denied. Try running: chmod 644 ~/.config/gh-star/stars.db"

**Testing**:
- Review all error paths
- Test with invalid inputs
- Test network failures
- Verify exit codes

---

### Task 10.2: Command Documentation

**Goal**: Add help text and examples to all commands.

**What to Add**:
- Detailed command descriptions
- Flag descriptions
- Usage examples in help text
- Common use cases

**Files to Modify**:
- All `cmd/*.go` files

**Example Help Text**:
```
Search for repositories across names, descriptions, topics, and notes.

Usage:
  gh star search <query> [flags]

Examples:
  # Simple search
  gh star search kubernetes

  # Search with filters
  gh star search docker --language Go --tag production

  # Interactive search with fzf
  gh star search kubernetes -i

Flags:
  -i, --interactive    Launch fzf for interactive selection
      --tag strings    Filter by tags (repeatable)
      --language       Filter by programming language
  ...
```

---

### Task 10.3: README and User Documentation

**Goal**: Create comprehensive README with installation, usage, and examples.

**Files to Create/Update**:
- `README.md` - Main documentation
- `docs/commands.md` - Command reference (optional)
- `docs/examples.md` - Usage examples (optional)

**README Sections**:
1. Overview and features
2. Installation (from source, via `gh extension install`)
3. Quick start guide
4. Command reference (brief)
5. Examples and workflows
6. Configuration
7. Contributing
8. License

**Key Examples to Include**:
- First-time setup
- Daily workflows
- Searching and filtering
- Tagging and organization
- Export for backup
- Integration with other tools (fzf, jq)

---

### Task 10.4: Testing and Quality Assurance

**Goal**: Ensure comprehensive test coverage and quality.

**Testing Checklist**:
- [ ] Unit tests for all database operations
- [ ] Unit tests for all formatters
- [ ] Integration tests for commands
- [ ] Test cross-platform behavior (paths, file handling)
- [ ] Test with large datasets (3000+ repos)
- [ ] Test edge cases (empty database, network failures)
- [ ] Test interactive mode (with mock fzf)

**Quality Checks**:
- [ ] Run `go vet`
- [ ] Run `golangci-lint`
- [ ] Check test coverage: `go test -cover ./...`
- [ ] Memory profiling for large datasets
- [ ] Cross-compile for all platforms

---

## Phase 11: Build and Release

### Task 11.1: Build System

**Goal**: Set up build scripts and cross-compilation.

**Files to Create**:
- `Makefile` - Build tasks
- `.goreleaser.yml` - Release automation (optional)

**Makefile Targets**:
```makefile
.PHONY: build test install clean

build:
	go build -o gh-star

test:
	go test -v ./...

install:
	go build -o gh-star
	gh extension install .

clean:
	rm -f gh-star
```

**Cross-Compilation**:
- Build for: Linux (amd64, arm64), macOS (amd64, arm64), Windows (amd64)
- Use `goreleaser` or manual `GOOS/GOARCH` builds

---

### Task 11.2: GitHub Actions CI

**Goal**: Set up continuous integration for testing and builds.

**Files to Create**:
- `.github/workflows/ci.yml` - CI workflow
- `.github/workflows/release.yml` - Release workflow

**CI Workflow**:
- Run on: push, pull requests
- Steps:
  - Checkout code
  - Setup Go
  - Run tests
  - Run linters
  - Build binary

**Release Workflow**:
- Triggered on: tag push (e.g., `v1.0.0`)
- Use `goreleaser` to:
  - Cross-compile for all platforms
  - Create GitHub release
  - Upload binaries

---

## Implementation Order and Dependencies

### Recommended Implementation Order

1. **Phase 1** (Foundation) → No dependencies
2. **Phase 2** (Database) → Depends on Phase 1
3. **Phase 3** (GitHub API) → Depends on Phase 1
4. **Phase 4** (Init/Sync) → Depends on Phases 1-3
5. **Phase 5** (Search/Display) → Depends on Phases 1-2
6. **Phase 6** (Tagging/Notes) → Depends on Phases 1-2
7. **Phase 7** (Batch Ops) → Depends on Phases 1-3, 5-6
8. **Phase 8** (Export/Import) → Depends on Phases 1-2
9. **Phase 9** (Interactive) → Depends on Phase 5
10. **Phase 10** (Polish) → Depends on all previous phases
11. **Phase 11** (Build/Release) → Final phase

### Parallelization Opportunities

These phases can be worked on in parallel by different developers:

- **Phase 5** (Search/Display) and **Phase 6** (Tagging/Notes) - Independent after Phase 2
- **Phase 7** (Batch Ops) and **Phase 8** (Export/Import) - Independent after Phase 6
- **Phase 9** (Interactive) can start after Phase 5 task 5.1

## Testing Strategy

### Unit Tests

Each package should have comprehensive unit tests:

- `internal/config/*` - Path resolution, config loading
- `internal/db/*` - CRUD operations, search, filters
- `internal/github/*` - API calls (with mocks)
- `internal/export/*` - Serialization/deserialization
- `internal/output/*` - Output formatters

### Integration Tests

Test commands end-to-end with test databases:

- Create test fixtures in `testdata/fixtures/`
- Use temporary databases for each test
- Mock GitHub API responses
- Test command outputs with golden files

### Test Fixtures

Create reusable test data:

```
testdata/
├── fixtures/
│   ├── small.db (10 repos)
│   ├── medium.db (100 repos)
│   └── large.db (1000 repos)
├── mocks/
│   ├── starred_repos.json
│   ├── star_lists.json
│   └── list_items.json
└── golden/
    ├── list_output.txt
    ├── search_output.json
    └── show_output.txt
```

### Performance Testing

Test with large datasets:

- Create test database with 3000+ repos
- Measure query performance
- Profile memory usage
- Test pagination and limits

## Commit Strategy

Follow atomic commit principles:

**Good Commit Examples**:
- "Add database schema and migration logic"
- "Implement repository CRUD operations"
- "Add fzf integration for interactive mode"

**Bad Commit Examples**:
- "WIP" or "Fix stuff"
- "Update files" (too vague)
- "Implement multiple commands" (too large)

**Commit Message Format**:
```
<type>: <short summary>

<detailed description if needed>

<reference to issues/PRs if applicable>
```

**Types**: `feat`, `fix`, `refactor`, `test`, `docs`, `chore`

## Resources and References

### Go Libraries Documentation

- SQLite driver: https://gitlab.com/cznic/sqlite
- Cobra CLI: https://github.com/spf13/cobra
- GitHub CLI: https://github.com/cli/go-gh
- YAML: https://github.com/go-yaml/yaml

### GitHub API Documentation

- REST API: https://docs.github.com/en/rest
- Starred endpoints: https://docs.github.com/en/rest/activity/starring
- User lists: https://docs.github.com/en/rest/activity/starring#list-star-gazers

### GitHub CLI Extension Guidelines

- Extension docs: https://docs.github.com/en/github-cli/github-cli/creating-github-cli-extensions
- Best practices: Use `gh` for auth, follow CLI conventions

### SQLite Documentation

- FTS5: https://www.sqlite.org/fts5.html
- JSON functions: https://www.sqlite.org/json1.html
- WAL mode: https://www.sqlite.org/wal.html

## Troubleshooting Common Issues

### Database Locked Errors

- Ensure WAL mode is enabled
- Check for long-running transactions
- Verify proper connection closing

### GitHub API Rate Limits

- Check rate limit before operations
- Implement exponential backoff
- Use conditional requests (ETags)

### Cross-Platform Path Issues

- Always use `filepath.Join()`, never string concatenation
- Test on Windows (backslashes)
- Handle `~` expansion explicitly

### FTS5 Query Errors

- Escape special characters: `" ( ) - *`
- Validate queries before execution
- Provide clear error messages for invalid syntax
