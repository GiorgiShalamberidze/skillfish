---
name: vibe-coding-guide
description: Structured AI-pair-programming: prompt decomposition, iterative refinement, context management, quality gates, and session workflows.
---

# Vibe Coding Guide

Vibe coding is the practice of using AI coding assistants as a pair programmer in a fluid, conversational workflow. Instead of writing every line by hand, you describe intent, review output, steer corrections, and build iteratively through dialogue. The term captures the improvisational, back-and-forth rhythm: you set the vibe, the AI generates, you refine.

But unstructured vibe coding produces mediocre results. Developers who treat the AI like a magic box ("build me an app") get brittle, incoherent code. Developers who apply structure to the conversation -- decomposing problems, managing context, verifying outputs, and following deliberate workflows -- get production-grade code at 3-10x speed.

This skill provides that structure. It is a comprehensive methodology for AI-pair-programming sessions that consistently produce high-quality, maintainable software.

## Table of Contents

- [When to Use](#when-to-use)
- [Session Setup](#session-setup)
- [Prompt Decomposition](#prompt-decomposition)
- [Context Management](#context-management)
- [Quality Gates](#quality-gates)
- [Anti-Patterns](#anti-patterns)
- [Advanced Techniques](#advanced-techniques)
- [Session Templates](#session-templates)

---

## When to Use

Activate vibe coding methodology when:

- **Starting a new feature** -- you have requirements and need to translate them into working code across multiple files
- **Exploring an unfamiliar codebase** -- you need to understand existing patterns before modifying them
- **Prototyping and spiking** -- you want to test feasibility quickly while keeping code structured enough to evolve
- **Refactoring** -- you need to restructure code without changing behavior, and want a second pair of eyes on each transformation
- **Debugging complex issues** -- you need to systematically narrow down root causes across layers
- **Learning a new framework or language** -- you want idiomatic code examples tailored to your actual project, not generic tutorials
- **Writing tests for existing code** -- you need comprehensive coverage and the AI can identify edge cases you might miss
- **Building boilerplate-heavy features** -- CRUD endpoints, form handling, database migrations, CI configs

**Do not use** for trivial one-line changes, security-critical cryptographic code (verify independently), or when you need to deeply understand every line yourself (learning exercises).

---

## Session Setup

The first five minutes of a vibe coding session determine its success. Rushing into prompts without context produces generic, off-target code.

### Step 1: Share the Architecture

Before writing any code, ground the AI in your system. Provide:

```
Here's the project structure:
- /src/api/ -- Express routes, one file per resource
- /src/services/ -- Business logic, no HTTP concerns
- /src/models/ -- Sequelize models with associations
- /src/middleware/ -- Auth, validation, error handling
- Tests live next to source files as *.test.ts

We use TypeScript strict mode, Zod for validation, and return
consistent { data, error, meta } response envelopes.
```

### Step 2: Share Relevant Files

Don't dump the entire codebase. Share the 2-5 files most relevant to what you're building:

```
Here's our existing auth middleware (auth.middleware.ts):
[paste file]

And our response envelope helper (response.ts):
[paste file]

New code should follow these same patterns.
```

### Step 3: Set Constraints

Be explicit about what the AI should and should not do:

```
Constraints:
- Use the existing database connection from src/db.ts, don't create a new one
- Follow our error handling pattern: throw AppError, let middleware catch it
- No new dependencies unless absolutely necessary
- Keep functions under 30 lines
- All new code needs TypeScript types, no `any`
```

### Step 4: Define Acceptance Criteria

State what "done" looks like before generating any code:

```
Acceptance criteria for this feature:
1. POST /api/v1/teams creates a team with name and description
2. Creator is automatically added as team owner
3. Returns 201 with the team object including member count
4. Returns 409 if team name already exists for this org
5. Validated with Zod schema, returns 400 with field errors
6. Has unit tests for service layer and integration test for endpoint
```

### Step 5: Establish the Working Agreement

Tell the AI how you want to work:

```
Working agreement for this session:
- Generate one file at a time, wait for my review before moving on
- Include brief comments explaining non-obvious decisions
- If you're unsure about a pattern, ask rather than guess
- When I say "next", move to the next file in the plan
```

---

## Prompt Decomposition

The single biggest mistake in vibe coding is asking for too much in one prompt. The result is a wall of code that's hard to review, hard to correct, and hard to build on.

### The Specify-Generate-Verify-Refine Loop

Every meaningful unit of work follows this cycle:

1. **Specify** -- describe exactly what you need, with constraints and examples
2. **Generate** -- let the AI produce the code
3. **Verify** -- review the output, run it, test it
4. **Refine** -- request targeted corrections or move to the next unit

### Prompt Sizing

**Too large (avoid):**
```
Build me a complete user management system with registration,
login, password reset, email verification, role-based access
control, and an admin dashboard.
```

This produces thousands of lines that are impossible to review meaningfully.

**Too small (avoid):**
```
Write an if statement that checks if the user is null.
```

This micromanagement wastes context window and your time.

**Right-sized:**
```
Create the Zod validation schema for team creation.
Fields: name (string, 2-50 chars, trimmed), description
(string, optional, max 500 chars). Export the schema
and the inferred TypeScript type.
```

One coherent unit. Reviewable in 30 seconds. Testable independently.

### Dependency Ordering

Break complex features into prompts that build on each other:

1. **Data model first** -- schema, types, migrations
2. **Validation layer** -- input schemas, type inference
3. **Service/business logic** -- pure functions, no I/O concerns
4. **Data access** -- repository or model methods
5. **API/controller layer** -- routes, request handling
6. **Tests** -- unit tests for service, integration tests for API
7. **Glue** -- wiring, middleware, error handling updates

Each step produces a small, verifiable artifact that the next step depends on.

### Prompt Templates for Decomposition

When starting a complex task, ask the AI to help decompose it:

```
I need to add team management to our app. Before writing any
code, break this down into a sequence of atomic implementation
steps. Each step should produce one file or one small change.
Order them by dependency -- no step should reference code that
hasn't been written yet.
```

Review the plan, adjust the ordering, then execute step by step.

---

## Context Management

AI coding assistants have finite context windows. Managing what's in that window is a core vibe coding skill.

### What to Include

- **The file you're working on** -- always include the current file in full
- **Interfaces and types** -- the contracts that the current code must satisfy
- **Immediate dependencies** -- files that the current code imports or calls
- **Relevant tests** -- if you're fixing a bug, include the failing test
- **Error messages** -- full stack traces, not summaries

### What to Exclude

- **Unrelated files** -- code that has no connection to the current task
- **Generated files** -- lock files, build output, compiled assets
- **Large data fixtures** -- provide a small representative sample instead
- **Entire directories** -- pick the specific files that matter

### The Context Budget Rule

Think of your context window as a budget. Every token spent on irrelevant content is a token unavailable for reasoning about your actual problem.

| Context Size | Guidance |
|-------------|----------|
| Small (< 10% of window) | Include liberally, add examples |
| Medium (10-40%) | Be selective, focus on interfaces and the working file |
| Large (40-70%) | Strip comments, omit implementations you're not changing |
| Critical (> 70%) | Summarize and restart (see below) |

### When Context Windows Fill Up

Long sessions accumulate stale context -- old attempts, abandoned approaches, irrelevant discussion. When you notice the AI:

- Forgetting earlier decisions
- Contradicting previous outputs
- Generating code that conflicts with what was already built
- Losing track of variable names or function signatures

It's time to reset. Use this pattern:

```
Let's start a fresh context. Here's where we are:

## Completed (don't regenerate these)
- Team model: src/models/team.ts (done, tested, working)
- Team schema: src/schemas/team.ts (done, tested, working)

## Current state of team service (needs the delete method added):
[paste current file]

## What's left to build:
- deleteTeam service method with soft-delete
- DELETE /api/v1/teams/:id endpoint
- Tests for both

Continue from the delete service method.
```

### File-by-File vs. Whole-Project Context

**File-by-file** (recommended for most work):
- Share 1-3 files at a time
- Higher quality output per file
- Easier to verify and correct
- Works within any context window

**Whole-project** (use sparingly):
- Useful for understanding cross-cutting concerns
- Good for initial architecture discussions
- Quality degrades as project size increases
- Reserve for planning phases, not implementation

### Managing Conversation Drift

Long conversations naturally drift from the original goal. Prevent this by:

1. **Restating the goal periodically** -- every 5-10 exchanges, remind the AI what you're building
2. **Numbering your steps** -- "Step 3 of 7: Now build the service method"
3. **Rejecting tangents** -- "That's interesting but let's stay focused on the delete endpoint"
4. **Using explicit transitions** -- "The model layer is complete. Moving to the API layer now."

---

## Quality Gates

Vibe coding at speed without quality gates produces code that looks right but breaks in production. Every AI output must pass through verification before you build on it.

### The Review Gate

Before accepting any generated code, check:

- [ ] Does it compile/parse without errors?
- [ ] Does it follow the project's existing patterns?
- [ ] Are there any hardcoded values that should be configurable?
- [ ] Does the error handling cover realistic failure modes?
- [ ] Would you be comfortable explaining this code in a review?

If any answer is no, refine before moving on. **Never build on broken foundations.**

### Test-First Vibe Coding

The most effective quality pattern for vibe coding:

1. **Write the test first** (or have the AI write it):
```
Write a test for a deleteTeam service method.
It should:
- Soft-delete (set deletedAt timestamp, don't remove row)
- Return the deleted team object
- Throw NotFoundError if team doesn't exist
- Throw ForbiddenError if user isn't team owner
Use our existing test helpers from tests/helpers.ts.
```

2. **Verify the test makes sense** -- read it, make sure the assertions match your requirements

3. **Generate the implementation**:
```
Now implement the deleteTeam method that passes these tests.
Here's the test file for reference:
[paste the test]
```

4. **Run the tests** -- don't trust "this should pass." Run them.

5. **Iterate if needed** -- if tests fail, share the failure output and let the AI fix it.

### Incremental Commits

Commit after each verified unit of work:

```bash
# After model is verified and tested
git add src/models/team.ts src/models/team.test.ts
git commit -m "feat: add team model with soft-delete support"

# After service layer is verified and tested
git add src/services/team.ts src/services/team.test.ts
git commit -m "feat: add team service with CRUD operations"

# After API layer is verified and tested
git add src/api/teams.ts src/api/teams.test.ts
git commit -m "feat: add team API endpoints"
```

This gives you rollback points. If step 3 goes sideways, you can revert without losing steps 1-2.

### Trust But Verify Patterns

| AI Output | Verification Method |
|-----------|-------------------|
| Business logic | Unit tests with edge cases |
| API endpoints | Integration tests with realistic payloads |
| Database queries | Run against test DB, check query plans |
| Type definitions | Compile with strict mode |
| Regex patterns | Test with positive AND negative cases |
| Security code | Manual review + known vulnerability checklist |
| Performance claims | Benchmark, don't believe estimates |

### The "Looks Right" Trap

AI-generated code often passes visual inspection while containing subtle issues:

- Off-by-one errors in pagination
- Race conditions in async flows
- Missing null checks on optional chains
- Incorrect operator precedence
- Silent swallowing of errors in catch blocks

Always run the code. Read the test output. Check the actual database state. **Compiles is not correct. Looks right is not tested.**

---

## Anti-Patterns

Patterns that reliably produce bad outcomes in vibe coding sessions.

### The "Just Fix It" Trap

**Pattern:** Pasting an error and saying "fix it" without providing context.

**Why it fails:** The AI guesses at the root cause. It patches the symptom. The actual bug remains, now hidden under a band-aid.

**Instead:**
```
I'm getting this error when calling deleteTeam:
[full error with stack trace]

Here's the function:
[paste deleteTeam]

And here's the calling code:
[paste the route handler]

The team exists in the database -- I verified with a direct query.
What's the root cause?
```

### Context Overload

**Pattern:** Pasting 20 files into the context "just in case."

**Why it fails:** The AI's attention degrades with irrelevant context. Important details get lost in noise. Output quality drops significantly.

**Instead:** Share only what's directly relevant. If you're not sure what's relevant, describe the problem first and ask the AI which files it needs.

### Skipping Review

**Pattern:** Accepting generated code without reading it and immediately asking for the next piece.

**Why it fails:** Errors compound. Each new piece of code builds on assumptions from the previous piece. One wrong assumption in step 2 corrupts steps 3 through 10.

**Instead:** Spend 30-60 seconds reviewing each output. It's faster to catch issues early than to debug cascading failures later.

### Building on Broken Foundations

**Pattern:** Seeing an issue in generated code, noting it mentally as "I'll fix that later," and asking for the next piece.

**Why it fails:** The AI doesn't know about your mental note. It builds on the code as-written, propagating the bug. By the time you circle back, the fix requires changing five files instead of one.

**Instead:** Fix issues immediately. A 30-second correction now saves a 30-minute debugging session later.

### Prompt Ambiguity

**Pattern:** "Add error handling to this function."

**Why it fails:** What errors? What should happen when they occur? Log and continue? Throw to caller? Return a default? The AI picks something, and it's often not what you wanted.

**Instead:**
```
Add error handling to createTeam:
- If db insert fails with unique violation, throw ConflictError("Team name already exists")
- If db insert fails for any other reason, throw InternalError and log the original error
- If addMember fails after team creation, delete the team and throw InternalError
```

### The "Rewrite Everything" Impulse

**Pattern:** "This code is messy, rewrite the whole module."

**Why it fails:** The AI doesn't know which behaviors are intentional and which are bugs. It produces clean code that breaks existing integrations, removes intentional workarounds, and changes API contracts.

**Instead:** Refactor incrementally. One function at a time. Run tests after each change. Preserve behavior first, then improve structure.

### Ignoring the AI's Questions

**Pattern:** The AI asks a clarifying question. You ignore it and restate your original request more forcefully.

**Why it fails:** The AI asked because the request was genuinely ambiguous. Forcing it to guess produces one of multiple possible interpretations, and often not the one you wanted.

**Instead:** Answer the question. The 15 seconds you spend clarifying saves minutes of rework.

---

## Advanced Techniques

### Multi-Agent Workflows

Use two AI sessions for complex work:

**Agent 1 -- Builder:**
- Generates the implementation
- Focuses on writing clean, working code
- Optimizes for speed and correctness

**Agent 2 -- Reviewer:**
- Reviews Agent 1's output
- Focuses on finding issues, edge cases, security concerns
- Provides feedback that you relay back to Agent 1

This mimics a real pair programming dynamic. The builder has blind spots that the reviewer catches.

Example reviewer prompt:
```
Review this code for:
1. Security vulnerabilities (injection, auth bypass, data exposure)
2. Error handling gaps (what fails silently?)
3. Performance issues (N+1 queries, unnecessary allocations)
4. Deviation from the patterns in our codebase (I'll share the patterns)
5. Missing edge cases

Here's the code to review:
[paste]

Here are our established patterns:
[paste relevant examples]
```

### Subagent Parallelism

When using tools that support subagents (like Claude Code), parallelize independent work:

```
Run these tasks in parallel using subagents:
1. Generate the Team model with Sequelize in src/models/team.ts
2. Generate the Zod validation schemas in src/schemas/team.ts
3. Generate the test fixtures in tests/fixtures/team.ts

These have no dependencies on each other. I'll wire them together
after reviewing all three.
```

This cuts wall-clock time for independent tasks significantly.

### Iterative Architecture

Don't design the perfect architecture upfront. Use vibe coding to evolve it:

**Round 1 -- Naive implementation:**
```
Build the simplest possible version of team management.
One file, no abstractions, just make it work.
```

**Round 2 -- Extract patterns:**
```
Now that we have working code, let's refactor. Extract the
database logic into a repository pattern. Keep the same tests
passing.
```

**Round 3 -- Add sophistication:**
```
Add soft-delete support to the team repository. Update the
existing methods to filter out soft-deleted records by default,
with an option to include them.
```

Each round produces working, tested code. You never have a half-built abstraction that doesn't compile.

### AI-Assisted Debugging Workflows

When debugging, follow a structured diagnostic process:

**Step 1 -- Reproduce:**
```
I'm seeing this error in production:
[error log]

Help me write a minimal reproduction test that triggers this
exact error. Here's the relevant code:
[paste]
```

**Step 2 -- Hypothesize:**
```
The reproduction test passes/fails. Given these results,
what are the top 3 most likely root causes? For each,
tell me what I can check to confirm or rule it out.
```

**Step 3 -- Narrow:**
```
I checked hypothesis 1: [result]. Hypothesis 2: [result].
Given these findings, what's the root cause and what's
the correct fix?
```

**Step 4 -- Fix and verify:**
```
Apply the fix. Then write a regression test that would
catch this bug if it were reintroduced.
```

### Context Priming with Conventions

For long-running projects, maintain a conventions document that you paste at the start of each session:

```markdown
## Project Conventions

### API Patterns
- All routes: /api/v1/{resource}
- Response envelope: { data, error, meta }
- Pagination: cursor-based, meta.nextCursor
- Auth: Bearer token, middleware extracts req.user

### Code Patterns
- Services throw domain errors (NotFoundError, ConflictError)
- Controllers catch nothing -- global error middleware handles it
- All DB access through repository classes
- Repository methods return plain objects, not ORM instances

### Testing Patterns
- Unit tests: *.test.ts next to source
- Integration tests: tests/integration/*.test.ts
- Test DB: reset between suites, seed per test
- Factories: tests/factories/*.ts
```

This ensures consistent output across sessions without re-explaining your codebase each time.

### Prompt Chains for Complex Reasoning

For problems that require multi-step reasoning, chain your prompts explicitly:

```
Step 1: Analyze the current data flow for team invitations.
Trace the path from the API endpoint through the service
layer to the database. List each function call in order.
Don't write any code yet.
```

```
Step 2: Now identify where the race condition could occur
in that flow. What happens if two users accept the same
invitation simultaneously?
```

```
Step 3: Propose a fix using database-level locking. Show
me the SQL or ORM code, not the full function -- just the
critical section.
```

```
Step 4: Now write the complete updated function with the
fix applied, plus a test that verifies two concurrent
acceptances don't both succeed.
```

Each step builds on verified understanding from the previous step.

---

## Session Templates

Complete prompt sequences for common vibe coding tasks. Copy and adapt these to your project.

### Template: Building a New API Endpoint

**Prompt 1 -- Plan:**
```
I need to build a POST /api/v1/teams endpoint.
Here are our conventions: [paste conventions]
Here's an existing endpoint for reference: [paste similar endpoint]

Break this into implementation steps with file paths.
```

**Prompt 2 -- Data model:**
```
Create the Team model following our Sequelize patterns.
Fields: id (UUID), name (string, unique per org), description (text, nullable),
orgId (UUID, FK), createdBy (UUID, FK), deletedAt (timestamp, nullable).
Include associations: belongsTo Org, belongsTo User (as creator),
hasMany TeamMember.
```

**Prompt 3 -- Validation:**
```
Create the Zod schemas for team creation and update.
Create: name (required, 2-50 chars, trimmed), description (optional, max 500).
Update: same fields but all optional. Export inferred types.
```

**Prompt 4 -- Service:**
```
Create the team service with a createTeam method.
Input: { name, description, orgId, userId }.
Behavior: validate name uniqueness within org, create team,
add creator as owner in TeamMember, return team with member count.
Use a transaction for the create + addMember.
Throw ConflictError if name exists.
```

**Prompt 5 -- Route:**
```
Create the POST /api/v1/teams route handler.
Use our auth middleware (req.user available).
Validate body with Zod schema.
Call teamService.createTeam.
Return 201 with response envelope.
Register in src/api/index.ts.
```

**Prompt 6 -- Tests:**
```
Write tests for the team service and endpoint.
Service tests: success, duplicate name conflict, transaction rollback on member add failure.
Endpoint tests: 201 success, 400 validation errors, 409 conflict, 401 unauthorized.
Use our test helpers and factories.
```

### Template: Adding a React Feature

**Prompt 1 -- Plan:**
```
I need to add a team creation modal to the dashboard.
Here's our component structure: [paste relevant components]
We use: React 18, TypeScript, Tailwind, React Query, React Hook Form + Zod.

Break this into components and implementation steps.
```

**Prompt 2 -- Types and API client:**
```
Add the Team types and API client methods.
Types: Team, CreateTeamInput, TeamListResponse.
API client: createTeam(input) and listTeams(orgId) using our
existing api client pattern from src/lib/api.ts.
```

**Prompt 3 -- Form component:**
```
Create CreateTeamForm using React Hook Form with Zod resolver.
Fields: name (text input), description (textarea, optional).
Show field-level errors from Zod validation.
Show server errors (409 conflict) as a form-level error.
Submit button shows loading state.
On success, call onCreated callback.
```

**Prompt 4 -- Modal wrapper:**
```
Create CreateTeamModal that wraps the form in our Modal component.
Trigger: "New Team" button in the team list header.
Close on successful creation or cancel.
Use React Query's useMutation for the API call with
optimistic update to the team list.
```

**Prompt 5 -- Integration and tests:**
```
Wire the modal into the TeamList page.
Write tests: render form, validate inputs, submit success,
submit conflict error, loading state, modal open/close.
Use React Testing Library and MSW for API mocking.
```

### Template: Debugging a Production Issue

**Prompt 1 -- Context:**
```
We have a production bug: users are intermittently getting 500 errors
when creating teams. It works most of the time but fails roughly
10% of requests.

Here's a sample error from our logs:
[paste full error with stack trace]

Here's the relevant code:
[paste the endpoint and service]

What are the most likely causes of an intermittent failure here?
```

**Prompt 2 -- Diagnostic:**
```
Your top hypothesis was connection pool exhaustion. Here's what
I found:
- Pool max: 10, current active: 10, waiting: 3
- Other endpoints are also occasionally slow

How do I confirm this is the root cause? And what's the
immediate fix vs. the proper fix?
```

**Prompt 3 -- Fix:**
```
Confirmed: pool exhaustion from leaked connections in the
invitation flow (not closing on error path). Write the fix
for src/services/invitation.ts -- here's the current code:
[paste]

Make sure every path (success, error, timeout) releases
the connection.
```

**Prompt 4 -- Regression test:**
```
Write a test that verifies connections are released on all
code paths in the invitation service. Mock the pool to
track acquire/release calls.
```

### Template: Refactoring a Module

**Prompt 1 -- Audit:**
```
Here's our current auth module (4 files):
[paste each file]

Audit it for:
- Code smells and duplication
- Unclear responsibility boundaries
- Missing error handling
- Testability issues

Don't suggest fixes yet, just list findings ranked by severity.
```

**Prompt 2 -- Plan:**
```
Based on the audit, here's what I want to fix:
[pick specific findings]

Propose a refactoring plan. Each step must keep all
existing tests passing. No behavior changes -- just
structural improvements.
```

**Prompt 3-N -- Execute step by step:**
```
Execute refactoring step 1: [specific step from plan].
Here's the current state of the file:
[paste]

Show me the refactored version. I'll run tests before
we move to step 2.
```

---

## Summary

Effective vibe coding is not about typing less. It is about thinking clearly, communicating precisely, and verifying relentlessly. The AI amplifies whatever you bring to the session -- structured thinking produces structured code, vague prompts produce vague implementations.

The core loop is always: **set context, decompose, generate, verify, refine, commit.** Master this loop and you will build faster without sacrificing quality.

Key principles to internalize:

1. **Front-load context.** Five minutes of setup saves thirty minutes of rework.
2. **Right-size your prompts.** One coherent unit per prompt. Reviewable in under a minute.
3. **Manage your context window.** Include what matters, exclude what doesn't, reset when it fills up.
4. **Verify every output.** Run the code. Read the tests. Check the actual behavior.
5. **Fix issues immediately.** Never build on broken foundations.
6. **Commit incrementally.** Every verified unit gets its own commit.
7. **Use structure, not hope.** Templates, conventions, and quality gates beat wishful thinking.
