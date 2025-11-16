# Contributing to gh-star

Thank you for your interest in contributing to gh-star! This document provides guidelines and information for contributors.

## Development Setup

### Prerequisites

- Go 1.21 or later
- GitHub CLI (`gh`) installed and authenticated
- (Optional) `fzf` for testing interactive mode

### Getting Started

```bash
# Clone the repository
git clone https://github.com/serpro69/gh-star.git
cd gh-star

# Build the project
go build .

# Run tests
go test ./...

# Install as GitHub CLI extension for testing
gh extension install .
```

## Project Structure

```
gh-star/
├── main.go                      # CLI entry point
├── cmd/                         # Command implementations
│   ├── root.go                  # Root command setup
│   ├── init.go, sync.go        # Setup and sync
│   ├── list.go, search.go      # Query commands
│   ├── tag.go, note.go         # Organization
│   ├── bulk.go                  # Batch operations
│   └── export.go, import.go    # Backup/restore
├── internal/                    # Internal packages
│   ├── config/                  # Configuration management
│   ├── db/                      # Database layer (SQLite)
│   ├── github/                  # GitHub API integration
│   ├── export/                  # Export/import functionality
│   ├── interactive/             # fzf integration
│   └── output/                  # Output formatters
├── testdata/                    # Test fixtures and mocks
└── docs/                        # Design documentation
    └── wip/gh-star-core/       # Current design specs
```

## Design Documentation

Before contributing, please review the design documentation:

- **Design Specification**: `docs/wip/gh-star-core/design.md`
  - Complete technical architecture
  - Database schema and data model
  - Command specifications
  - GitHub integration strategy

- **Implementation Plan**: `docs/wip/gh-star-core/implementation.md`
  - Detailed phase-by-phase task breakdown
  - Testing strategies
  - Technology recommendations
  - Task dependencies and parallelization opportunities

- **PRD**: `.taskmaster/docs/gh-star-core-prd.txt`
  - Product requirements and success criteria

## Technology Stack

- **Language**: Go 1.21+
- **Database**: SQLite with pure-Go driver (`modernc.org/sqlite`)
- **CLI Framework**: `github.com/spf13/cobra`
- **GitHub Integration**: `github.com/cli/go-gh`
- **YAML Support**: `gopkg.in/yaml.v3`
- **No CGo** - Pure Go for easy cross-platform compilation

## Development Workflow

### Finding Tasks

This project uses Task Master for task management:

```bash
# View all tasks
task-master list

# Get next available task
task-master next

# View specific task details
task-master show <task-id>
```

Tasks are defined in `.taskmaster/tasks/tasks.json` and follow the implementation plan phases.

### Making Changes

1. **Choose a task** from Task Master or the implementation plan
2. **Read the design docs** for the relevant phase/feature
3. **Write tests first** (TDD approach where practical)
4. **Implement the feature** following the design specifications
5. **Run tests** and ensure they pass
6. **Update documentation** if needed

### Code Style

- Follow standard Go conventions and idioms
- Use `gofmt` for formatting (enforced by CI)
- Run `go vet` before committing
- Use `golangci-lint` for additional checks
- Write clear, descriptive commit messages

### Testing

```bash
# Run all tests
go test ./...

# Run tests with coverage
go test -cover ./...

# Run specific package tests
go test ./internal/db/...

# Benchmark tests
go test -bench=. ./...
```

#### Testing Guidelines

- **Unit tests** for all packages in `internal/`
- **Integration tests** for commands using test databases
- **Mock GitHub API** responses in tests (don't hit real API)
- **Test fixtures** in `testdata/` for reproducible tests
- **Golden files** for output validation
- **Benchmark tests** for operations on large datasets (3000+ repos)

### Commit Messages

Follow conventional commits format:

```
<type>: <short summary>

<optional detailed description>

<optional references>
```

**Types**: `feat`, `fix`, `refactor`, `test`, `docs`, `chore`

**Examples**:
- `feat: add full-text search using FTS5`
- `fix: handle rate limiting in GitHub API client`
- `refactor: simplify tag normalization logic`
- `test: add integration tests for export command`
- `docs: update README with usage examples`

### Pull Requests

1. **Create a branch** for your changes
2. **Follow the implementation plan** for the feature
3. **Write/update tests** for your changes
4. **Update documentation** if adding new features
5. **Ensure CI passes** (tests, linting)
6. **Reference related tasks** in the PR description

## Architecture Guidelines

### Database Layer

- Use **prepared statements** for all queries
- Wrap batch operations in **transactions**
- Implement proper **error handling** (return errors, don't panic)
- Follow the **schema defined** in design.md exactly
- Keep FTS5 index **synchronized** with repository data

### GitHub API Integration

- Use `gh` CLI for **authentication** (no separate tokens)
- Handle **rate limiting** with exponential backoff
- Implement **pagination** for all list operations
- Add **progress indicators** for long-running operations
- **Mock API responses** in tests

### CLI Commands

- **Pipeline-friendly** output by default (simple text)
- Support `--json` flag for **structured output**
- Use `--interactive` flag for **optional fzf** mode
- Provide **clear error messages** with actionable suggestions
- Follow **gh CLI conventions** for consistency

### Tag Namespacing

- `gh:*` prefix is **reserved** for GitHub Lists
- User tags must **not start** with `gh:`
- Sync operations only modify `gh:*` tags
- **Never touch** user tags during sync

## Testing Best Practices

### Unit Tests

- Test all **public functions** in internal packages
- Use **table-driven tests** for multiple scenarios
- Mock **external dependencies** (GitHub API, database)
- Test **error cases** thoroughly

### Integration Tests

- Use **temporary databases** for each test
- Create **reusable fixtures** in testdata/
- Test **complete command workflows**
- Validate **output formats** with golden files

### Performance Tests

- Benchmark with **3000+ repositories**
- Verify queries execute in **< 1 second**
- Test **memory usage** for large datasets
- Profile with `go test -cpuprofile` and `go test -memprofile`

## Common Development Tasks

### Adding a New Command

1. Review design.md for command specification
2. Create `cmd/<command>.go` with Cobra setup
3. Implement command logic using internal packages
4. Add unit tests in `cmd/<command>_test.go`
5. Update help text and examples
6. Add integration tests

### Modifying Database Schema

1. Create new migration file: `migrations/XXX_description.sql`
2. Update schema.sql with the changes
3. Increment schema version
4. Test migration on existing databases
5. Add rollback migration if possible

### Adding a Filter/Flag

1. Add flag definition in command file
2. Update `ListOptions` or relevant struct
3. Modify query builder to handle new filter
4. Add tests for new filter
5. Update help text and README examples

## Getting Help

- **Design Questions**: Refer to `docs/wip/gh-star-core/design.md`
- **Implementation Details**: See `docs/wip/gh-star-core/implementation.md`
- **Task Breakdown**: Use Task Master or check `.taskmaster/tasks/tasks.json`
- **Issues**: Open an issue on GitHub for bugs or feature requests

## Code Review Checklist

Before submitting a PR, ensure:

- [ ] Tests pass: `go test ./...`
- [ ] Linting passes: `golangci-lint run`
- [ ] Code is formatted: `gofmt -w .`
- [ ] Design specifications followed
- [ ] Error handling is appropriate
- [ ] Documentation updated (if needed)
- [ ] Commit messages follow convention
- [ ] No debug code or commented-out sections
- [ ] Performance tested with large datasets (if relevant)

## Resources

### Documentation

- [GitHub CLI Extensions Guide](https://docs.github.com/en/github-cli/github-cli/creating-github-cli-extensions)
- [Cobra CLI Documentation](https://github.com/spf13/cobra)
- [SQLite FTS5 Documentation](https://www.sqlite.org/fts5.html)
- [go-gh Library](https://github.com/cli/go-gh)

### Tools

- [golangci-lint](https://golangci-lint.run/) - Linting
- [goreleaser](https://goreleaser.com/) - Release automation
- [Task Master](https://github.com/kris-liu-0/task-master-ai) - Task management

Thank you for contributing to gh-star!
