---
name: study-compare
description: compare multiple related technical projects, systems, modules, or implementations and write evidence-based cross-project analysis into a study workspace. use when the user wants side-by-side architecture review, implementation comparison, pattern extraction, reuse analysis, tradeoff discussion, or durable comparison reports across two to five related projects.
---

# Study Compare

Produce structured, evidence-based comparison outputs across multiple related projects or systems.

This skill is for durable analysis, not just ad hoc discussion.

## Purpose

Help the user learn by comparing.

Comparison should reveal:

- what is shared
- what differs
- why the differences may exist
- what abstractions or reusable ideas emerge
- which approach is stronger for a given goal

## Primary output locations

Choose the most appropriate destination:

- `comparisons/architecture/`
- `comparisons/modules/`
- `comparisons/patterns/`
- `comparisons/reuse/`
- `summaries/` for higher-level conclusions

## Comparison procedure

### 1. Establish scope

Before comparing, define:

- which items are being compared
- which dimension is in focus
- what question the comparison is trying to answer

If no explicit dimension is given, default to:

- purpose
- tech stack
- structure
- architectural style
- core paths
- module boundaries
- error handling
- testing/build signals
- extensibility
- reuse opportunities

### 2. Build individual understanding first

Do not make broad cross-project claims until each project or target has at least a minimal standalone understanding.

If per-project notes are missing, create or update them first.

### 3. Compare with evidence

Support important claims with concrete observations such as:

- key files
- major directories
- entry points
- config or build definitions
- interfaces
- execution paths
- module boundaries
- repeated patterns

### 4. Separate the levels of judgment

Use these headings when relevant:

- Shared Patterns
- Meaningful Differences
- Likely Reasons for Divergence
- Tradeoffs
- Reuse Opportunities
- Recommendations
- Open Questions

## Comparison output types

### Architecture comparison

Use for overall design and layering.

Include:

- major layers
- coupling patterns
- boundaries
- lifecycle or startup shape
- key extension points
- likely maintenance implications

### Module comparison

Use for specific subsystems or paths.

Include:

- side-by-side responsibilities
- control-flow differences
- data-flow differences
- error-path differences
- concurrency or lifecycle differences if relevant

### Pattern comparison

Use for recurring implementation patterns.

Include:

- common abstractions
- duplicated logic
- places where the same concept appears under different structure
- candidate canonical pattern

### Reuse analysis

Use when the user wants consolidation or shared-library insight.

Include:

- code or idea that could be abstracted
- barriers to reuse
- what should stay separate
- sequencing for consolidation

## Output style

- Be crisp and evidence-driven.
- Prefer side-by-side structure when useful.
- Use tables sparingly and only when they clarify.
- Avoid vague summary language such as “these are similar” without specifics.
- State when a claim is only tentative.

## Required caution

Do not equate surface similarity with true architectural similarity.

Pay special attention to:

- different responsibilities hidden behind similar filenames
- different lifecycle models under similar APIs
- historical artifacts
- environment-specific constraints

## Relationship to other study files

After meaningful comparison work:

- update `current-status.md` if the comparison changes current understanding
- update `resume/resume-brief.md` if the comparison creates a new recommended next step
- create or update `summaries/` if a stable cross-project conclusion has emerged

## Suggested file naming

Examples:

- `comparisons/architecture/overall-architecture.md`
- `comparisons/modules/init-paths.md`
- `comparisons/patterns/resource-lifecycle.md`
- `comparisons/reuse/shared-abstractions.md`

## Avoid

- comparing before understanding each target
- presenting preference judgments without criteria
- writing only prose without structure
- losing evidence trails
- mixing architecture comparison with tutorial notes unless the user wants both

## Examples

- “Compare these three codebases and explain their architectural differences.”
- “Which project has the cleanest initialization path and why?”
- “Find repeated implementation ideas that could become shared abstractions.”
- “Write a durable comparison report I can revisit later.”
- “Compare the module boundaries and control paths across these systems.”