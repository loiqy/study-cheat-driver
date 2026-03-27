---
name: systems-code-lens
description: analyze low-level and systems-oriented codebases with attention to lifecycle, boundaries, control flow, resource ownership, concurrency, error handling, configuration, and operating environment constraints. use when studying c, c++, rust, kernel-adjacent code, drivers, operating-system components, backend infrastructure with strong lifecycle concerns, or other low-level technical projects where generic application analysis would miss critical systems behavior.
---

# Systems Code Lens

Apply a systems-oriented reading strategy to code and technical artifacts.

This skill adjusts how the workspace is analyzed. It does not replace note-taking, comparison, stage tracking, or resume management.

Use it to improve the quality of learning artifacts for low-level and systems code.

## Purpose

When the studied material is low-level, lifecycle-driven, environment-constrained, or close to hardware or the operating system, generic software analysis is often insufficient.

This skill exists to make Claude pay attention to:

- initialization and teardown
- resource ownership and cleanup
- control-flow shape
- state transitions
- user/kernel or process/environment boundaries
- concurrency and synchronization
- failure handling
- build and configuration switches
- environment-specific constraints

## Core reading priorities

When reading a project, prioritize these questions.

### 1. What is the execution environment?

Establish:

- user mode or kernel mode
- operating system and platform assumptions
- runtime constraints
- privilege boundaries
- external interfaces to the environment

Look for evidence in:

- build files
- project files
- target definitions
- compiler flags
- platform macros
- include paths
- deployment or installation artifacts
- entry-point signatures
- manifest or service definitions

### 2. What are the true entry points?

Do not assume the top-level `main` equivalent tells the full story.

Identify actual entry paths such as:

- driver entry or initialization routines
- registration callbacks
- dispatch tables
- exported interfaces
- interrupt or request handlers
- service startup hooks
- plugin registration points
- thread or worker creation paths

If there are multiple entry paths, map them explicitly.

### 3. How does lifecycle work?

Track lifecycle carefully.

Identify:

- startup
- steady-state handling
- reconfiguration
- shutdown
- cleanup after failure
- unload or destruction
- restart behavior

Pay close attention to asymmetry between initialization and cleanup.

When possible, write lifecycle notes as ordered steps rather than vague prose.

### 4. What resources exist, and who owns them?

Track ownership and lifetime for:

- memory
- handles
- file or device objects
- buffers
- locks
- threads
- timers
- queues
- context structures
- mapped regions
- registration objects

Ask:

- where is the resource created
- who owns it
- who may borrow it
- where is it released
- what happens on partial failure
- what happens on repeated initialization or cancellation

### 5. What are the important boundaries?

Look for boundaries such as:

- user mode vs kernel mode
- client vs service
- caller vs callee ownership
- public API vs private implementation
- synchronous vs asynchronous execution
- interrupt/deferred/worker contexts
- trusted vs untrusted input
- compile-time vs runtime configuration

Boundary clarity often explains the real architecture better than directory layout alone.

### 6. How are requests or work items processed?

For request-driven systems, reconstruct the handling path.

Track:

- source of request
- validation
- dispatch
- state/context lookup
- work scheduling
- processing
- completion
- cleanup
- error propagation

If the project is a driver or handler-based system, explicitly map dispatch logic and completion behavior.

### 7. What concurrency model exists?

Look for:

- threads and worker pools
- callbacks and asynchronous completion
- locks and lock ordering
- atomics
- interrupt vs deferred execution
- producer/consumer queues
- reentrancy concerns
- cancellation and teardown races

Do not treat concurrency as an afterthought.

If concurrency is present, note both the mechanism and the likely correctness risks.

### 8. How does error handling really behave?

Systems code often hides complexity in failure paths.

Inspect:

- status codes and translation layers
- goto-based cleanup chains
- partial initialization rollback
- retry logic
- timeout handling
- assertions and bug checks
- fallback behavior
- silent failure or logging-only failure

Prefer mapping the cleanup path explicitly when it is nontrivial.

### 9. What build or configuration switches matter?

Low-level projects often behave differently depending on:

- preprocessor macros
- target platform
- debug/release mode
- optional features
- driver model version
- compiler or linker settings
- conditional source inclusion

Record meaningful build-time branches when they affect architecture or behavior.

## Domain-specific guidance for drivers and kernel-adjacent code

When the project appears to be a driver or otherwise kernel-adjacent, pay extra attention to:

- entry and unload flow
- device/object creation and destruction
- dispatch tables and request handling
- IRP or request lifecycle if applicable
- PnP or power-related paths if applicable
- synchronization constraints
- pageable vs nonpageable assumptions if visible
- I/O model and buffering assumptions
- interaction between framework code and project code

Do not force these concepts if the project does not use them. Apply only when supported by evidence.

## Preferred outputs

When helping a study workspace, enrich files under:

- `notes/projects/`
- `notes/modules/`
- `notes/concepts/`
- `comparisons/architecture/`
- `comparisons/modules/`
- `comparisons/patterns/`

Useful note types include:

- lifecycle maps
- request path maps
- resource ownership notes
- boundary analysis
- concurrency risk notes
- build/configuration impact notes

## Output structure suggestions

When relevant, use headings like:

- Execution Environment
- Entry Points
- Lifecycle
- Resource Ownership
- Boundaries
- Request Path
- Concurrency Model
- Error Handling
- Build and Configuration Signals
- Open Risks
- Open Questions

## Interpretation rules

- Prefer concrete control-flow and ownership facts over superficial naming.
- Treat cleanup and error paths as first-class architecture.
- Distinguish “confirmed by code” from “likely based on pattern”.
- If code appears framework-driven, separate framework responsibilities from project-specific responsibilities.
- If a project has multiple operational contexts, describe them separately.

## Relationship to other skills

This skill should strengthen, not replace, other study skills:

- use `study-notes` for durable learning notes
- use `study-compare` for cross-project analysis
- use `study-stage-manager` for stage progression
- use `study-resume` for handoff files

## Avoid

- treating low-level projects like ordinary CRUD apps
- ignoring teardown and cleanup paths
- ignoring concurrency and environment constraints
- assuming directory structure alone explains behavior
- making driver-specific claims without code evidence
- locking this skill to one platform or one subdomain

## Examples

- “Study this c++ driver project and explain the lifecycle.”
- “Map the request handling path in this systems component.”
- “Compare the resource ownership model across these low-level projects.”
- “Write notes on concurrency and cleanup risks in this codebase.”
- “Explain this kernel-adjacent code in a way that helps long-term learning.”