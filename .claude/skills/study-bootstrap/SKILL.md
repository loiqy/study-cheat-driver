---
name: study-bootstrap
description: initialize and maintain a structured study workspace for long-horizon technical learning in claude code. use when the current directory should become a persistent learning repository with notes, comparisons, stage tracking, roadmap files, resume files, and durable markdown outputs that help future claude sessions quickly continue from the current state.
---

# Study Bootstrap

Turn the current directory into a durable study workspace.

Treat the workspace as a long-lived learning repository, not as an application repo.

Prefer creating or updating files over keeping important state only in conversation.

## Goals

Establish a stable repository structure that supports:

- staged learning
- durable note-taking
- project comparison
- progress tracking
- fast resume in a new context

## Default repository structure

If missing, create these top-level files:

- `README.md`
- `study-manifest.md`
- `current-status.md`
- `learning-roadmap.md`

If missing, create these directories:

- `notes/`
- `comparisons/`
- `stages/`
- `progress/`
- `plans/`
- `summaries/`
- `resume/`
- `references/`
- `lectures/`

Inside `notes/`, create:

- `notes/projects/`
- `notes/modules/`
- `notes/concepts/`
- `notes/questions/`

Inside `comparisons/`, create:

- `comparisons/architecture/`
- `comparisons/modules/`
- `comparisons/patterns/`
- `comparisons/reuse/`

## Operating rules

- Do not overwrite existing high-value files unless the user explicitly asks.
- If a file exists, update it in place only when that is clearly better than creating a companion file.
- Prefer minimal, durable scaffolding over generating excessive empty documents.
- Keep filenames stable and descriptive so later sessions can rely on them.
- If the repository already has a structure, reconcile with it instead of forcing a full reset.

## Required initialization outputs

### `README.md`

Write a short explanation of what this study repository is for.

Include:

- current study theme
- target projects or materials
- how the repository is organized
- where a new session should start reading

### `study-manifest.md`

This is the main operating contract for the study workspace.

Include:

- study topic
- target materials or projects
- learning goals
- expected deliverables
- stage model
- canonical file locations
- rules for facts vs inferences vs recommendations
- rule that important state must be written to files

### `current-status.md`

This is the primary status snapshot.

Include:

- current stage
- most recent completed work
- key current conclusions
- open questions
- exact recommended next steps
- priority files to read next

### `learning-roadmap.md`

Write a multi-stage plan.

Include:

- major learning objectives
- recommended sequence
- what “done” means at each phase
- risks and likely blockers

## Stage scaffolding

If missing, create initial stage definition files:

- `stages/stage-0-overview.md`
- `stages/stage-1-project-mapping.md`
- `stages/stage-2-core-paths.md`
- `stages/stage-3-cross-project-comparison.md`
- `stages/stage-4-abstractions-and-reuse.md`
- `stages/stage-5-teaching-and-review.md`

Each stage file should include:

- objective
- entry criteria
- completion criteria
- expected outputs
- recommended next actions

## Resume scaffolding

If missing, create:

- `resume/resume-brief.md`
- `resume/resume-deep.md`

Use these rules:

- `resume-brief.md` should be readable in 1 to 2 minutes
- `resume-deep.md` should support a careful handoff to a new session

## Templates for initial content

When bootstrapping from scratch, initialize files with concise placeholders and explicit headings rather than long filler text.

Use headings such as:

- Scope
- Current Focus
- Known Facts
- Open Questions
- Next Steps
- Canonical Files

## Canonical truth rules

Treat these as high-priority state files:

1. `study-manifest.md`
2. `current-status.md`
3. `resume/resume-brief.md`

When these files conflict with older notes, prefer the newer explicit status documents unless there is clear evidence they are outdated.

## Avoid

- creating dozens of empty files with no purpose
- overwriting user-authored notes without permission
- leaving the workspace without a clear starting point for the next session
- keeping important stage or resume state only in chat

## Examples

- “Initialize this folder as a study workspace for three operating systems projects.”
- “Set up a reusable learning repo for comparing several codebases.”
- “Repair this study repo and create missing roadmap and resume files.”
- “Turn this directory into a persistent learning workspace with stage tracking.”