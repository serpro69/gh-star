# gh-star

<div markdown="1" style="text-align: center; font-size: 1.2em;">

<b>🚧 Fork in progress, expect some dust 🚧</b>

[![github-tag](https://img.shields.io/github/v/tag/serpro69/gh-star?style=for-the-badge&logo=semver&logoColor=white)](https://github.com/serpro69/gh-star/tags)
[![github-license](https://img.shields.io/github/license/serpro69/gh-star?style=for-the-badge&logo=unlicense&logoColor=white)](https://opensource.org/license/mit)
[![github-stars](https://img.shields.io/github/stars/serpro69/gh-star?logo=github&logoColor=white&color=gold&style=for-the-badge)](https://github.com/serpro69/gh-star)

</div>

`gh-star` is a [GitHub CLI extension](https://docs.github.com/en/github-cli/github-cli/creating-github-cli-extensions#about-github-cli-extensions) for power users managing large collections of starred repositories. It provides robust search, organization, batch operations, and local control that GitHub's web interface doesn't offer.

## Why gh-star?

If you have hundreds or thousands of GitHub stars, you know the pain:
- **GitHub's search is limited** - Finding that repo you starred 6 months ago is frustrating
- **Organization is basic** - Star Lists help, but you need more: tags, notes, custom metadata
- **No batch operations** - Cleaning up archived repos or bulk tagging requires clicking through pages
- **No local control** - Can't backup, version control, or truly own your curated collection
- **CLI workflows are manual** - Hard to integrate with scripts, fzf, jq, and other terminal tools

**gh-star solves these problems** with a local-first approach: fast SQLite database for queries, flexible tagging beyond GitHub's lists, full-text search across all metadata, and version-controllable backups.

## Key Features

### Local-First Storage
- **SQLite database** with sub-second queries on 3000+ stars
- **Offline access** - search and browse without network
- **Full-text search** using FTS5 across names, descriptions, topics, and notes

### Powerful Organization
- **Flexible tagging** - Add as many tags as you need, organize your way
- **Notes and metadata** - Add context, priorities, or any custom information
- **GitHub Lists sync** - Imports your existing Star Lists as `gh:*` tags (one-way sync)

### Search and Discovery
- **Fast full-text search** - Find repos instantly across all metadata
- **Advanced filtering** - By language, date ranges, tags, archived status
- **Interactive mode** - Optional fzf integration for fuzzy finding

### Pipeline-Friendly CLI
- **Unix philosophy** - Works seamlessly with grep, jq, fzf, xargs
- **Multiple output formats** - Simple text, JSON, or TSV for different workflows
- **Scriptable** - Integrate into automation, dotfiles, or development workflows

### Backup and Portability
- **Export to YAML/JSON** - Version control your star collection
- **Git-friendly format** - One file per repo for clean diffs
- **Import/merge** - Restore backups or share collections

## How It Works

gh-star uses a hybrid architecture:

1. **Local SQLite database** (`~/.config/gh-star/stars.db`) stores all repository data and metadata
2. **Sync with GitHub** via `gh star sync` fetches new stars and updates metadata
3. **GitHub Lists** become `gh:*` tags (e.g., "CLI Tools" → `gh:cli-tools`) - synced automatically
4. **Your tags** stay local and are never touched by sync
5. **Export when you want** to YAML/JSON for version control or backup

## Installation

Install the latest release using the GitHub CLI:

```bash
gh extension install serpro69/gh-star
```

### Installation from Source

```bash
git clone https://github.com/serpro69/gh-star.git
cd gh-star && go build . && gh extension install .
```

> [!TIP]
> Clone the repo to a permanent location, gh cli will alias the binary when you run `gh extension install .` command.

## Quick Start

```bash
# First time: Initialize and import your stars
gh star init

# Search for repositories
gh star search kubernetes

# List repos with filters
gh star list --language Go --tag production

# Add your own tags
gh star tag add golang/go backend production priority-1

# Add notes to a repo
gh star note set kubernetes/kubernetes "Production k8s cluster dependency"

# Sync new stars from GitHub
gh star sync

# Export for backup
gh star export ~/my-stars-backup
cd ~/my-stars-backup && git init && git add . && git commit -m "Backup stars"
```

## Usage Examples

### Search and Filter

```bash
# Full-text search
gh star search "docker compose"

# Filter by language and tags
gh star list --language Go --tag cli

# Find recently starred repos
gh star list --since 2024-01-01 --sort starred

# Combine search with filters
gh star search kubernetes --language Go --tag production
```

### Organization

```bash
# Add multiple tags to a repo
gh star tag add myorg/myrepo production backend priority-1

# Remove tags
gh star tag remove myorg/myrepo priority-1

# List all your tags
gh star tags

# Add notes
gh star note set golang/go "Core language repo. Check for security updates."
```

### Interactive Mode

```bash
# Interactive search with fzf
gh star search cli --interactive

# Browse and select with preview
gh star list --tag production -i

# Pipe selected repos to other commands
gh star search docker -i | xargs -I {} gh star tag add {} container
```

### Batch Operations

```bash
# Tag all Go repos
gh star bulk-tag --language Go --tags backend

# Clean up archived repos
gh star bulk-unstar --archived

# Pipe to xargs for custom workflows
gh star list --tag legacy | xargs -I {} gh star tag remove {} production
```

### Integration with Other Tools

```bash
# Use with jq for complex queries
gh star list --json | jq '.[] | select(.stars_count > 1000) | .full_name'

# Pipe to grep
gh star list | grep kubernetes

# Open repos in browser
gh star search terraform -i | xargs -I {} gh browse {}
```

### Backup and Version Control

```bash
# Export your collection
gh star export ~/my-stars

# Version control it
cd ~/my-stars
git init
git add .
git commit -m "Initial star collection"

# Later: export updates
gh star export ~/my-stars
cd ~/my-stars && git add . && git commit -m "Update stars"

# Restore from backup
gh star import ~/my-stars
```

## Commands Reference

### Core Commands

- `gh star init` - Initialize gh-star and import your stars
- `gh star sync` - Sync new stars and updates from GitHub
- `gh star list` - List repositories with filters
- `gh star search <query>` - Full-text search across all metadata
- `gh star show <repo>` - Show detailed information about a repository

### Organization

- `gh star tag add <repo> <tags...>` - Add tags to a repository
- `gh star tag remove <repo> <tags...>` - Remove tags
- `gh star tags` - List all tags with counts
- `gh star note set <repo> <text>` - Add notes to a repository
- `gh star note show <repo>` - Display notes

### Batch Operations

- `gh star bulk-tag` - Add tags to multiple repos matching filters
- `gh star bulk-unstar` - Unstar repositories matching filters

### Import/Export

- `gh star export <path>` - Export to YAML/JSON for backup
- `gh star import <path>` - Import from backup

### Common Flags

- `--tag TAG` - Filter by tag (repeatable for AND logic)
- `--language LANG` - Filter by programming language
- `--since DATE` / `--until DATE` - Filter by date range
- `--format json|tsv` - Output format (default: text)
- `--interactive` / `-i` - Launch fzf for interactive selection
- `--sort starred|name|stars|updated` - Sort order

## Configuration

gh-star stores data in `~/.config/gh-star/`:
- `stars.db` - SQLite database with all your stars and metadata
- `config.yaml` - Configuration (future: custom settings)

### GitHub Lists Sync

When you run `gh star init` or `gh star sync`, GitHub Star Lists are imported as special tags:
- GitHub List **"CLI Tools"** → tag `gh:cli-tools`
- GitHub List **"Go Libraries"** → tag `gh:go-libraries`

These `gh:*` tags are automatically managed by sync. Your own tags (without prefix) are completely independent and never touched by sync.

## Contributing

Contributions are welcome! See [CONTRIBUTING.md](CONTRIBUTING.md) for development setup, project structure, and guidelines.

## License

This project is licensed under MIT License.

See [LICENSE.md](/LICENSE.md) file for details.
