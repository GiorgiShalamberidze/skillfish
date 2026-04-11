---
name: readme-generator
description: Generate README files that explain setup, usage, architecture, workflows, and contribution details with developer-friendly structure.
---

# README Generator

Write README files that help the right reader succeed quickly. This skill prioritizes fast orientation, copy-pasteable setup steps, accurate command examples, honest architecture summaries, and contribution guidance that reflects the repository instead of generic template filler.

## When to Use

- Creating a README for a new repository, package, service, or CLI
- Rewriting a vague or outdated README into something actionable
- Documenting install, run, test, build, deploy, or contribution workflows
- Aligning README depth to the intended audience: users, contributors, or operators
- Standardizing README structure across multiple repos
- Improving repository first impression and onboarding speed

## Workflow

1. Identify the primary audience.
   Decide whether the README serves users, contributors, maintainers, operators, or a mix. The opening section should optimize for the primary audience.
2. Extract facts from the repo before writing.
   Verify install commands, runtime versions, scripts, environment variables, and entry points from actual files instead of guessing.
3. Lead with the fastest successful path.
   Put value proposition and quickstart ahead of deep architecture or contribution details.
4. Explain the working model.
   Summarize what the project does, how the main pieces fit, and where readers should go for deeper docs.
5. Add operationally useful sections only when true.
   Deployment, troubleshooting, release, or roadmap sections should exist because the repo needs them, not because a template says so.
6. Tighten for accuracy and scanability.
   Remove filler, keep examples runnable, and prefer headings, lists, and short code blocks over long prose.

## README Blueprint

1. **Project summary** - what it is, who it is for, and why it exists
2. **Quickstart** - install, configure, and run the simplest working path
3. **Common commands** - test, lint, build, dev, release, or package commands
4. **How it works** - concise architecture or directory overview
5. **Configuration** - env vars, config files, secrets expectations
6. **Contributing** - local workflow, tests, standards, and PR expectations

## Quality Bar

- Every command shown should exist and be runnable
- Required tools and versions are explicit
- Placeholder values are realistic and clearly marked
- Internal jargon is either defined or removed
- The opening screen answers "what is this?" and "how do I start?"
- Links point to the next useful document, not to generic sections

## Output Format

When using this skill, produce:

1. **Audience and goal** - who the README is for and what they need first
2. **README draft** - structured Markdown ready for the repo root
3. **Command validation list** - which commands or paths were verified from the repo
4. **Follow-up docs list** - deeper docs that should live outside the README
5. **Gaps or assumptions** - missing facts that need confirmation before publishing
