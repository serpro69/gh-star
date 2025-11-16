# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

`gh-star` is a GitHub CLI extension for working with personal GitHub Stars and Star Lists from the terminal. The project is currently in early setup phase.

**Language**: Go
**Project Type**: GitHub CLI Extension
**Status**: Initial setup

## Development Commands

### Build and Installation

```bash
# Build from source
go build .

# Install as GitHub CLI extension
gh extension install .

# Install from remote
gh extension install serpro69/gh-star
```

**Important**: Clone the repo to a permanent location - gh cli will alias the binary when running `gh extension install .`

### Running the Extension

```bash
# After installation, use via gh CLI
gh star <command>
```

## Project Architecture

### Current State
The repository is currently being set up. The codebase structure will follow GitHub CLI extension conventions:
- Main entry point will be in root directory
- CLI command structure using a Go CLI framework (to be determined)
- Integration with GitHub API for Stars and Star Lists management

### Key Dependencies
- GitHub CLI (`gh`) - Required for extension installation and usage
- Go standard library
- GitHub API client (likely `go-github` or similar)

### Future Structure (Expected)
```
gh-star/
├── main.go                  # CLI entry point
├── cmd/                     # Command implementations
├── internal/                # Internal packages
│   ├── api/                 # GitHub API interactions
│   ├── config/              # Configuration management
│   ├── stars/               # Star management logic
│   └── lists/               # Star Lists management logic
├── go.mod                   # Go module definition
├── go.sum                   # Go module checksums
└── README.md                # Documentation
```

## Development Workflow

### Prerequisites
1. Go installed (version TBD based on go.mod when created)
2. GitHub CLI (`gh`) installed and authenticated

### Testing Changes
After making changes, rebuild and test:
```bash
go build .
# Test the extension
gh star <test-command>
```

### GitHub CLI Extension Guidelines
- Entry point must be executable
- Extension commands are invoked as `gh <extension-name> <args>`
- Follow GitHub CLI extension best practices for auth and API usage
- Use `gh` for GitHub authentication rather than implementing custom auth

## Code Style and Patterns

### Go Conventions
- Follow standard Go project layout
- Use `gofmt` for formatting
- Follow Go naming conventions (exported vs unexported identifiers)
- Prefer standard library where possible

### Error Handling
- Return errors rather than panicking
- Provide helpful error messages to users
- Handle GitHub API rate limits gracefully

### GitHub API Integration
- Respect rate limits
- Use authenticated requests (via gh CLI)
- Handle pagination for list operations
- Cache data when appropriate to minimize API calls

## Task Master AI Instructions

**IMPORTANT!!! Import Task Master's development workflow commands and guidelines, treat as if import is in the main CLAUDE.md file.**

@./.taskmaster/CLAUDE.md
