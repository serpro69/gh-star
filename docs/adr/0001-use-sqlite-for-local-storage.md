# ADR 0001: Use SQLite for Local Storage

## Status

Accepted

## Context

gh-star needs a local storage solution to manage 3000+ starred repositories with associated metadata (tags, notes, custom fields). The storage solution must support:

- Fast queries with complex filters (language, tags, date ranges)
- Full-text search across repository metadata and notes
- Multi-tag filtering with AND/OR logic
- Efficient handling of 3000+ records
- Cross-platform compatibility (Linux, macOS, Windows)
- Zero external dependencies for users
- Easy backup and version control

Options considered:
1. **SQLite** (relational database)
2. **Embedded NoSQL** (BoltDB, BadgerDB, LevelDB)
3. **Document stores** (embedded MongoDB, UnQLite)
4. **File-based** (JSON/YAML files)

## Decision

We will use **SQLite with the pure-Go driver `modernc.org/sqlite`** as the primary storage backend.

### Rationale

**Query Patterns Are Fundamentally Relational:**
- Multi-tag filtering ("repos with tag A AND tag B") is a natural JOIN operation in SQL
- In NoSQL, this requires building custom indexes and performing set intersections in application code
- Complex filters (language + tags + date range) are straightforward WHERE clauses in SQL but require significant custom logic in NoSQL

**Full-Text Search:**
- SQLite FTS5 provides battle-tested, optimized full-text search out of the box
- Supports phrases, boolean operators, ranking, stemming
- NoSQL alternatives would require integrating a separate search library (like Bleve) or building custom search

**Pure-Go SQLite Driver Eliminates CGo Concerns:**
- `modernc.org/sqlite` is a transpilation of SQLite C code to pure Go
- No CGo dependency means simple cross-compilation
- Eliminates the main historical drawback of SQLite in Go projects
- Same performance and features as native SQLite

**Performance:**
- Decades of query optimization in SQLite's C codebase
- Proper indexing makes queries sub-second even with 3000+ repos
- Application-level query engines in NoSQL would not match this performance

**Robustness:**
- SQLite is one of the most tested pieces of software in existence
- ACID transactions prevent data corruption
- WAL mode provides better concurrency
- File corruption risk is negligible for single-user CLI application

## Consequences

### Positive

- Sub-second query times for complex filters on 3000+ repos
- Built-in FTS5 for powerful full-text search
- No external dependencies or database servers
- Simple backup (single file)
- ACID transactions ensure data integrity
- Proven, battle-tested technology
- Pure-Go driver simplifies cross-compilation

### Negative

- Binary format is opaque (not human-readable)
  - Mitigated by explicit export to YAML/JSON for version control
- Single database file (though this is also a benefit for simplicity)
- Learning curve for SQL (though minimal for basic CRUD)

### Mitigation Strategies

- **Binary format concern**: Implement explicit `export` command to YAML/JSON for version control and human readability
- **Backup**: Single file makes backup trivial (`cp ~/.config/gh-star/stars.db ~/backup/`)
- **Version control**: Export to git-friendly YAML format when users want to version their data

## Alternatives Considered

### BoltDB / BadgerDB (Pure-Go Key-Value Stores)

**Rejected because:**
- Would require building query engine logic in application code
- Multi-tag filtering requires custom index maintenance and set operations
- No built-in full-text search (would need Bleve or custom implementation)
- Significantly more complexity for marginal benefit (avoiding CGo)
- Pure-Go SQLite driver makes "Go purity" argument moot

### File-Based Storage (JSON/YAML)

**Rejected because:**
- Reading/parsing 3000+ files for each query is prohibitively slow
- No indexing capabilities
- Full-text search would require custom implementation
- Inefficient for frequent updates

### Single JSON File

**Rejected because:**
- Entire file must be read into memory for every query
- No indexing or query optimization
- Slow for 3000+ entries
- Race conditions on concurrent access

## References

- [SQLite FTS5 Documentation](https://www.sqlite.org/fts5.html)
- [modernc.org/sqlite - Pure Go SQLite](https://gitlab.com/cznic/sqlite)
- Zen chat analysis: See design discussion in `.taskmaster/docs/gh-star-core-prd.txt`
- Design documentation: `docs/wip/gh-star-core/design.md`
