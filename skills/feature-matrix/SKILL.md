---
name: feature-matrix
description: Build and incrementally maintain a feature matrix document that maps RFC feature areas, stories, and deliverables from git history and codebase exploration.
---

# Feature Matrix

> *"Every project accumulates features over time, but nobody maintains the map. The README lists what the project does today, git log shows what changed, and GitHub issues show what was planned — but nothing shows the full picture: what was built, when, why, and which issues drove it.*
>
> *I needed a skill that could walk into any repo, reconstruct the feature timeline from commits and issues, and produce a single living document that maps every deliverable to its RFC and status — then keep it current as new work ships."*
>
> — **John Efemer**, creator of AI Coach

## When to Use

- Onboarding onto an existing project and need to understand what has been built
- Creating a feature inventory for stakeholder reporting or roadmap planning
- Auditing what was shipped vs. what was planned
- Generating a living document that maps features to deliverables and stories
- Preparing for a product review, retrospective, or architecture decision record
- After shipping new features — incrementally updating the matrix
- Running `/feature-matrix` or `/update-features` slash commands

## Slash Commands

| Command | What it does |
|---------|--------------|
| `/feature-matrix` | Generate or fully rebuild the feature matrix document |
| `/update-features` | Incremental update — scan for changes since last recorded RFC date |

## Concepts

### RFC (Request for Comments)

An RFC is a **feature area** — a major capability of the system that groups related work together. One RFC can span weeks or months and contain multiple features and stories.

- Assigned chronologically: RFC-101 is the earliest feature area, RFC-102 is the next
- Named descriptively: `RFC-104: MCP Registry Platform` not `RFC-104: Build stuff`
- Carries a date (earliest commit in that area) and a status (Done / In Progress / Planned)
- Starting at 101 leaves room to insert earlier discoveries without renumbering

### Feature

A feature is a **specific capability** within an RFC. It gets an F-code: `F-104.1`, `F-104.2`, etc.

- One feature = one coherent thing the system can do
- Described with a title and a one-liner explanation
- Contains a deliverable table with stories

### Story

A story is a **concrete shipped deliverable** (or planned work item) inside a feature. Stories are rows in the deliverable table.

- Describes a capability, not an implementation step
- Maps to GitHub issues when available (dash for untracked work)
- Has its own status: Done, In Progress, or Planned
- Multiple issues can map to one story when they contributed to the same outcome

**Hierarchy:** `RFC > Feature > Story`

```
RFC-104: MCP Registry Platform          ← feature area
  F-104.1: Server Discovery & Search    ← capability
    Story 1: Registry listing page...   ← deliverable (row in table)
    Story 2: 3-col card layout...       ← deliverable (row in table)
  F-104.2: Server Detail Pages          ← capability
    Story 1: Detail page with...        ← deliverable
```

## Prerequisites

- Working directory is a git repository
- **Optional:** `gh` CLI for GitHub issue mapping (skipped if unavailable)
- **Optional:** `.github/project-config.json` from `github-project-manager` skill for issue/label context

## Workflow

### Phase 1: Detect Context

#### Step 1: Check for Existing Feature Matrix

Look for an existing feature matrix file in this order:

```
docs/features.md
docs/FEATURES.md
docs/feature-matrix.md
FEATURES.md
```

If found, read it and enter **incremental mode** (Phase 3). If not found, proceed with **full generation** (Phase 2).

#### Step 2: Check for Project Config

```bash
cat .github/project-config.json 2>/dev/null
```

If `.github/project-config.json` exists (from `github-project-manager`):
- Read `owner`, `repositories`, `project.number`, and `labels`
- Use this to pull GitHub issues with full label and project board context
- Map issues to features using domain labels (e.g. `domain:auth`, `domain:mcp`)

If it does **not** exist:
- Skip GitHub issue mapping entirely
- Use git history and codebase exploration as the sole data sources
- Mark all stories with `—` in the Issues column

#### Step 3: Pick Output File

If the target path already has a file with different content (e.g. `docs/features.md` is a manual document):
- Try `docs/feature-matrix.md` instead
- If that also exists, try `docs/project-features.md`
- Never overwrite an unrelated file

### Phase 2: Full Generation (First Run)

Run data collection in parallel, then cluster and generate.

#### Stream 1: Git Timeline

```bash
git log --format="%h %ad %s" --date=short --reverse
```

Identify major milestones: initial commit, large feature additions, migration files, new page routes, API endpoints. Build a chronological phase list with date ranges.

#### Stream 2: Codebase Exploration

Scan project structure to catalog what exists:

```bash
# Page routes / entry points
find src/pages -name "*.astro" -o -name "*.tsx" -o -name "*.ts" 2>/dev/null | head -100

# API endpoints
find src/pages/api -name "*.ts" 2>/dev/null | head -100

# Database schema and migrations
find . -name "schema.*" -o -name "*.sql" -path "*/migrations/*" 2>/dev/null | head -50

# Components
find src/components -name "*.astro" -o -name "*.tsx" 2>/dev/null | head -100
```

Map each discovery to a feature area. Note migration file dates for timeline ordering.

#### Stream 3: GitHub Issues (When Project Config Exists)

Only runs when `.github/project-config.json` is present.

```bash
gh issue list --state all --limit 500 --json number,title,state,createdAt,closedAt,labels,body
```

Sort oldest-first. Group by domain labels from project config. Map closed issues to Done stories, open issues to Planned stories.

#### Stream 4: Existing Documentation

```bash
find . -maxdepth 3 \( -name "CHANGELOG*" -o -name "ROADMAP*" -o -name "*.md" -path "*/docs/*" \) | head -20
```

#### Clustering

Group collected data into RFC feature areas:

1. **Temporal clustering** — commits in the same 1-3 day window on related files belong together
2. **Directory clustering** — files under the same tree (e.g. `src/pages/mcp/`) belong together
3. **Migration ordering** — database migrations mark the chronological boundary of feature areas
4. **Issue label clustering** — issues sharing domain labels belong to the same RFC (when available)

Assign RFC codes chronologically starting from RFC-101.

#### Generate Document

Write the feature matrix file using the output format below. Add a metadata footer for incremental tracking.

### Phase 3: Incremental Update

When an existing feature matrix is found, update it without regenerating from scratch.

#### Step 1: Read State Footer

The feature matrix ends with a metadata block:

```markdown
<!-- feature-matrix-state
last_rfc: RFC-112
last_date: 2026-04-10
total_rfcs: 13
total_features: 30
-->
```

Read `last_date` to scope the incremental scan.

#### Step 2: Scan for Changes

```bash
# New commits since last update
git log --format="%h %ad %s" --date=short --since="{last_date}"

# New/changed files
git diff --name-only HEAD~20

# New issues (if project config exists)
gh issue list --state all --json number,title,state,createdAt --jq "[.[] | select(.createdAt > \"{last_date}\")]"
```

#### Step 3: Classify Changes

For each new commit cluster:
- Does it belong to an existing RFC? → Add stories to existing features
- Does it represent a new feature area? → Create a new RFC with the next available code

#### Step 4: Update Document

- Append new RFCs at the end (before the Summary table)
- Add new story rows to existing feature tables
- Update the Summary table with new RFC entries and revised totals
- Update the metadata footer with new `last_rfc`, `last_date`, and totals

### Phase 4: Validation

After generation or update, verify:

- [ ] Every GitHub issue (if mapped) appears in at least one story row
- [ ] Every major directory/route maps to a feature
- [ ] Chronological ordering matches git history
- [ ] Summary table tallies match actual RFC count
- [ ] State footer reflects current totals and latest date
- [ ] No duplicate RFC codes

## Output Format

```markdown
# Project Features & Roadmap

> {One-liner describing this document's purpose and project.}

---

## RFC-101: {Descriptive Title} — {YYYY-MM-DD} — {Done|Planned|In Progress}
> {One-liner explaining what this feature area does and why it exists}

### F-101.1: {Feature Title}
{One-liner explaining the feature scope.}

| # | Deliverable | Issues | Status |
|---|------------|--------|--------|
| 1 | {Concrete capability merging related work into one line} | #3, #5 | Done |
| 2 | {Another capability} | — | Done |
| 3 | {Planned capability} | #12 | Planned |

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

<!-- feature-matrix-state
last_rfc: RFC-{NNN}
last_date: {YYYY-MM-DD}
total_rfcs: {N}
total_features: {N}
-->
```

## Structural Rules

### Story Rows Are Capabilities, Not Tasks

Deliverable rows describe **shipped capabilities**, not implementation steps. Each row should answer "what can the system do now?" — not "what did the developer type."

- Bad: 3 rows for "Add search input", "Add debounce logic", "Add loading spinner"
- Good: 1 row for "Live search with debounced queries, loading state, and result count"

Merge related implementation work. Multiple issues can map to one story when they drove the same outcome.

### Title Convention

- **RFC title**: Broad feature area name — `RFC-107: Docs Hub, API Versioning & OAuth`
- **Feature title**: Specific capability — `F-107.1: Professional Docs Hub`
- **Story**: Concrete deliverable — `Docs layout with sidebar navigation, breadcrumbs, and section cards`

Bad: `RFC-107: Fix bugs and add docs` (too vague, activity-focused)

### Date Assignment

- Use the date of the earliest commit in that RFC's feature area
- For planned RFCs with no commits, use the date of the earliest related issue
- Format: YYYY-MM-DD throughout

### Status Values

| Status | Meaning |
|--------|---------|
| Done | All stories in this RFC are shipped |
| In Progress | At least one story is being actively worked on |
| Planned | No stories have shipped yet; RFC exists from open issues or roadmap |

Status is tracked at **both** RFC level (header) and story level (table row). An RFC can be "Done" overall while containing a "Planned" story for future enhancement.

## Integration with github-project-manager

When `.github/project-config.json` exists, the feature matrix leverages it:

| project-config field | How feature-matrix uses it |
|---------------------|---------------------------|
| `owner` + `repositories` | Scopes `gh issue list` to correct repo |
| `labels.customDomains` | Maps domain labels to RFC groupings |
| `project.number` | Reads board status to determine story completion |
| `fields.Status.options` | Maps "Done" status to story status |

When the config does **not** exist:
- GitHub issue mapping is skipped entirely
- All Issues columns show `—`
- Git history and codebase structure are the sole data sources
- The document is still complete — it just lacks issue cross-references

## Examples

### Example 1: First Run on a Project Without GitHub Issues

**Trigger:** User runs `/feature-matrix` on a repo with no project-config.

**Actions:**
1. No `docs/features.md` found — enter full generation mode
2. No `.github/project-config.json` — skip issue mapping
3. Run git timeline + codebase scan in parallel
4. Find 3 migrations dated March 8, March 13, March 31 — cluster into 3 RFCs
5. Scan `src/pages/` to find 15 routes, `src/pages/api/` for 20 endpoints
6. Generate `docs/features.md` with all Issues columns as `—`
7. Write state footer: `last_rfc: RFC-103, last_date: 2026-03-31`

### Example 2: Incremental Update After New Features Ship

**Trigger:** User runs `/update-features` two weeks after initial generation.

**Actions:**
1. Read existing `docs/features.md` — find state footer with `last_date: 2026-03-31`
2. Run `git log --since="2026-03-31"` — find 15 new commits across 2 new feature areas
3. Check for new issues (project-config exists) — find 3 new closed issues
4. Create RFC-104 and RFC-105 for the new feature areas
5. Map the 3 issues to story rows in the new RFCs
6. Append both RFCs before the Summary table
7. Update Summary table and state footer: `last_rfc: RFC-105, last_date: 2026-04-10`

### Example 3: File Name Conflict

**Trigger:** `docs/features.md` already exists but contains unrelated content (e.g. a feature spec).

**Actions:**
1. Detect `docs/features.md` exists but doesn't contain `<!-- feature-matrix-state` footer
2. This is not a feature matrix file — do not overwrite
3. Try `docs/feature-matrix.md` — does not exist
4. Write feature matrix to `docs/feature-matrix.md`
5. Inform user: "Wrote to docs/feature-matrix.md (docs/features.md already had different content)"

### Example 4: Project With github-project-manager Active

**Trigger:** User runs `/feature-matrix` on a project that has `.github/project-config.json`.

**Actions:**
1. Read project-config: owner=`acme`, repo=`platform`, project #4
2. Pull all issues: 45 closed, 12 open — sorted by creation date
3. Group issues by `domain:*` labels into feature areas
4. Cross-reference with git timeline for date ordering
5. Map closed issues to Done stories, open issues to Planned stories
6. Generate with full issue references in every applicable row

## Good to Know

### The State Footer Enables Incremental Maintenance

The HTML comment at the bottom of the document (`<!-- feature-matrix-state ... -->`) is invisible when rendered but machine-readable by the skill. It stores the last RFC code, last date scanned, and running totals.

This means the skill never needs to re-analyze the entire git history on subsequent runs. It reads the footer, scopes the scan to new changes, and appends. On a project with 500 commits, the incremental update touches only the 10-20 commits since the last run.

If the footer is missing or corrupted, the skill falls back to full regeneration.

### Without Issues, the Matrix is Still Valuable

Many projects — especially solo or early-stage — don't track work in GitHub Issues. The feature matrix still works because its primary data source is the **git history and codebase structure**, not issues. Issues are a bonus layer that adds traceability when available.

A matrix with all `—` in the Issues column is still the only place where someone can see: "this project has 12 feature areas, 30 capabilities, shipped between March and April" — information that exists nowhere else in a single view.

### RFC Is Not an Epic

An RFC is a **feature area** that can span the life of the project. An Epic (in `github-project-manager` terms) is a **single large deliverable** with a defined scope and completion criteria.

One RFC can contain work from many Epics over time. RFC-104 (MCP Registry) might include Epic #3 (initial registry) shipped in March, Epic #15 (enrichment pipeline) shipped in April, and Epic #28 (search redesign) planned for June — all under the same RFC umbrella.

### Parallel Data Collection

The data streams (git, codebase, issues, docs) are independent and should run simultaneously. On a project with 100+ commits, sequential collection wastes 2-3 minutes. Use background agents or parallel tool calls.

## Common Pitfalls

| Pitfall | Fix |
|---------|-----|
| Making stories too granular (task-level) | Merge into capability descriptions — "what can the system do?" |
| Ordering RFCs by importance instead of chronology | Always use first-commit date as the ordering key |
| Missing untracked work (no issue exists) | Codebase scan catches features built without issues |
| Overwriting an existing non-matrix file | Always check for state footer before treating a file as the matrix |
| Forgetting to update state footer | Footer update is the last step of every generation/update cycle |
| Full regeneration when incremental would suffice | Read state footer first — only regenerate when footer is missing |
| Trying to map issues without project-config | Skip issues gracefully — don't fail or produce empty columns |

## Related Skills

| Skill | Relationship |
|-------|-------------|
| **github-project-manager** | Upstream — provides `.github/project-config.json` and creates the issues that appear in story rows |
| **project-history** | Complementary — changelog/worklog provides granular session data; feature matrix is the high-level inventory |
| **solution-architect** | Downstream — feature matrix informs architecture reviews and technical debt assessments |
| **project-planning** | Downstream — uses the matrix as input for WBS, Gantt charts, and resource allocation |
| **agile-product-owner** | Downstream — feature matrix feeds backlog grooming and sprint planning |
