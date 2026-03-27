---
name: study-notes
description: generate and maintain structured learning notes for technical materials, codebases, modules, concepts, and questions in a persistent study workspace. use when the user wants teaching-oriented notes, project understanding, module breakdowns, concept explanations, reading guides, question lists, or durable markdown study artifacts rather than only conversational answers.
---

# Study Notes

Create durable, teaching-oriented study notes.

Prefer writing clear markdown files into the study workspace over keeping learning progress only in chat.

## Purpose

Help the user learn technical material systematically by producing notes that are:

- evidence-based
- easy to resume later
- organized for long-term accumulation
- suitable for self-teaching and later review

## Primary output locations

Choose the most appropriate destination:

- `notes/projects/` for per-project or per-repository notes
- `notes/modules/` for module-level or subsystem notes
- `notes/concepts/` for conceptual explanations and topic primers
- `notes/questions/` for unresolved questions, hypotheses, and follow-up items
- `lectures/` for tutorial-style teaching notes when the user wants structured lessons

## Core note types

### Project note

Use for a whole repository, service, codebase, paper implementation, or technical system.

Include:

- purpose
- scope
- primary language and framework signals
- top-level structure
- entry points
- important modules
- dependencies and build signals
- likely execution flow
- notable conventions
- current understanding
- open questions
- recommended next files to inspect

### Module note

Use for a specific subsystem, path, service component, algorithm, or driver path.

Include:

- what this module appears to do
- main inputs and outputs
- important files and symbols
- control flow
- data flow
- lifecycle or state transitions if relevant
- dependencies
- risks or confusing areas
- follow-up questions

### Concept note

Use for concepts the user is learning.

Include:

- plain-language explanation
- why it matters in this codebase or topic
- related concrete examples
- common confusion points
- how to recognize it in real code
- what to study next

### Question note

Use when ambiguity remains.

Include:

- exact question
- why it matters
- current evidence
- likely hypotheses
- what evidence would resolve it
- recommended next inspection steps

## Writing rules

- Write for future learning, not just for the current moment.
- Prefer explanation over raw dumping.
- Separate observed facts from interpretation.
- Keep terminology precise.
- Use examples from the studied material when available.
- When dealing with code, anchor explanations in concrete files, functions, configs, or interfaces.
- Use tables only when they improve understanding materially.

## Required section split

Whenever uncertainty exists, distinguish these headings explicitly:

- Facts
- Inferences
- Open Questions
- Next Steps

## File naming guidance

Prefer stable, descriptive names.

Examples:

- `notes/projects/project-a.md`
- `notes/modules/request-dispatch-path.md`
- `notes/concepts/kernel-user-boundary.md`
- `notes/questions/unresolved-init-order.md`

Avoid generic names like:

- `notes1.md`
- `random-thoughts.md`
- `misc.md`

## Teaching mode

When the user wants notes that feel like guided teaching material:

- define terms before using them heavily
- explain why each subsystem matters
- point out likely stumbling points
- propose a learning order
- include a short recap at the end

For especially instructional outputs, consider placing them under `lectures/`.

## Relationship to status files

After substantial note creation, check whether `current-status.md` or `resume/resume-brief.md` should be updated to reflect important progress.

Do this especially when the new note changes the current learning boundary.

## Avoid

- producing only chat answers when durable notes are clearly needed
- copying a file tree without interpretation
- mixing speculation into fact sections
- writing notes that cannot be resumed by a future session
- duplicating the same explanation across many files without need

## Examples

- “Write a study note for each of these three projects.”
- “Explain this subsystem like course notes, not like a code review.”
- “Create a concept note for interrupt handling and how it shows up here.”
- “Turn my current understanding into structured notes I can continue later.”
- “Write a question log for the things we still do not understand.”