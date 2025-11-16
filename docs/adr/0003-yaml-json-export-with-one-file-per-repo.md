# ADR 0003: YAML/JSON Export with One File Per Repository

## Status

Accepted

## Context

Users need to backup, version control, and share their star collections. The export format must be:

- **Human-readable** for browsing and manual editing
- **Version control friendly** with clean git diffs
- **Machine-readable** for programmatic access
- **Portable** for sharing collections or migrating between systems
- **Complete** - includes both GitHub data and user metadata

Options considered:
1. **Single JSON file** - All repos in one file
2. **Single YAML file** - All repos in one file
3. **Multiple files** - One file per repository
4. **SQLite dump** - Binary database backup
5. **CSV format** - Tabular export

## Decision

We will support **YAML and JSON formats with one file per repository**:

- **Directory structure**: `<path>/stars/owner/repo.yaml` (or `.json`)
- **User choice**: `--format yaml` (default) or `--format json`
- **Metadata file**: `metadata.yaml` with export info
- **Comprehensive**: Each file contains GitHub data + user metadata

### File Structure

```
~/my-stars/
├── metadata.yaml
└── stars/
    ├── golang/
    │   ├── go.yaml
    │   └── groupcache.yaml
    ├── kubernetes/
    │   └── kubernetes.yaml
    └── ...
```

### File Content Example

```yaml
# Exported by gh-star
github_id: 23096959
full_name: golang/go
description: The Go programming language
language: Go
topics: [compiler, go, programming-language]
starred_at: "2020-03-15T10:30:00Z"

# User metadata
tags:
  - gh:programming-languages  # From GitHub List
  - backend                    # User tag
  - production
notes: |
  Core Go repository. Check releases for security updates.
```

## Consequences

### Positive

- **Clean git diffs**: Changing tags on one repo only affects one file
- **Human browsable**: Can explore directory and read files with text editor
- **Selective operations**: Can edit, delete, or share individual repo files
- **Merge-friendly**: Git merge conflicts are isolated to specific repos
- **Format choice**: YAML for readability, JSON for `jq` processing
- **Directory organization**: Repos grouped by owner (optional: by language or other criteria)

### Negative

- **More files**: 3000 repos = 3000 files (could stress some file systems)
  - Mitigated: Modern file systems handle this fine; users can organize by subdirectories
- **Slower export**: Creating 3000 files takes longer than one file
  - Mitigated: Only export when user wants to backup/version; add progress bar
- **No atomic write**: Partial exports if process interrupted
  - Mitigated: Export to temp directory, move on success

### User Workflows

**Version Control:**
```bash
gh star export ~/my-stars
cd ~/my-stars
git add .
git diff  # Clean diff showing exactly which repos changed
git commit -m "Add new Docker tools"
```

**Manual Editing:**
```bash
# Edit a specific repo's metadata
vim ~/my-stars/stars/golang/go.yaml
# Change tags, add notes

gh star import ~/my-stars --merge
```

**Selective Sharing:**
```bash
# Share just Go repos
cp -r ~/my-stars/stars/golang/ /tmp/go-stars/
# Share individual repos
cp ~/my-stars/stars/kubernetes/kubernetes.yaml ~/share/
```

## Alternatives Considered

### Single JSON/YAML File

**Rejected because:**
- Entire file changes on any modification (noisy git diffs)
- Harder to browse (must parse entire file)
- Merge conflicts affect entire collection
- Single point of failure (corruption loses everything)

**Example of noisy diff:**
```diff
  {
    "repos": [
      { "name": "repo1", ... },
-     { "name": "repo2", "tags": ["old"] },
+     { "name": "repo2", "tags": ["old", "new"] },  # Only changed this
      { "name": "repo3", ... },
      ...
      # Git shows entire file as changed
    ]
  }
```

### SQLite Dump

**Rejected because:**
- Binary format (not human-readable)
- Not version control friendly (entire dump changes on any modification)
- No way to manually edit or selectively share
- Defeats purpose of export (might as well backup the .db file)

### CSV Format

**Rejected because:**
- Can't represent nested data (tags array, custom fields JSON)
- Not human-readable for complex data
- Poor git diff experience
- Lossy format (can't round-trip all data)

### Directory Structure Options

We considered organizing by:
- **Owner** (chosen): `stars/golang/go.yaml`
- **Language**: `stars/Go/golang-go.yaml`
- **Flat**: `stars/golang-go.yaml`

**Owner organization chosen because:**
- Matches GitHub's natural organization
- Allows easy selective operations by owner
- Prevents filename collisions (multiple repos named "utils")
- More intuitive for users

## Implementation Notes

### Export Process

1. Query all repositories from SQLite (with optional filters)
2. For each repo, serialize to YAML/JSON
3. Create directory structure: `mkdir -p stars/<owner>/`
4. Write file: `stars/<owner>/<repo>.yaml`
5. Create metadata file with export stats
6. Show progress bar for large exports

### Import Process

1. Read metadata file to validate export
2. Walk directory structure, collect all `.yaml`/`.json` files
3. Auto-detect format from extension
4. Parse each file, deserialize to repository struct
5. Insert/merge into SQLite (based on mode)
6. Show progress and summary

### Format Interchangeability

YAML and JSON are structurally equivalent - users can:
- Export as YAML, import as YAML
- Export as JSON, import as JSON
- Mix and match (though not recommended)

## References

- Zen chat analysis: Storage format discussion
- Design documentation: `docs/wip/gh-star-core/design.md` (Export/Import File Format section)
- Implementation plan: `docs/wip/gh-star-core/implementation.md` (Phase 8)
