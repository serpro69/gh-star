# ADR 0004: Hybrid CLI Mode - Pipeline-Friendly with Interactive Option

## Status

Accepted

## Context

gh-star needs to support diverse user workflows:

1. **Unix pipeline integration** - Piping to grep, jq, xargs, fzf
2. **Interactive exploration** - Browsing and fuzzy finding through 3000+ repos
3. **Scripting and automation** - Integrating into scripts and workflows
4. **Quick lookups** - One-liner commands for fast searches

Users need both:
- **Machine-readable output** for scripting (JSON, TSV)
- **Interactive UIs** for exploration (fzf, potential TUI)

Options considered:
1. **Always interactive** (like `lazygit`, `k9s`)
2. **Always pipeline-friendly** (simple text output only)
3. **Separate commands** (`gh star list` vs `gh star browse`)
4. **Hybrid with flag** (default pipe-friendly, `--interactive` for TUI)
5. **Auto-detect** (interactive if TTY, pipe-friendly if piped)

## Decision

We will use a **hybrid approach with explicit flag**:

- **Default behavior**: Pipeline-friendly output (simple text, one repo per line)
- **Opt-in interactive**: `--interactive` / `-i` flag launches fzf
- **Format flags**: `--format json|tsv` for structured output
- **Clear and explicit**: User controls mode, no magic auto-detection

### Implementation

```bash
# Default: pipeline-friendly (scriptable)
gh star list
# Output:
# golang/go
# kubernetes/kubernetes
# ...

# JSON for jq
gh star list --json | jq '.[] | select(.language == "Go")'

# Interactive with fzf
gh star list --interactive
# Opens fzf with preview pane, multi-select, etc.

# Pipe interactive selection to other commands
gh star search docker -i | xargs -I {} gh star tag add {} container
```

## Consequences

### Positive

- **Unix philosophy**: Default output works seamlessly with pipes and standard tools
- **Explicit control**: Users choose mode based on their current task
- **Maximum flexibility**: Same command supports both workflows
- **No surprises**: No magic auto-detection that might behave unexpectedly
- **Composable**: Interactive mode outputs to stdout, can be piped further
- **Gradual learning**: New users can start with simple commands, discover interactive mode later

### Negative

- **Extra typing**: Users must type `-i` for interactive mode
  - Mitigated: Short flag `-i` is quick to type; users can create shell aliases
- **Two code paths**: Must maintain both output modes
  - Mitigated: Small complexity; interactive mode wraps normal query logic

### User Workflows

**Quick scripting:**
```bash
# No extra flags needed for piping
gh star list --language Go | wc -l
gh star search kubernetes | head -5
```

**Interactive exploration:**
```bash
# Explicit opt-in to interactive mode
gh star search cli -i
gh star list --tag production -i
```

**Mixed workflows:**
```bash
# Interactive selection, then pipe to batch command
gh star list -i | xargs -I {} gh star tag add {} reviewed

# Filter first, then interactive
gh star list --language Go --since 2024 -i
```

## Alternatives Considered

### Always Interactive (TUI)

**Rejected because:**
- Not composable with other tools
- Can't pipe output to grep, jq, xargs
- Poor for scripting and automation
- Requires user interaction for batch operations
- Many users prefer CLI over TUI

### Auto-Detection (Interactive if TTY)

**Rejected because:**
- Surprising behavior when output is redirected
- Scripts might break if run differently (TTY vs non-TTY)
- Harder to test (behavior changes based on environment)
- Users lose explicit control
- Common pattern in CLI tools, but explicit is better for power users

### Separate Commands

**Example**: `gh star list` (text) vs `gh star browse` (interactive)

**Rejected because:**
- More commands to learn and remember
- Duplicated flag definitions across commands
- Splitting what is conceptually the same operation
- Hybrid flag approach is simpler

### Always Pipeline-Friendly Only

**Rejected because:**
- Misses opportunity for better UX with fzf integration
- Interactive exploration is valuable for 3000+ repos
- Users explicitly requested interactive mode (option A from refinement)

## Interactive Mode Details

### fzf Integration

When `--interactive` flag is used:
1. Check if `fzf` is installed (`exec.LookPath`)
2. If not found: show warning with install instructions, fall back to simple list
3. If found: pipe results to fzf with:
   - Preview pane showing `gh star show {}`
   - Multi-select with TAB
   - Custom key bindings (CTRL-T for tag, CTRL-N for notes)
   - Selected items written to stdout

### Future: Rich TUI

Option to build full TUI (like `lazygit`) in the future:
- `--tui` flag for full-screen terminal UI
- Separate command: `gh star ui`
- But always keep pipeline-friendly default

## Implementation Notes

### Output Formats

**Simple text** (default):
```
golang/go
kubernetes/kubernetes
```

**JSON** (`--format json`):
```json
[
  {
    "full_name": "golang/go",
    "description": "The Go programming language",
    "language": "Go",
    ...
  }
]
```

**TSV** (`--format tsv`):
```
golang/go	The Go programming language	Go	115234
kubernetes/kubernetes	Production-Grade Container Orchestration	Go	98765
```

### fzf Configuration

Users can customize fzf behavior via environment variable:
```bash
export GH_STAR_FZF_OPTS="--height 80% --layout reverse"
```

## References

- User requirements: Interactive exploration (A) + quick lookups/piping (C)
- Design documentation: `docs/wip/gh-star-core/design.md` (Interactive Mode section)
- Implementation plan: `docs/wip/gh-star-core/implementation.md` (Phases 5 and 9)
