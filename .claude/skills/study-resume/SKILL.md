---
name: study-resume
description: create and maintain high-density resume files for a persistent technical study workspace so new claude code sessions can quickly continue from the current boundary. use when the user wants a fast handoff, session continuation, compact project status, restart context, explicit next steps, or durable memory files that prevent repeated rediscovery work.
---

# Study Resume

Create durable handoff files so a new Claude session can resume the study efficiently.

This skill is for continuity, not general summarization.

## Purpose

Reduce repeated rediscovery.

A strong resume file should let a new session understand:

- what is being studied
- why it matters
- what has already been established
- what remains uncertain
- where to continue next
- which files are canonical

## Primary outputs

Maintain two core files:

- `resume/resume-brief.md`
- `resume/resume-deep.md`

Also update `current-status.md` when necessary.

## Resume file roles

### `resume/resume-brief.md`

This is the first file a new session should read.

Keep it compact and high signal.

Target contents:

- topic and scope
- target projects or materials
- current stage
- most important confirmed findings
- most important open questions
- exact next actions
- priority files to read next
- files that are canonical sources of truth

### `resume/resume-deep.md`

This is the detailed handoff.

Include:

- current study background
- major artifacts produced so far
- summary of per-project understanding
- summary of cross-project conclusions
- unresolved risks or ambiguities
- recommended continuation plan
- known dead ends or work that should not be repeated

## Resume update triggers

Update resume files when:

- a stage changes
- a major comparison changes current understanding
- a new canonical note is created
- a previous open question is resolved
- the recommended next steps change materially
- the user explicitly asks for continuity or handoff

## Writing rules

- prefer precision over completeness
- be terse but not cryptic
- name files explicitly
- identify what is confirmed versus tentative
- include instruction-like next steps
- make the resume useful to a future session that has no chat history

## Required headings for `resume-brief.md`

Use these headings unless the workspace has a better established pattern:

- Scope
- Current Stage
- Confirmed Findings
- Open Questions
- Immediate Next Actions
- Read These First
- Canonical Files

## Recommended headings for `resume-deep.md`

- Background
- Study Targets
- Current Understanding
- Comparison Status
- Important Artifacts
- Known Uncertainties
- Recommended Continuation Plan
- Do Not Repeat

## Canonical file policy

When in doubt, point the next session to these before secondary notes:

1. `study-manifest.md`
2. `current-status.md`
3. `resume/resume-brief.md`
4. the most recent relevant note or comparison file

## Strong handoff behavior

A good resume file should include enough structure that the next session can begin work without first asking what happened before.

Prefer explicit statements like:

- “Read `notes/projects/project-a.md` and `comparisons/modules/init-paths.md` first.”
- “Do not re-map the repository structure; that work is already stable.”
- “The next unresolved issue is whether the cleanup path differs between project B and project C.”

## Avoid

- generic summaries with no actionable next step
- repeating large sections of existing notes
- hiding critical uncertainty
- omitting canonical file references
- writing only for the current session instead of the next one

## Examples

- “Create a resume file so the next Claude session can continue.”
- “Summarize exactly where we are and what to do next.”
- “Write a compact handoff for this study repo.”
- “Update the resume after today’s progress.”
- “Produce a restart context that prevents repeated analysis.”