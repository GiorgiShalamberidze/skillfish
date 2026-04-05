---
name: ai-code-review-tutor
description: Teach AI to review like your team: custom review checklists, tone calibration, org-specific patterns, and AI-assisted code review workflows.
---

# AI Code Review Tutor

Teach AI coding agents to perform code reviews that match your team's standards, voice, and domain expertise. This skill covers building custom review checklists, calibrating review tone, crafting effective AI review prompts, encoding org-specific conventions, automating reviews in CI, and building feedback loops that make reviews better over time. Every pattern is designed for real-world teams shipping production code.

---

## Table of Contents

- [Review Philosophy](#review-philosophy)
- [Custom Checklists](#custom-checklists)
- [Tone Calibration](#tone-calibration)
- [AI Review Prompts](#ai-review-prompts)
- [Org-Specific Patterns](#org-specific-patterns)
- [Review Automation](#review-automation)
- [Learning from Reviews](#learning-from-reviews)
- [Review Metrics](#review-metrics)

---

## Review Philosophy

A great code review is not a gate. It is a conversation that improves the code, grows the author, and protects the system. Understanding the difference between meaningful feedback and noise is the foundation of every other section in this skill.

### What Makes a Great Code Review

Great reviews focus on three things: correctness, clarity, and maintainability. Everything else is secondary. A reviewer who catches a race condition is more valuable than one who flags ten style violations the linter should handle.

**The review hierarchy (highest to lowest priority):**

1. **Correctness** -- Does the code do what it claims? Are there bugs, edge cases, or data integrity issues?
2. **Security** -- Are there vulnerabilities, injection vectors, or permission gaps?
3. **Architecture** -- Does the change fit the system design? Will it cause scaling or maintenance problems?
4. **Clarity** -- Can someone new to the codebase understand this in six months?
5. **Performance** -- Are there obvious inefficiencies or resource leaks?
6. **Style** -- Naming, formatting, and conventions (automate this layer whenever possible)

**Example -- a review comment that matters:**

```
The `processOrder()` function mutates the input `order` object directly.
If this runs concurrently (which it will in the queue worker), two
threads can corrupt the same order. Consider deep-cloning the order
at the start of the function, or switching to an immutable pattern.
```

**Example -- a review comment that does not matter:**

```
Nit: rename `idx` to `index` for readability.
```

The first comment prevents a production bug. The second is a preference with no impact. Train your AI reviewer to tell the difference.

### Review vs Nitpick

Nitpicks are comments about style, naming preferences, or minor formatting that have no functional impact. They are not harmful on their own, but they become a problem when they dominate a review and bury the important feedback.

**Rule of thumb:** If the comment would not survive a "Would this matter if the codebase were on fire?" test, it is a nitpick. Label it as such or omit it.

**Classifying comments in your review config:**

```yaml
# .review/classification.yaml
severity_levels:
  blocker:
    description: "Must fix before merge. Bugs, security issues, data loss."
    label: "BLOCKER"
    merge_block: true

  concern:
    description: "Should fix. Architecture drift, unclear logic, missing tests."
    label: "CONCERN"
    merge_block: false

  suggestion:
    description: "Nice to have. Better naming, alternative approaches."
    label: "SUGGESTION"
    merge_block: false

  nitpick:
    description: "Style preference. No functional impact."
    label: "NIT"
    merge_block: false
    max_per_review: 2  # Cap nitpicks to avoid noise
```

### Constructive Feedback

Every review comment should answer three questions: what is the problem, why does it matter, and what should be done instead.

**Anti-pattern -- vague review:**

```
This doesn't look right.
```

**Better -- constructive review:**

```
CONCERN: The retry logic here retries on all exceptions, including
ValueError (invalid input). Retrying on bad input will never succeed
and burns through the retry budget. Consider retrying only on
ConnectionError and TimeoutError.
```

**Pattern for constructive comments:**

```
[SEVERITY]: [What is wrong]
[Why it matters -- the consequence]
[What to do instead -- specific suggestion or example]
```

### Review Goals

Define what your team wants from code review before you configure any tooling. Different teams have different goals.

| Goal | Review focus | AI behavior |
|------|-------------|-------------|
| Bug prevention | Correctness, edge cases, error handling | Deep logic analysis, trace data flows |
| Knowledge sharing | Clarity, documentation, naming | Explain patterns, link to docs |
| Standards enforcement | Style, architecture, conventions | Check against team rules, flag drift |
| Security hardening | Auth, input validation, secrets | Scan for OWASP top 10, check permissions |
| Onboarding acceleration | Context, why-not-just, alternatives | Explain existing patterns, suggest learning resources |

**Example -- defining review goals in a config:**

```yaml
# .review/goals.yaml
primary_goals:
  - bug_prevention
  - security_hardening

secondary_goals:
  - knowledge_sharing

out_of_scope:
  - style_enforcement  # Handled by prettier/eslint
  - formatting          # Handled by pre-commit hooks
```

---

## Custom Checklists

Generic review checklists miss the things that matter most to your team. Build checklists that encode your domain knowledge, past incidents, and architectural decisions.

### Building Team-Specific Review Checklists

Start from your incident history and past review comments. The best checklists come from real bugs your team has shipped.

**Step 1 -- Mine your incident history:**

```bash
# Extract common themes from post-mortems
# Look for patterns: "We should have caught this in review"
gh issue list --label "postmortem" --json title,body --limit 50 | \
  jq -r '.[].title'
```

**Step 2 -- Structure your checklist by category:**

```yaml
# .review/checklists/backend-api.yaml
name: Backend API Review Checklist
applies_to:
  paths:
    - "src/api/**"
    - "src/routes/**"
  file_types: [".ts", ".py"]

checks:
  error_handling:
    - "All API endpoints return consistent error response shapes"
    - "Database errors are caught and wrapped, never leaked to client"
    - "Validation errors return 400 with field-level messages"
    - "Authentication failures return 401, not 403"

  data_access:
    - "Queries use parameterized statements, never string concatenation"
    - "N+1 query patterns are avoided (check for loops with DB calls)"
    - "Database transactions wrap multi-step mutations"
    - "Pagination is enforced on list endpoints (no unbounded queries)"

  api_contract:
    - "New fields are additive, not replacing existing fields"
    - "Removed fields go through deprecation, not immediate deletion"
    - "Response types match the OpenAPI spec"
    - "Breaking changes increment the API version"
```

**Step 3 -- Create role-specific checklists:**

```yaml
# .review/checklists/frontend-react.yaml
name: Frontend React Review Checklist
applies_to:
  paths:
    - "src/components/**"
    - "src/pages/**"
  file_types: [".tsx", ".jsx"]

checks:
  rendering:
    - "Components do not call hooks conditionally"
    - "useEffect dependencies are complete (no missing deps)"
    - "Large lists use virtualization or pagination"
    - "Re-renders are bounded (no object literals in JSX props)"

  accessibility:
    - "Interactive elements have aria labels or visible text"
    - "Images have alt text"
    - "Focus management is handled for modals and drawers"
    - "Color is not the only means of conveying information"

  state_management:
    - "Server state uses react-query/SWR, not local useState"
    - "Form state uses a form library, not manual useState per field"
    - "Global state changes are minimal and justified"
```

### Security Checks

Security review items should never be optional. Encode them as blockers.

```yaml
# .review/checklists/security.yaml
name: Security Review Checklist
severity: blocker
applies_to:
  paths: ["**/*"]

checks:
  authentication:
    - "New endpoints require authentication unless explicitly public"
    - "JWT tokens are validated on every request, not just at login"
    - "Session tokens have expiration and rotation"

  authorization:
    - "Resource access checks use the current user's permissions"
    - "No direct object references without ownership validation"
    - "Admin-only routes have role checks, not just auth checks"

  input_validation:
    - "All user input is validated and sanitized before use"
    - "File uploads validate type, size, and content"
    - "Redirect URLs are validated against an allowlist"
    - "SQL queries use parameterized statements"

  secrets:
    - "No hardcoded API keys, passwords, or tokens"
    - "Environment variables are used for all secrets"
    - "Secrets are not logged or included in error messages"
    - ".env files are in .gitignore"

  data_exposure:
    - "API responses do not include internal IDs unnecessarily"
    - "Stack traces are not exposed in production error responses"
    - "Sensitive fields (password, SSN, tokens) are never in logs"
```

### Performance Checks

```yaml
# .review/checklists/performance.yaml
name: Performance Review Checklist
applies_to:
  paths: ["src/**"]

checks:
  database:
    - "New queries have appropriate indexes"
    - "No SELECT * in production queries"
    - "Batch operations use bulk insert/update, not loops"
    - "Connection pools are properly sized"

  network:
    - "API calls are batched where possible"
    - "Large payloads use pagination or streaming"
    - "Cache headers are set for static and semi-static responses"
    - "No synchronous external API calls in hot paths"

  memory:
    - "Large collections are streamed, not loaded into memory"
    - "Event listeners and subscriptions are cleaned up"
    - "Closures do not capture unnecessary references"
```

### Domain Rules

Every team has domain-specific rules that no generic tool knows about.

```yaml
# .review/checklists/domain-fintech.yaml
name: Fintech Domain Rules
applies_to:
  paths: ["src/payments/**", "src/ledger/**", "src/transactions/**"]

checks:
  money_handling:
    - "All monetary values use integer cents, never floating point"
    - "Currency is always stored alongside amount"
    - "Rounding is explicit and documented (ROUND_HALF_UP for display)"
    - "Cross-currency operations go through the exchange rate service"

  audit:
    - "All state changes to financial records create audit log entries"
    - "Audit entries include who, what, when, and previous value"
    - "Soft-delete is used for financial records, never hard delete"

  compliance:
    - "PII fields are encrypted at rest"
    - "Data retention policies are enforced on new tables"
    - "Export functionality respects GDPR data scope"
```

---

## Tone Calibration

How you say something matters as much as what you say. AI reviewers default to a robotic, clinical tone. Calibrate the tone to match your team culture.

### Mentoring vs Gatekeeping

Two ends of the review tone spectrum:

**Gatekeeping tone (default AI behavior -- avoid this):**

```
This function has a bug. The variable `count` is not initialized
before the loop. Fix this.
```

**Mentoring tone (what you want):**

```
Heads up -- `count` is used in the loop on line 42 but isn't
initialized before the loop starts. On the first iteration it'll be
`undefined`, which makes the sum calculation return NaN. Setting
`let count = 0` before the loop should fix it.

Similar pattern: see how we handled it in `src/utils/aggregate.ts:28`.
```

**Configure the tone in your review prompt:**

```yaml
# .review/tone.yaml
voice: mentoring

principles:
  - "Assume the author is competent and had reasons for their choices"
  - "Explain the why, not just the what"
  - "Link to examples in the codebase when suggesting alternatives"
  - "Use 'we' language -- the team owns the code, not the individual"
  - "Ask questions when intent is unclear, don't assume mistakes"

avoid:
  - "Imperative commands without explanation (Fix this, Change this)"
  - "Sarcasm or rhetorical questions"
  - "Excessive exclamation marks"
  - "Phrases like 'obviously', 'simply', 'just'"

examples:
  before: "This is wrong. Use a Map instead of an object."
  after: >
    This object is used for frequent lookups by key. A Map would be
    more appropriate here since it handles non-string keys and has
    better performance for frequent additions/deletions. We switched
    to Maps for similar cases in the cache module -- see
    src/cache/store.ts for the pattern.
```

### Suggestions vs Requirements

Not every comment demands action. Make the distinction explicit.

**Pattern -- tag your comments:**

```
MUST: This SQL query is vulnerable to injection. Use parameterized
queries.

SHOULD: This function is 80 lines. Consider extracting the
validation logic into a separate function for testability.

COULD: The variable name `d` could be more descriptive. Something
like `deliveryDate` would clarify intent.

PRAISE: Nice use of the builder pattern here. This is much cleaner
than the old switch-case approach.
```

**Configure tag behavior:**

```yaml
# .review/tags.yaml
comment_tags:
  MUST:
    blocks_merge: true
    description: "Required change. Will not approve without this."
    tone: "Direct but respectful. Explain the risk."

  SHOULD:
    blocks_merge: false
    description: "Strongly recommended. Will approve without, but prefers fix."
    tone: "Persuasive. Explain the benefit."

  COULD:
    blocks_merge: false
    description: "Optional improvement. Take it or leave it."
    tone: "Light. Frame as an idea, not a correction."

  PRAISE:
    blocks_merge: false
    description: "Positive reinforcement for good patterns."
    tone: "Genuine. Be specific about what is good and why."
```

### Praise Patterns

AI reviewers almost never give positive feedback. This is a mistake. Praise reinforces good patterns and makes the review feel balanced.

**Anti-pattern -- review with no praise:**

```
Line 12: Missing null check.
Line 34: This should use the repository pattern.
Line 56: Variable name is unclear.
Line 78: Missing test for error case.
```

**Better -- balanced review:**

```
Overall: Clean separation of concerns. The new service layer is well
structured.

PRAISE: The error handling in `createOrder()` is thorough -- covering
both validation and downstream service failures with appropriate
error types. This is the pattern we want everywhere.

SHOULD: Line 34 -- the direct database call here bypasses the
repository layer. Moving it to `OrderRepository.findByStatus()` would
keep the data access patterns consistent.

MUST: Line 56 -- the user ID from the request body is used without
verifying it matches the authenticated user. This allows users to
create orders on behalf of other users.
```

**Configure a praise ratio:**

```yaml
# .review/tone.yaml
praise:
  enabled: true
  min_ratio: 0.2  # At least 1 praise per 5 comments
  triggers:
    - "Good test coverage for edge cases"
    - "Clean separation of concerns"
    - "Consistent with existing patterns"
    - "Good error messages for debugging"
    - "Thoughtful API design"
```

---

## AI Review Prompts

The quality of an AI code review depends entirely on the prompt. Generic prompts produce generic reviews. Specific prompts produce reviews that sound like your team.

### Crafting Prompts for Claude/Copilot Reviews

**Anti-pattern -- generic prompt:**

```
Review this code and suggest improvements.
```

**Better -- structured prompt with team context:**

```markdown
You are a senior code reviewer for a fintech backend team. Our stack
is Python 3.12, FastAPI, SQLAlchemy 2.0, and PostgreSQL.

Review the following pull request diff. For each comment:
1. Classify as MUST / SHOULD / COULD / PRAISE
2. Reference the specific file and line
3. Explain why the change matters (not just what to change)
4. Link to our coding standards when applicable

Focus areas (in priority order):
- Correctness: race conditions, data integrity, error handling
- Security: SQL injection, auth bypasses, data leaks
- Architecture: does this fit our service-repository pattern?
- Performance: N+1 queries, missing indexes, unbounded loops

Do NOT flag:
- Import ordering (handled by isort)
- Formatting (handled by black)
- Type hints on tests (we skip those deliberately)
- Naming preferences unless genuinely unclear

Tone: mentoring, not gatekeeping. Assume the author is competent.
Use "we" language. Explain the "why" for every comment.
```

### Context Injection

Give the AI reviewer the context it needs to make good decisions.

**Architecture context:**

```markdown
## Architecture Context

Our application follows a layered architecture:
- Routes (FastAPI) -> Services -> Repositories -> Database
- Services contain business logic, never import from routes
- Repositories contain data access, never import from services
- Cross-service communication goes through events, not direct calls

When reviewing, flag any violations of these boundaries.
```

**Recent incident context:**

```markdown
## Recent Incidents (Review with Extra Care)

- 2025-12-01: Race condition in payment processing caused double
  charges. Root cause: no distributed lock on order state transitions.
  Flag any order/payment state changes without locking.

- 2025-11-15: Customer data leaked in error logs. Root cause:
  exception handler logged full request body including PII.
  Flag any logging of request bodies or user-facing error details.
```

**Dependency context:**

```markdown
## Dependency Rules

- Never add dependencies without a lockfile update
- No dependencies with fewer than 1000 GitHub stars (risk threshold)
- Security-critical deps (crypto, auth) must be from well-known orgs
- Check for known vulnerabilities before approving new deps
```

### File-Specific Rules

Different files deserve different review intensity.

```yaml
# .review/file-rules.yaml
rules:
  - pattern: "src/payments/**"
    review_level: critical
    extra_checks:
      - "Verify idempotency keys on all mutation endpoints"
      - "Check for proper decimal/cents handling"
      - "Ensure audit trail for state changes"

  - pattern: "src/api/routes/**"
    review_level: standard
    extra_checks:
      - "Verify auth middleware is applied"
      - "Check request/response schema validation"

  - pattern: "tests/**"
    review_level: light
    extra_checks:
      - "Verify test names describe the scenario, not the implementation"
      - "Check that assertions are specific, not just 'assert result'"

  - pattern: "*.config.*"
    review_level: critical
    extra_checks:
      - "Verify no secrets in config files"
      - "Check for environment-specific overrides"

  - pattern: "migrations/**"
    review_level: critical
    extra_checks:
      - "Verify migration is reversible"
      - "Check for data loss in column drops or type changes"
      - "Ensure indexes are created concurrently for large tables"
```

---

## Org-Specific Patterns

Every team has conventions that live in people's heads. Encode them so AI reviewers can enforce them consistently.

### Encoding Team Conventions

**Create a conventions document the AI reviewer can reference:**

```yaml
# .review/conventions.yaml
conventions:
  error_handling:
    description: "We use typed errors with error codes, not string messages"
    correct: |
      raise OrderNotFoundError(order_id=order_id)
    incorrect: |
      raise Exception("Order not found")
    reference: "src/errors/base.py"

  logging:
    description: "Structured logging with context, never print statements"
    correct: |
      logger.info("order_created", order_id=order.id, user_id=user.id)
    incorrect: |
      print(f"Created order {order.id}")
    reference: "docs/logging-guide.md"

  api_responses:
    description: "All API responses use our standard envelope"
    correct: |
      return ApiResponse(data=order.to_dict(), meta={"version": "v1"})
    incorrect: |
      return {"order": order.to_dict()}
    reference: "src/api/response.py"

  testing:
    description: "Use factories for test data, never raw dicts"
    correct: |
      order = OrderFactory.create(status="pending")
    incorrect: |
      order = {"id": 1, "status": "pending", "user_id": 42}
    reference: "tests/factories/README.md"
```

### Architecture Rules

```yaml
# .review/architecture.yaml
layers:
  routes:
    can_import: ["services", "schemas", "middleware"]
    cannot_import: ["repositories", "models", "database"]
    rule: "Routes handle HTTP concerns only. Business logic goes in services."

  services:
    can_import: ["repositories", "schemas", "events", "errors"]
    cannot_import: ["routes", "database", "models"]
    rule: "Services contain business logic. They work with repository abstractions, not raw models."

  repositories:
    can_import: ["models", "database", "errors"]
    cannot_import: ["routes", "services", "schemas"]
    rule: "Repositories encapsulate data access. They return domain objects, not ORM models."

boundaries:
  - name: "No cross-domain direct imports"
    rule: "The payments module cannot import from the users module directly. Use events or a shared interface."
    flag: "import from src.users in any file under src.payments"

  - name: "No circular service dependencies"
    rule: "If ServiceA calls ServiceB, ServiceB must not call ServiceA."
    flag: "bidirectional imports between service files"
```

### Naming Standards

```yaml
# .review/naming.yaml
naming:
  files:
    components: "PascalCase.tsx (e.g., OrderList.tsx)"
    services: "snake_case.py (e.g., order_service.py)"
    tests: "test_<module>.py or <Component>.test.tsx"
    migrations: "<timestamp>_<description>.sql"

  code:
    classes: "PascalCase (e.g., OrderService, PaymentRepository)"
    functions: "snake_case for Python, camelCase for TypeScript"
    constants: "UPPER_SNAKE_CASE (e.g., MAX_RETRY_COUNT)"
    private: "Leading underscore for Python (e.g., _validate_order)"
    boolean_vars: "Prefixed with is/has/should/can (e.g., is_active, has_permission)"

  api:
    endpoints: "kebab-case, plural nouns (e.g., /api/v1/order-items)"
    query_params: "snake_case (e.g., page_size, sort_by)"
    response_fields: "snake_case in JSON (e.g., created_at, order_id)"

  anti_patterns:
    - pattern: "data, info, stuff, thing, item"
      reason: "Too generic. Use domain-specific names."
    - pattern: "temp, tmp, foo, bar"
      reason: "Placeholder names should not survive code review."
    - pattern: "handle, process, manage"
      reason: "Too vague for function names. Be specific about what is handled."
```

### Import Ordering

```yaml
# .review/imports.yaml
python_import_order:
  groups:
    - name: "stdlib"
      description: "Python standard library"
      example: "import os, sys, json"
    - name: "third_party"
      description: "Installed packages"
      example: "from fastapi import FastAPI"
    - name: "first_party"
      description: "Our application code"
      example: "from src.services import OrderService"
    - name: "local"
      description: "Relative imports from the same package"
      example: "from .models import Order"
  separator: "Blank line between each group"
  enforced_by: "isort (automated, do not flag manually)"

typescript_import_order:
  groups:
    - name: "react"
      description: "React and React DOM"
      example: "import React from 'react'"
    - name: "third_party"
      description: "node_modules packages"
      example: "import { useQuery } from '@tanstack/react-query'"
    - name: "internal"
      description: "Absolute imports from our code"
      example: "import { OrderService } from '@/services/order'"
    - name: "relative"
      description: "Relative imports"
      example: "import { OrderRow } from './OrderRow'"
    - name: "styles"
      description: "CSS/SCSS imports last"
      example: "import styles from './Order.module.css'"
  enforced_by: "eslint-plugin-import (automated, do not flag manually)"
```

---

## Review Automation

Manual code review does not scale. Automate the repeatable parts so humans can focus on the judgment calls.

### CI Integration

**GitHub Actions workflow for AI-assisted review:**

```yaml
# .github/workflows/ai-review.yaml
name: AI Code Review

on:
  pull_request:
    types: [opened, synchronize]

permissions:
  contents: read
  pull-requests: write

jobs:
  ai-review:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Get changed files
        id: changed
        run: |
          FILES=$(gh pr diff ${{ github.event.pull_request.number }} --name-only)
          echo "files<<EOF" >> $GITHUB_OUTPUT
          echo "$FILES" >> $GITHUB_OUTPUT
          echo "EOF" >> $GITHUB_OUTPUT
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}

      - name: Load review config
        id: config
        run: |
          # Determine which checklists apply based on changed files
          python scripts/select-checklists.py \
            --files "${{ steps.changed.outputs.files }}" \
            --config .review/checklists/ \
            --output review-context.md

      - name: Run AI review
        run: |
          python scripts/ai-review.py \
            --pr ${{ github.event.pull_request.number }} \
            --context review-context.md \
            --tone .review/tone.yaml \
            --conventions .review/conventions.yaml
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

**The review script:**

```python
# scripts/ai-review.py
import os
import json
import subprocess
import anthropic

def get_pr_diff(pr_number: int) -> str:
    """Get the diff for a pull request."""
    result = subprocess.run(
        ["gh", "pr", "diff", str(pr_number)],
        capture_output=True, text=True
    )
    return result.stdout

def get_pr_files(pr_number: int) -> list[str]:
    """Get the list of changed files."""
    result = subprocess.run(
        ["gh", "pr", "diff", str(pr_number), "--name-only"],
        capture_output=True, text=True
    )
    return result.stdout.strip().split("\n")

def load_context(context_path: str) -> str:
    """Load review context (checklists, conventions, etc.)."""
    with open(context_path) as f:
        return f.read()

def build_review_prompt(diff: str, context: str, tone_config: dict) -> str:
    """Build the full review prompt with all context."""
    return f"""You are a code reviewer for our team.

## Review Guidelines
{context}

## Tone
Voice: {tone_config.get('voice', 'mentoring')}
Principles:
{chr(10).join('- ' + p for p in tone_config.get('principles', []))}

## Pull Request Diff
```diff
{diff}
```

Review this diff following our team guidelines. For each finding:
1. Tag as MUST / SHOULD / COULD / PRAISE
2. Reference the file and line number
3. Explain why it matters
4. Suggest a fix when applicable

Include at least one PRAISE comment if there is anything positive.
End with a brief overall assessment.
"""

def post_review_comments(pr_number: int, review: str):
    """Post the review as a PR comment."""
    subprocess.run(
        ["gh", "pr", "comment", str(pr_number), "--body", review],
        check=True
    )

def main():
    import argparse, yaml
    parser = argparse.ArgumentParser()
    parser.add_argument("--pr", type=int, required=True)
    parser.add_argument("--context", required=True)
    parser.add_argument("--tone", required=True)
    parser.add_argument("--conventions", required=True)
    args = parser.parse_args()

    diff = get_pr_diff(args.pr)
    context = load_context(args.context)

    with open(args.tone) as f:
        tone_config = yaml.safe_load(f)

    client = anthropic.Anthropic()
    prompt = build_review_prompt(diff, context, tone_config)

    response = client.messages.create(
        model="claude-sonnet-4-20250514",
        max_tokens=4096,
        messages=[{"role": "user", "content": prompt}],
    )

    review_text = response.content[0].text
    post_review_comments(args.pr, review_text)

if __name__ == "__main__":
    main()
```

### Auto-Review on PR

Trigger reviews automatically but control when to run full vs lightweight reviews.

```yaml
# .review/auto-review.yaml
triggers:
  on_open:
    action: full_review
    description: "Full review when PR is opened"

  on_push:
    action: incremental_review
    description: "Review only new commits, not the entire diff"

  on_label:
    label: "needs-ai-review"
    action: full_review
    description: "Manual trigger via label for re-review"

skip_conditions:
  - label: "skip-ai-review"
  - title_prefix: "chore:"
  - title_prefix: "docs:"
  - author_bot: true
  - draft: true

size_limits:
  max_files: 50
  max_diff_lines: 3000
  on_exceed: "Post summary only, skip line-by-line review"
```

### Selective Review (Only Changed Files)

Do not review the entire codebase on every PR. Focus on what changed and what it touches.

```python
# scripts/select-checklists.py
"""Select relevant review checklists based on changed files."""

import argparse
import yaml
from pathlib import Path
from fnmatch import fnmatch

def match_checklists(changed_files: list[str], checklist_dir: str) -> list[dict]:
    """Match changed files to applicable checklists."""
    checklists = []
    for checklist_path in Path(checklist_dir).glob("*.yaml"):
        with open(checklist_path) as f:
            checklist = yaml.safe_load(f)

        applies_to = checklist.get("applies_to", {})
        patterns = applies_to.get("paths", ["**/*"])
        file_types = applies_to.get("file_types", [])

        for changed_file in changed_files:
            matches_pattern = any(
                fnmatch(changed_file, p) for p in patterns
            )
            matches_type = (
                not file_types
                or any(changed_file.endswith(t) for t in file_types)
            )
            if matches_pattern and matches_type:
                checklists.append(checklist)
                break

    return checklists

def generate_context(checklists: list[dict]) -> str:
    """Generate review context from matched checklists."""
    sections = []
    for cl in checklists:
        section = f"### {cl['name']}\n"
        for category, items in cl.get("checks", {}).items():
            section += f"\n**{category.replace('_', ' ').title()}:**\n"
            for item in items:
                section += f"- [ ] {item}\n"
        sections.append(section)
    return "\n".join(sections)

if __name__ == "__main__":
    parser = argparse.ArgumentParser()
    parser.add_argument("--files", required=True)
    parser.add_argument("--config", required=True)
    parser.add_argument("--output", required=True)
    args = parser.parse_args()

    changed_files = args.files.strip().split("\n")
    checklists = match_checklists(changed_files, args.config)
    context = generate_context(checklists)

    with open(args.output, "w") as f:
        f.write(context)

    print(f"Selected {len(checklists)} checklists for review context")
```

### Review Comments API

Post structured review comments programmatically, not just plain text.

```python
# scripts/post-review.py
"""Post structured review comments using the GitHub API."""

import json
import subprocess

def post_line_comment(
    pr_number: int,
    file_path: str,
    line: int,
    body: str,
    side: str = "RIGHT",
):
    """Post a review comment on a specific line."""
    subprocess.run([
        "gh", "api",
        f"repos/{{owner}}/{{repo}}/pulls/{pr_number}/comments",
        "-f", f"body={body}",
        "-f", f"path={file_path}",
        "-F", f"line={line}",
        "-f", f"side={side}",
        "-f", "commit_id=$(gh pr view {pr_number} --json headRefOid -q .headRefOid)",
    ], check=True)

def post_review_with_comments(
    pr_number: int,
    comments: list[dict],
    overall_body: str,
    event: str = "COMMENT",
):
    """
    Post a full review with inline comments.

    event: APPROVE, REQUEST_CHANGES, or COMMENT
    comments: [{"path": "file.py", "line": 10, "body": "..."}]
    """
    payload = {
        "body": overall_body,
        "event": event,
        "comments": [
            {
                "path": c["path"],
                "line": c["line"],
                "body": c["body"],
            }
            for c in comments
        ],
    }

    subprocess.run([
        "gh", "api",
        f"repos/{{owner}}/{{repo}}/pulls/{pr_number}/reviews",
        "--input", "-",
    ], input=json.dumps(payload), text=True, check=True)

def parse_ai_review(review_text: str) -> tuple[str, list[dict]]:
    """
    Parse AI review output into overall summary and line comments.
    Expected format from AI:

    OVERALL: Clean implementation with one security concern.

    FILE: src/api/routes/orders.py
    LINE: 42
    TAG: MUST
    COMMENT: The user_id from the request body is not validated...

    FILE: src/services/order_service.py
    LINE: 15
    TAG: PRAISE
    COMMENT: Good use of the repository pattern here...
    """
    overall = ""
    comments = []
    current_comment = {}

    for line in review_text.split("\n"):
        if line.startswith("OVERALL:"):
            overall = line.replace("OVERALL:", "").strip()
        elif line.startswith("FILE:"):
            if current_comment.get("path"):
                comments.append(current_comment)
            current_comment = {"path": line.replace("FILE:", "").strip()}
        elif line.startswith("LINE:"):
            current_comment["line"] = int(line.replace("LINE:", "").strip())
        elif line.startswith("TAG:"):
            tag = line.replace("TAG:", "").strip()
            current_comment["tag"] = tag
        elif line.startswith("COMMENT:"):
            current_comment["body"] = (
                f"**{current_comment.get('tag', 'NOTE')}**: "
                + line.replace("COMMENT:", "").strip()
            )

    if current_comment.get("path"):
        comments.append(current_comment)

    return overall, comments
```

---

## Learning from Reviews

A review process that does not learn from itself is just overhead. Track patterns, build knowledge, and feed learnings back into your review config.

### Tracking Common Issues

Log every review finding to build a dataset of what your team catches most often.

```yaml
# .review/tracking.yaml
tracking:
  enabled: true
  storage: ".review/data/findings.jsonl"
  fields:
    - timestamp
    - pr_number
    - file_path
    - severity        # MUST, SHOULD, COULD
    - category        # security, correctness, architecture, etc.
    - description
    - was_fixed       # Did the author address it?
    - fix_time_hours  # How long to resolve?
```

```python
# scripts/track-finding.py
"""Append a review finding to the tracking log."""

import json
from datetime import datetime
from pathlib import Path

FINDINGS_PATH = Path(".review/data/findings.jsonl")

def log_finding(
    pr_number: int,
    file_path: str,
    severity: str,
    category: str,
    description: str,
):
    """Log a single review finding."""
    FINDINGS_PATH.parent.mkdir(parents=True, exist_ok=True)

    finding = {
        "timestamp": datetime.utcnow().isoformat(),
        "pr_number": pr_number,
        "file_path": file_path,
        "severity": severity,
        "category": category,
        "description": description,
        "was_fixed": None,
        "fix_time_hours": None,
    }

    with open(FINDINGS_PATH, "a") as f:
        f.write(json.dumps(finding) + "\n")

def analyze_findings() -> dict:
    """Analyze accumulated findings for patterns."""
    if not FINDINGS_PATH.exists():
        return {"total": 0}

    findings = []
    with open(FINDINGS_PATH) as f:
        for line in f:
            findings.append(json.loads(line))

    # Most common categories
    category_counts = {}
    for f in findings:
        cat = f["category"]
        category_counts[cat] = category_counts.get(cat, 0) + 1

    # Fix rate
    fixed = sum(1 for f in findings if f.get("was_fixed") is True)
    total_resolved = sum(1 for f in findings if f.get("was_fixed") is not None)

    # Most flagged files
    file_counts = {}
    for f in findings:
        fp = f["file_path"]
        file_counts[fp] = file_counts.get(fp, 0) + 1

    return {
        "total": len(findings),
        "by_category": dict(sorted(
            category_counts.items(), key=lambda x: -x[1]
        )[:10]),
        "fix_rate": fixed / total_resolved if total_resolved > 0 else 0,
        "hotspot_files": dict(sorted(
            file_counts.items(), key=lambda x: -x[1]
        )[:10]),
    }
```

### Building a Review Knowledge Base

Turn recurring review comments into reusable documentation that prevents the same issues from appearing.

```yaml
# .review/knowledge-base/error-handling.yaml
topic: Error Handling
frequency: high  # How often this comes up in reviews
last_updated: 2025-12-01

common_mistakes:
  - name: "Catching all exceptions"
    description: "Bare except or except Exception hides bugs"
    bad: |
      try:
          result = process(data)
      except Exception:
          return None
    good: |
      try:
          result = process(data)
      except ValidationError as e:
          logger.warning("validation_failed", error=str(e))
          raise
      except ConnectionError as e:
          logger.error("connection_failed", error=str(e))
          return None  # Retry handled upstream
    reference: "docs/error-handling.md"

  - name: "Swallowing errors silently"
    description: "Catching an error without logging or re-raising"
    bad: |
      try:
          send_notification(user)
      except Exception:
          pass
    good: |
      try:
          send_notification(user)
      except NotificationError as e:
          logger.warning("notification_failed", user_id=user.id, error=str(e))
          # Non-critical, continue without blocking
    reference: "docs/error-handling.md#silent-failures"

  - name: "String-based error matching"
    description: "Checking error messages instead of error types"
    bad: |
      try:
          result = api_call()
      except Exception as e:
          if "timeout" in str(e):
              retry()
    good: |
      try:
          result = api_call()
      except TimeoutError:
          retry()
    reference: "docs/error-handling.md#error-types"

auto_comment: >
  This file contains patterns our team encounters frequently in
  reviews. Before submitting a PR that touches error handling, check
  these patterns to catch common issues early.
```

### Feedback Loops

Close the loop between review findings and team practices.

```yaml
# .review/feedback-loop.yaml
feedback_loop:
  weekly_digest:
    enabled: true
    schedule: "Monday 9:00 AM"
    channel: "#engineering"
    contents:
      - "Top 5 most common review findings this week"
      - "Files with the most review comments (hotspots)"
      - "New patterns added to the knowledge base"
      - "Fix rate trend (are authors addressing comments?)"

  checklist_updates:
    trigger: "When a category exceeds 10 findings in 30 days"
    action: "Propose a new checklist item via PR"
    template: |
      The category "{category}" has been flagged {count} times in
      the last 30 days. Here are the most common findings:
      {top_findings}

      Proposed checklist addition:
      - [ ] {proposed_check}

  onboarding_integration:
    trigger: "New team member's first PR"
    action: "Include links to knowledge base articles for flagged patterns"
    message: |
      Welcome to the team! Here are some resources for the patterns
      flagged in this review:
      {knowledge_base_links}

  prompt_refinement:
    trigger: "Monthly"
    action: "Analyze false positive rate and update AI review prompt"
    steps:
      - "Export all findings marked as 'dismissed' or 'won't fix'"
      - "Identify patterns the AI flags that the team consistently ignores"
      - "Update the review prompt to exclude or deprioritize those patterns"
      - "Track if false positive rate decreases in the next month"
```

---

## Review Metrics

What gets measured gets improved. Track the right metrics to understand whether your review process is helping or hindering your team.

### Review Turnaround Time

How long from PR opened to first review comment? Long turnaround kills velocity.

```python
# scripts/metrics/turnaround.py
"""Calculate review turnaround time metrics."""

import subprocess
import json
from datetime import datetime

def get_review_turnaround(repo: str, days: int = 30) -> dict:
    """Calculate average time from PR open to first review."""
    result = subprocess.run(
        [
            "gh", "api", f"repos/{repo}/pulls",
            "-q", f'[.[] | select(.state == "closed") | '
                  f'select(.merged_at != null)][:50]',
            "--paginate",
        ],
        capture_output=True, text=True,
    )
    prs = json.loads(result.stdout)

    turnaround_times = []
    for pr in prs:
        pr_number = pr["number"]
        created_at = datetime.fromisoformat(
            pr["created_at"].replace("Z", "+00:00")
        )

        # Get first review
        reviews_result = subprocess.run(
            [
                "gh", "api",
                f"repos/{repo}/pulls/{pr_number}/reviews",
                "-q", ".[0].submitted_at",
            ],
            capture_output=True, text=True,
        )
        first_review = reviews_result.stdout.strip()

        if first_review and first_review != "null":
            review_at = datetime.fromisoformat(
                first_review.replace("Z", "+00:00")
            )
            turnaround = (review_at - created_at).total_seconds() / 3600
            turnaround_times.append(turnaround)

    if not turnaround_times:
        return {"avg_hours": 0, "p50_hours": 0, "p90_hours": 0}

    turnaround_times.sort()
    n = len(turnaround_times)
    return {
        "avg_hours": round(sum(turnaround_times) / n, 1),
        "p50_hours": round(turnaround_times[n // 2], 1),
        "p90_hours": round(turnaround_times[int(n * 0.9)], 1),
        "sample_size": n,
    }
```

**Target benchmarks:**

| Metric | Good | Acceptable | Needs work |
|--------|------|-----------|------------|
| Avg turnaround | < 4 hours | 4-8 hours | > 8 hours |
| P90 turnaround | < 24 hours | 24-48 hours | > 48 hours |

### Nitpick Ratio

What percentage of your review comments are nitpicks? A high ratio means your automation is not catching enough, or your reviewers are focusing on the wrong things.

```python
# scripts/metrics/nitpick_ratio.py
"""Calculate the ratio of nitpick comments to substantive comments."""

import json
from pathlib import Path

def calculate_nitpick_ratio(findings_path: str) -> dict:
    """Calculate nitpick ratio from findings log."""
    findings = []
    with open(findings_path) as f:
        for line in f:
            findings.append(json.loads(line))

    nitpicks = sum(
        1 for f in findings if f.get("severity") == "nitpick"
    )
    substantive = sum(
        1 for f in findings
        if f.get("severity") in ("blocker", "concern")
    )
    suggestions = sum(
        1 for f in findings if f.get("severity") == "suggestion"
    )
    total = len(findings)

    return {
        "total_comments": total,
        "nitpick_count": nitpicks,
        "substantive_count": substantive,
        "suggestion_count": suggestions,
        "nitpick_ratio": round(nitpicks / total, 2) if total > 0 else 0,
        "substantive_ratio": round(
            substantive / total, 2
        ) if total > 0 else 0,
    }
```

**Target benchmarks:**

| Metric | Healthy | Warning | Action needed |
|--------|---------|---------|---------------|
| Nitpick ratio | < 15% | 15-30% | > 30% |
| Substantive ratio | > 50% | 30-50% | < 30% |

**Anti-pattern -- a review that is mostly nitpicks:**

```
NIT: Line 5 -- extra blank line
NIT: Line 12 -- use single quotes
NIT: Line 18 -- trailing whitespace
NIT: Line 24 -- import ordering
SHOULD: Line 45 -- missing null check on user input
```

This review has an 80% nitpick ratio. The four nitpicks should be caught by automated formatters, not human or AI reviewers. Fix your linter config instead of burning review bandwidth.

### Comment Resolution Rate

What percentage of review comments actually get addressed? Low resolution means the team does not trust the review process.

```python
# scripts/metrics/resolution.py
"""Track comment resolution rates."""

import json
from pathlib import Path
from collections import defaultdict

def calculate_resolution_rate(findings_path: str) -> dict:
    """Calculate how often review comments are resolved."""
    findings = []
    with open(findings_path) as f:
        for line in f:
            findings.append(json.loads(line))

    resolved = [f for f in findings if f.get("was_fixed") is not None]
    if not resolved:
        return {"resolution_rate": 0, "total_resolved": 0}

    fixed = sum(1 for f in resolved if f["was_fixed"] is True)
    dismissed = sum(1 for f in resolved if f["was_fixed"] is False)

    # Resolution by severity
    by_severity = defaultdict(lambda: {"fixed": 0, "dismissed": 0})
    for f in resolved:
        severity = f.get("severity", "unknown")
        if f["was_fixed"]:
            by_severity[severity]["fixed"] += 1
        else:
            by_severity[severity]["dismissed"] += 1

    return {
        "total_resolved": len(resolved),
        "fixed_count": fixed,
        "dismissed_count": dismissed,
        "resolution_rate": round(fixed / len(resolved), 2),
        "by_severity": {
            sev: {
                "fixed": counts["fixed"],
                "dismissed": counts["dismissed"],
                "rate": round(
                    counts["fixed"]
                    / (counts["fixed"] + counts["dismissed"]),
                    2,
                ),
            }
            for sev, counts in by_severity.items()
        },
    }
```

**Target benchmarks:**

| Severity | Expected resolution rate |
|----------|------------------------|
| Blocker | > 95% |
| Concern | > 70% |
| Suggestion | > 40% |
| Nitpick | > 20% |

If blocker resolution is below 95%, your team either disagrees on what constitutes a blocker, or the review process has no enforcement. Both need attention.

### Team Satisfaction

Quantitative metrics are not enough. Periodically survey the team.

```yaml
# .review/surveys/quarterly-review.yaml
name: Code Review Health Check
frequency: quarterly
questions:
  - id: useful
    text: "Are code reviews helping you write better code?"
    type: likert  # 1-5 scale
    target: ">= 3.5"

  - id: turnaround
    text: "Are you satisfied with how quickly you receive reviews?"
    type: likert
    target: ">= 3.0"

  - id: tone
    text: "Do review comments feel constructive and respectful?"
    type: likert
    target: ">= 4.0"

  - id: ai_quality
    text: "Are AI-generated review comments helpful?"
    type: likert
    target: ">= 3.0"

  - id: blocking
    text: "How often do reviews block your work for more than a day?"
    type: frequency  # Never, Rarely, Sometimes, Often, Always
    target: "<= Rarely"

  - id: top_improvement
    text: "What is the single biggest improvement we could make to our review process?"
    type: free_text
    action: "Discuss top themes in the next retro"

scoring:
  healthy: "All targets met"
  needs_attention: "1-2 targets missed"
  action_required: "3+ targets missed -- schedule a review process retro"
```

**Dashboard summary (example output):**

```
=== Code Review Health Dashboard ===
Period: Q1 2026

Turnaround:
  Avg: 3.2 hours (target: < 4h)       [PASS]
  P90: 18 hours (target: < 24h)        [PASS]

Quality:
  Nitpick ratio: 12% (target: < 15%)   [PASS]
  Substantive ratio: 58% (target: > 50%) [PASS]
  False positive rate: 8% (target: < 10%) [PASS]

Resolution:
  Blocker fix rate: 97% (target: > 95%) [PASS]
  Concern fix rate: 74% (target: > 70%) [PASS]
  Overall fix rate: 62%                 [INFO]

Satisfaction (survey avg):
  Usefulness: 4.1 / 5                  [PASS]
  Turnaround: 3.8 / 5                  [PASS]
  Tone: 4.3 / 5                        [PASS]
  AI quality: 3.2 / 5                  [PASS]

Hotspot files (most review comments):
  1. src/payments/charge.py (23 comments)
  2. src/api/routes/orders.py (18 comments)
  3. src/auth/middleware.py (15 comments)

Top finding categories:
  1. Error handling (31%)
  2. Missing tests (22%)
  3. Security (18%)
  4. Architecture (14%)
  5. Naming (8%)

Action items:
  - Consider extracting src/payments/charge.py into smaller modules
  - Add error handling patterns to onboarding docs
  - Schedule a testing workshop (22% of findings are missing tests)
```
