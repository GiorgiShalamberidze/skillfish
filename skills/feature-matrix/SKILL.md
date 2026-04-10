---
name: feature-matrix
description: Generate a comprehensive feature matrix document from git history, GitHub issues, and codebase exploration. Groups deliverables by RFC codes.
---

# Feature Matrix

> *"Every project accumulates features over time, but nobody maintains the map. The README lists what the project does today, git log shows what changed, and GitHub issues show what was planned — but nothing shows the full picture: what was built, when, why, and which issues drove it.*
>
> *I needed a skill that could walk into any repo, reconstruct the feature timeline from commits and issues, and produce a single document that maps every deliverable to its RFC, issue, and status — without me narrating the entire history from memory."*
>
> — **John Efemer**, creator of AI Coach

## When to Use

- Onboarding onto an existing project and need to understand what has been built
- Creating a feature inventory for stakeholder reporting or roadmap planning
- Auditing what was shipped vs. what was planned in GitHub issues
- Generating a living document that maps features to issues and deliverables
- Preparing for a product review, retrospective, or architecture decision record
- Migrating or forking a project and need a complete capability inventory

## Prerequisites

- `gh` CLI installed and authenticated (`gh auth status`)
- Working directory is a git repository with a GitHub remote
- Read access to the repository and its issues

## Workflow

### Phase 1: Data Collection (Parallel)

Run these four data streams simultaneously to maximize speed.

#### Stream 1: GitHub Issues

```bash
gh issue list --state all --limit 500 --json number,title,state,createdAt,closedAt,labels,body
```

Extract: issue number, title, state, creation date, closing date, labels, body (first 300 chars). Sort oldest-first. Group loosely by feature area based on title/label patterns.

#### Stream 2: Git Timeline

```bash
git log --format="%h %ad %s" --date=short --reverse
```

Identify major milestones: initial commit, large feature additions, migration files, new page routes, API endpoints. Build a chronological phase list with date ranges and commit references.

#### Stream 3: Codebase Exploration

Scan the project structure to catalog implemented features:

```bash
# Page routes (what users can access)
find src/pages -name "*.astro" -o -name "*.tsx" -o -name "*.ts" | head -100

# API endpoints (what the system exposes)
find src/pages/api -name "*.ts" | head -100

# Database schema (what data exists)
find . -name "schema.*" -o -name "*.sql" -path "*/migrations/*" | head -50

# Components (what UI exists)
find src/components -name "*.astro" -o -name "*.tsx" | head -100
```

Map each discovery to a feature area. Note migration file dates for feature timeline ordering.

#### Stream 4: Existing Documentation

Read any existing feature docs, changelogs, or roadmap files:

```bash
find . -maxdepth 3 -name "CHANGELOG*" -o -name "ROADMAP*" -o -name "features*" -o -name "*.md" -path "*/docs/*" | head -20
```

### Phase 2: Feature Clustering

Group collected data into RFC-worthy feature areas using these heuristics:

1. **Temporal clustering** — commits that land within the same 1-3 day window on related files belong together
2. **Issue clustering** — issues that reference the same feature area or share labels belong together
3. **Directory clustering** — files under the same directory tree (e.g. `src/pages/mcp/`) belong together
4. **Migration ordering** — database migrations establish the chronological sequence of feature areas

Assign RFC codes in chronological order starting from RFC-101 based on when the first commit for that feature area appeared.

### Phase 3: Document Generation

Produce a single markdown file following the output format below.

For each RFC:
1. Assign a descriptive title and one-liner explanation
2. Group features under the RFC with F-{RFC}.{seq} codes
3. For each feature, create a deliverable table with:
   - Numbered rows (referenceable)
   - Descriptive deliverable text (merge similar items into one line)
   - GitHub issue references (multiple per row if applicable, dash for untracked work)
   - Status (Done / Planned / In Progress)

### Phase 4: Validation

Verify completeness:
- [ ] Every GitHub issue appears in at least one deliverable row
- [ ] Every major directory/page route maps to a feature
- [ ] Every database migration maps to an RFC
- [ ] Chronological ordering matches git history
- [ ] No orphan issues (issues not mapped to any RFC)
- [ ] Summary table at the bottom tallies all RFCs with dates and status

## Output Format

```markdown
# Project Features & Roadmap

> {One-liner describing this document's purpose}

---

## RFC-101: {Descriptive Title} — {YYYY-MM-DD} — {Done|Planned|In Progress}
> {One-liner explaining what this feature area does and why it exists}

### F-101.1: {Feature Title}
{One-liner explaining the feature scope.}

| # | Deliverable | Issues | Status |
|---|------------|--------|--------|
| 1 | {Descriptive deliverable merging related work into one line} | #3, #5 | Done |
| 2 | {Another deliverable} | — | Done |
| 3 | {Planned deliverable} | #12 | Planned |

### F-101.2: {Feature Title}
{One-liner explaining the feature scope.}

| # | Deliverable | Issues | Status |
|---|------------|--------|--------|
| 1 | {Deliverable} | #7 | Done |

---

## Summary

| RFC | Feature Area | Date | Status | Issues |
|-----|-------------|------|--------|--------|
| RFC-101 | {Title} | {Date} | Done | #3, #5 |
| RFC-102 | {Title} | {Date} | Planned | #12 |

**Totals**: {N} RFCs, {N} features, {N}+ deliverables.
```

## Structural Rules

### RFC Codes

- Start at RFC-101 (not RFC-001 — leaves room for pre-history)
- Assign chronologically based on first commit in that feature area
- One RFC per major feature area (auth, registry, marketplace — not per-page)
- A single RFC can contain multiple features (F-101.1, F-101.2, etc.)

### Deliverable Rows

- Merge similar work into one row — "3-col card layout with server name, publisher, and transport badge" not three separate rows for layout, name, and badge
- Multiple issues per row when they contributed to the same deliverable
- Use dash (—) for untracked work that has no GitHub issue
- Status is per-row, not per-feature — a feature can have Done and Planned rows

### Title Convention

- RFC title: broad feature area name (not a task description)
- Feature title: specific capability within the RFC
- Deliverable: concrete thing that was shipped or will ship

Bad: `RFC-107: Fix bugs and add docs` (too vague)
Good: `RFC-107: Docs Hub, API Versioning & OAuth` (descriptive, scannable)

Bad deliverable: `Updated the search` (activity)
Good deliverable: `Full-text search API across name, description, and slug with transport/pricing filters` (capability)

### Date Assignment

- Use the date of the earliest commit in that RFC's feature area
- For planned RFCs with no commits, use the date of the earliest related issue
- Format: YYYY-MM-DD for RFC header, consistent throughout

## Examples

### Example 1: Mapping a Fresh Codebase

**Trigger:** User asks to create a feature matrix for a project with no existing documentation.

**Actions:**
1. Run all four data streams in parallel (issues, git log, codebase scan, existing docs)
2. Identify 3 database migrations dated March 8, March 13, March 31
3. Cluster commits around those dates into 3 RFC groups
4. Map 8 GitHub issues across the RFCs
5. Scan pages/api/ to find 15 API endpoints, assign to features
6. Generate `docs/features.md`

### Example 2: Updating an Existing Feature Matrix

**Trigger:** New features have been shipped since the last matrix update.

**Actions:**
1. Read existing `docs/features.md` to understand current RFC numbering
2. Run `git log --since="YYYY-MM-DD"` from the last known RFC date
3. Check for new GitHub issues since last update
4. Append new RFCs or add deliverable rows to existing features
5. Update the Summary table totals

### Example 3: Issue-Heavy Project with Sparse Commits

**Trigger:** Project has 50 issues but only 20 commits.

**Actions:**
1. Use issue creation dates as the primary timeline (not commits)
2. Cluster issues by labels and title patterns
3. Mark deliverables as Planned where issues exist but no code was committed
4. Cross-reference closed issues with commits to mark Done deliverables

## Good to Know

### Why RFC Codes Instead of Sequential Numbers

RFC codes (RFC-101, RFC-102) are stable identifiers that survive reordering. If you discover an earlier feature area that should come first chronologically, you can insert RFC-100 without renumbering everything. Starting at 101 leaves room for pre-project foundation work.

The F-{RFC}.{seq} feature codes (F-101.1, F-101.2) create a two-level hierarchy that's both human-scannable and machine-referenceable. "See F-104.3 row 2" is unambiguous in a 30-feature document.

### The Deliverable Table is Not a Task List

A common mistake is turning deliverable rows into granular tasks ("Add border radius to card", "Change font size on title"). Deliverables describe **shipped capabilities**, not implementation steps. Each row should make sense to someone who has never read the code — it answers "what can the system do now?" not "what did the developer type?"

Merge related implementation work into capability descriptions:
- Bad: 3 rows for "Add search input", "Add debounce logic", "Add loading spinner"
- Good: 1 row for "Live search with debounced queries, loading state, and result count"

### Parallel Data Collection is Essential

The four data streams (issues, git, codebase, docs) are independent and should run simultaneously. On a project with 100+ commits and 50+ issues, sequential collection wastes 2-3 minutes. Parallel collection completes in the time of the slowest stream.

Use background agents or parallel tool calls — never serialize what can be parallelized.

### Handling Projects Without GitHub Issues

Not every project tracks work in GitHub Issues. When issues are sparse or absent:
- Use git commit messages as the primary data source
- Use migration files and new page routes as milestone markers
- Mark all deliverables with — in the Issues column
- Note in the document header that the matrix was reconstructed from git history

The document is still valuable — it's the only place where the full feature inventory exists in one view.

## Integration with Other Skills

| Skill | Relationship |
|-------|-------------|
| **github-project-manager** | Upstream — issues created by this skill become input data for the feature matrix |
| **project-history** | Complementary — changelog/worklog provides granular session data; feature matrix is the high-level inventory |
| **solution-architect** | Downstream — feature matrix informs architecture reviews and technical debt assessments |
| **agile-product-owner** | Downstream — feature matrix feeds backlog grooming and sprint planning |
| **ac-complexity-assessor** | Complementary — complexity scores can annotate RFC-level effort estimates |

## Common Pitfalls

| Pitfall | Fix |
|---------|-----|
| Making deliverables too granular (task-level) | Merge into capability descriptions — "what can the system do?" |
| Ordering RFCs by importance instead of chronology | Always use first-commit date as the ordering key |
| Missing untracked work (no issue) | Codebase scan catches features that were built without issues |
| Putting planned work in Done RFCs | Planned deliverables can live inside Done RFCs — status is per-row |
| Stale matrix after new features ship | Re-run Streams 1-3 scoped to the date range since last update |
| Confusing RFC with Epic | An RFC is a feature area (can span months); an Epic is a single large deliverable |

## Related Skills

| Skill | Relationship |
|-------|-------------|
| **project-history** | Provides daily changelogs and feature-logs that feed into matrix updates |
| **github-project-manager** | Creates the issues that appear in the Issues column of deliverable tables |
| **solution-architect** | Consumes the feature matrix for architecture decision records |
| **project-planning** | Uses the matrix as input for WBS, Gantt charts, and resource allocation |
