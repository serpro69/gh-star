# ADR 0005: Defer Analytics and Insights to Future Enhancement

## Status

Accepted

## Context

During the requirements gathering phase, analytics and insights were identified as a potential feature area:

- Trending patterns (what languages/topics starred more recently)
- Activity tracking (repos with recent commits, new releases)
- Collection statistics (breakdown by language, topic, star count)
- Discovery/recommendations (similar repos, suggestions)

However, building these features adds significant complexity to the initial implementation.

## Decision

We will **defer analytics and insights to a future enhancement** and focus v1.0 on core functionality:

- Search and discovery
- Organization (tags, notes)
- Batch operations
- Import/export
- GitHub sync

Analytics will be revisited in v2.0 or later based on user demand.

### Scope for v1.0 (In Scope)

✅ Core data management and organization
✅ Fast search and filtering
✅ Tagging and notes
✅ Batch operations
✅ Export/import for backup
✅ GitHub Lists sync

### Deferred to Future (Out of Scope)

❌ Trending patterns and activity tracking
❌ Collection statistics and breakdowns
❌ Discovery and recommendations
❌ Visualization and charts
❌ Automated insights

## Consequences

### Positive

- **Focused v1.0**: Ship core features faster without scope creep
- **Simpler implementation**: Less code to write, test, and maintain
- **User feedback driven**: Build analytics based on actual user needs
- **Easier to add later**: Database schema supports future analytics queries
- **Lower initial complexity**: Fewer dependencies and features to learn

### Negative

- **Missing insights**: Users can't see trending patterns or stats in v1.0
- **Manual analysis**: Users must export to analyze with other tools (jq, etc.)
- **Delayed differentiation**: Analytics could be a unique selling point

### Mitigation Strategies

**Users can still get insights via export + external tools:**

```bash
# Export to JSON and analyze with jq
gh star export ~/my-stars --format json

# Language breakdown
jq '[.stars[].language] | group_by(.) | map({language: .[0], count: length})' \
   ~/my-stars/metadata.json

# Recently starred
gh star list --since 2024-01-01 --format json | jq length

# Most starred repos in collection
gh star list --format json | jq 'sort_by(.stars_count) | reverse | .[0:10]'
```

**Database schema supports future analytics:**
- All timestamp data is preserved
- Repository metadata is comprehensive
- Can add analytics queries without schema changes

## Future Analytics Features (v2.0+)

When we implement analytics, we could add:

### Commands

```bash
gh star stats
# Language breakdown, topic trends, collection growth

gh star trending
# Repos you starred that are trending (recent stars increase)

gh star activity
# Repos with recent releases or high commit activity

gh star insights
# Personalized insights about your collection
```

### Implementation Approach

- Query local SQLite for statistics (fast, offline)
- Optional GitHub API calls for trending data (with caching)
- Export to various formats (JSON, charts, tables)
- Integration with external visualization tools

## User Feedback Trigger

We will reconsider analytics when:
- 10+ users request analytics features
- Common use cases emerge from issue discussions
- External tools (jq queries) users create show patterns
- Clear value proposition emerges

## Alternatives Considered

### Include Basic Statistics in v1.0

**Rejected because:**
- Even "basic" stats require significant implementation
- Would delay v1.0 release
- Not part of core value proposition (search, organize, backup)
- Can be approximated with existing commands

### Build Analytics-First

**Rejected because:**
- Core organization features are more valuable
- Analytics without good search/tagging is less useful
- Wrong priority for v1.0

## References

- User requirements: "Let's skip analytics for now"
- Design documentation: `docs/wip/gh-star-core/design.md` (Future Enhancements section)
- PRD: `.taskmaster/docs/gh-star-core-prd.txt` (Out of Scope section)
