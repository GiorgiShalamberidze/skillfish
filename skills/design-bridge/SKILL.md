---
name: design-bridge
description: Design-to-engineering handoff: turn Figma and product intent into implementable components, tokens, states, and acceptance criteria.
---

# Design Bridge

Bridge design intent and implementation reality without visual drift or ambiguous handoff. This skill turns product and Figma decisions into component contracts, token usage, responsive rules, accessibility behavior, and acceptance criteria engineers can actually ship against.

## When to Use

- Translating designs into implementation-ready engineering plans
- Preparing handoff for new pages, flows, or component libraries
- Aligning designers and engineers on tokens, states, spacing, and responsiveness
- Reducing ambiguity around edge states, motion, and accessibility behavior
- Reviewing implemented UI against the original design intent
- Feeding implementation learnings back into the design system

## Workflow

1. Start with intent, not pixels.
   Clarify the user goal, primary interaction, failure modes, and success state before discussing exact implementation details.
2. Normalize the design into tokens and primitives.
   Convert ad hoc colors, spacing, radius, typography, and elevation choices into system tokens or explicit exceptions.
3. Define component contracts.
   Identify props, variants, responsive behavior, loading/empty/error states, and accessibility semantics before implementation begins.
4. Write the handoff for edge cases.
   Include keyboard behavior, validation rules, truncation, empty states, overflow, motion reductions, and focus handling.
5. Review implementation against intent.
   Compare shipped UI to state coverage, token usage, spacing rhythm, and interaction behavior rather than only static screenshots.
6. Feed lessons back into the system.
   If a pattern repeats, promote it into the design system instead of documenting the same exception repeatedly.

## Handoff Checklist

| Area | Must Be Explicit |
|---|---|
| Tokens | Colors, spacing, typography, radius, elevation, motion |
| States | Default, hover, focus, active, disabled, loading, empty, error |
| Responsiveness | Breakpoints, wrapping, truncation, reflow, minimum sizes |
| Accessibility | Roles, labels, contrast, focus order, keyboard behavior |
| Content rules | Max lengths, placeholders, validation, localization expansion |

## Bridge Rules

- Prefer reusable component variants over one-off pixel-perfect exceptions.
- Handoff should answer "what happens when this breaks or changes?" not only "what does the happy path look like?"
- Use acceptance criteria that engineering can verify in code review and QA.
- Capture where design intentionally deviates from the current system so it can be reviewed, not silently copied.
- Treat responsive behavior and accessibility as part of the design, not implementation detail.

## Output Format

When using this skill, produce:

1. **Intent summary** - user goal, flow context, and success criteria
2. **Component contract** - variants, props, states, and data dependencies
3. **Design tokens** - canonical token mapping and explicit exceptions
4. **Implementation notes** - responsive, accessibility, motion, and edge-case behavior
5. **Acceptance criteria** - concrete checks for design review, QA, and engineering signoff
