# ADR 0002: One-Way Sync with Namespaced Tags for GitHub Lists

## Status

Accepted

## Context

Users have existing stars organized in GitHub Star Lists and want to:
1. Import lists as initial organization when first using gh-star
2. Continue managing stars via both gh-star CLI and GitHub web UI
3. Keep lists in sync without conflicts or data loss
4. Add local-only tags and metadata beyond GitHub's capabilities

The tool needs a clear strategy for reconciling GitHub Star Lists with local tags.

Options considered:
1. **Import once, then local-only** - Fetch lists during init, ignore thereafter
2. **Two-way sync** - Push local changes back to GitHub Lists
3. **Read-only sync** - Periodically fetch lists, merge with local data
4. **Separate concepts** - GitHub lists and local tags are completely independent
5. **One-way sync with namespaced tags** - Auto-manage `gh:*` tags from GitHub, keep user tags separate

## Decision

We will use **one-way sync with namespaced tags**:

- GitHub Star Lists are synced as special `gh:*` prefixed tags
- On sync: delete ALL `gh:*` tags, then re-apply from current GitHub Lists
- User tags (without `gh:` prefix) are NEVER touched by sync
- Clear mental model: "`gh:*` tags = from GitHub, everything else = mine"

### Implementation Details

**Initial Import:**
```bash
gh star init
# GitHub List "CLI Tools" → tag `gh:cli-tools`
# GitHub List "Go Libraries" → tag `gh:go-libraries`
```

**Ongoing Sync:**
```bash
gh star sync
# 1. Delete all tags where is_github_list = true
# 2. Re-fetch current GitHub Lists
# 3. Re-apply gh:* tags based on current memberships
# 4. User tags remain completely untouched
```

**Tag Naming Convention:**
- List name normalization: lowercase, spaces → hyphens
- "CLI Tools" → `gh:cli-tools`
- "Programming Languages" → `gh:programming-languages`

## Consequences

### Positive

- **Simple mental model**: Users immediately understand which tags come from GitHub
- **No conflict resolution needed**: Sync is a clean delete-and-reapply operation
- **Predictable behavior**: Sync outcome is always determinable
- **User data protected**: Local tags are never modified by sync
- **Solves both scenarios**:
  - Initial import: Get existing organization from GitHub
  - Ongoing sync: Stay updated with changes made on GitHub web UI
- **No API write-back complexity**: Read-only operations are simpler and safer
- **Flexibility**: Users can ignore `gh:*` tags if they prefer local-only workflow

### Negative

- **One-way only**: Local changes to `gh:*` tags are lost on next sync
  - Mitigated: Users use their own tags for local organization
- **Namespace reservation**: `gh:` prefix is reserved for GitHub Lists
  - Mitigated: Clear validation error if users try to create `gh:*` tags
- **List changes require GitHub**: Can't modify GitHub Lists from CLI
  - Accepted: GitHub's web UI is fine for occasional list management

### User Workflow

**Scenario 1: First-time user with existing lists**
```bash
gh star init
# Imports all stars and lists
# Lists become gh:cli-tools, gh:backend-tech, etc.

gh star list --tag gh:cli-tools
# See repos from "CLI Tools" list

gh star tag add some-repo my-tag
# Add personal organization
```

**Scenario 2: Starred something on GitHub web UI**
```bash
# Star new repo on github.com, add to "DevOps" list

gh star sync
# Fetches new repo
# Applies gh:devops tag automatically

gh star tag add new-repo production priority-1
# Add personal tags
```

**Scenario 3: List changes on GitHub**
```bash
# Remove repo from "CLI Tools" list on github.com

gh star sync
# gh:cli-tools tag is removed from that repo
# All user tags (production, priority-1) remain untouched
```

## Alternatives Considered

### Two-Way Sync

**Rejected because:**
- Requires write-back to GitHub API (more complex)
- Conflict resolution needed (what if changed in both places?)
- GitHub API rate limits make frequent sync impractical
- Adds significant complexity for marginal benefit
- Most users will manage via CLI once they adopt the tool

### Import Once, Then Local-Only

**Rejected because:**
- Organizational work done on GitHub web UI is lost
- Users who star repos on mobile/web don't get list associations
- Wastes existing GitHub List organization

### Separate Concepts (Lists and Tags Independent)

**Rejected because:**
- Cognitive overhead of two separate systems
- Query complexity (search across lists AND tags?)
- Duplicate organization (same repo in list and tag)
- Namespace approach is simpler

### Read-Only Sync with Merge

**Rejected because:**
- Merge logic is complex (union vs replace?)
- Unexpected behavior if users remove `gh:*` tags locally
- Not clear what "source of truth" is
- Namespaced approach provides clearer boundaries

## References

- Zen chat analysis: Storage and sync strategy discussion
- Design documentation: `docs/wip/gh-star-core/design.md` (GitHub Integration & Sync Strategy section)
- Implementation plan: `docs/wip/gh-star-core/implementation.md` (Phase 4: Init and Sync)
