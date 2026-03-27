---
name: study-stage-manager
description: manage staged technical learning in a persistent study workspace by defining stages, advancing progress, recording completion criteria, and updating next actions. use when the user wants learning to proceed step by step with explicit stage boundaries, progress logs, completion signals, and durable state files that keep future claude sessions aligned.
---

# Study Stage Manager

Run the study workspace as a staged learning program.

Use explicit stages, durable progress updates, and concrete completion criteria.

## Purpose

Prevent learning from becoming an unstructured accumulation of notes.

This skill exists to answer:

- where are we now
- what is this stage for
- what is already done
- what remains unfinished
- what should happen next

## Canonical state files

Treat these as the main stage-control files:

- `current-status.md`
- `learning-roadmap.md`
- `stages/`
- `progress/`
- `plans/`
- `resume/resume-brief.md`

## Default stage model

If no stage model exists, use:

- Stage 0: Overview and map-building
- Stage 1: Per-project structure understanding
- Stage 2: Core execution paths and modules
- Stage 3: Cross-project comparison
- Stage 4: Abstractions, reuse, and design judgment
- Stage 5: Teaching outputs and review consolidation

Adapt the labels when the subject matter requires it, but keep the same logical progression unless there is a strong reason to change it.

## Stage file expectations

Each `stages/stage-*.md` file should define:

- objective
- entry criteria
- completion criteria
- required outputs
- common failure modes
- recommended next actions

## Progress update procedure

When the user completes meaningful work or asks for planning:

1. Identify the current stage.
2. Check whether stage goals were advanced materially.
3. Update `progress/` with concrete evidence of progress.
4. Update `current-status.md`.
5. If stage completion criteria are met, mark the stage complete and recommend the next stage.
6. Update `resume/resume-brief.md` if a future session should resume from a new boundary.

## Required status structure

When updating `current-status.md`, include:

- Current Stage
- Stage Goal
- Recently Completed
- Current Findings
- Unresolved Questions
- Immediate Next Actions
- Priority Files or Notes to Read

## Completion rules

Do not mark a stage complete only because many notes exist.

A stage should be treated as complete only when its completion criteria are explicitly satisfied.

Examples:

- Stage 1 is not complete until each target project has a usable project note.
- Stage 3 is not complete until meaningful cross-project comparison artifacts exist.
- Stage 5 is not complete until teaching or summary outputs are durable and resumable.

## Planning rules

When generating next steps:

- prefer concrete actions over vague suggestions
- name files, modules, or topics explicitly
- sequence actions by dependency
- distinguish urgent blockers from optional exploration

## Progress file guidance

Prefer one of these patterns:

- `progress/stage-1-log.md`
- `progress/stage-2-findings.md`
- `progress/2026-03-28-progress.md`

Use whichever is more stable for the workspace, but remain consistent once a pattern is established.

## Relationship to learning notes

Do not duplicate long note content in stage files.

Stage files should answer:

- what this stage is for
- what counts as done
- where evidence lives

Progress files should answer:

- what we actually did
- what changed
- what should happen next

## Avoid

- vague stage names with no criteria
- marking progress without durable artifacts
- keeping the actual plan only in chat
- confusing exploration volume with stage completion
- resetting the roadmap too often

## Examples

- “What stage are we in, and what should we do next?”
- “Advance the study plan after today’s analysis.”
- “Update current status and mark whether stage 2 is complete.”
- “Turn this loose learning effort into explicit phases.”
- “Write concrete completion criteria for the next stage.”