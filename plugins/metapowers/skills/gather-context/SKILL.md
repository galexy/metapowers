---
name: gather-context
description: Gather context for a beads issue by exploring the codebase, reading referenced docs, and summarizing findings. Use when starting work on a story, feature, or task and you need to understand the relevant code, architecture, and acceptance criteria before planning.
argument-hint: [beads-issue-id]
context: fork
agent: Explore
---

# Gather Context

Gather full context for beads issue `$ARGUMENTS` by exploring the codebase and referenced documents, then store a structured context summary in the issue notes.

```mermaid
flowchart TB
  bd_show["bd show issue"]
  claimable{"Claimable?"}
  error["Error: not claimable"]
  has_context{"Has task_context?"}
  explore["Explore codebase\n& referenced docs"]
  store["Store brief\nin issue notes"]
  claim["Claim & Report"]

  bd_show --> claimable
  claimable -- "no" --> error
  claimable -- "yes" --> has_context
  has_context -- "yes" --> claim
  has_context -- "no" --> explore
  explore --> store
  store --> claim
```

---

## Requirements

Run `bd show $ARGUMENTS` and verify the following before proceeding. If any requirement is not met, report the corresponding error (see Common Issues) and stop.

- **Claimable status** — The issue's status must be claimable (e.g., `open`, `backlog`, or `todo`). If the issue is already `in_progress`, `done`, `closed`, or otherwise not claimable, report an error.

---

## Step 1: Check for Existing Context

Inspect the issue's notes (from the `bd show` output above) for an existing `<task_context>` block. If present, go to **Step 4: Claim and Report** and report:

> Context already gathered for issue `$ARGUMENTS`. The task context is available in the issue notes.

Do not re-gather or output the brief.

## Step 2: Explore Context

Read and investigate the codebase directly (you are already running in a forked Explore context):

1. **Referenced documents** — Read any architecture docs, specs, PRDs, or parent issues linked from the issue.
2. **Relevant code areas** — Identify and read the key source files, modules, types, and interfaces that the issue touches or depends on.
3. **Dependencies** — Identify internal modules and external libraries involved.
4. **Existing tests** — Find and read test files related to the affected code areas to understand current coverage and test style.

## Step 3: Store Context in Issue

Synthesize your findings into a context brief and append it to the issue's notes wrapped in a `<task_context>` tag:

```bash
bd update $ARGUMENTS --notes "<task_context>
## Issue summary
<What needs to be done and why>

## PRD or Parent Issue context

<The relevant section(s) from the PRD that this issue implements or context from the parent issue>

## Affected code areas

<Key files and modules with brief descriptions of their role>

## Relevant types and interfaces

<Important type definitions and contracts>

## Dependencies

<Internal and external dependencies involved>

## Test landscape

<Existing test coverage and test style in affected areas>

## Open questions

<Anything unclear or needing user input>
</task_context>"
```

Fill in each section with the actual findings — do not use the template text above verbatim.

## Step 4: Claim and Report

```bash
bd update $ARGUMENTS --status=in_progress
```

Report to the caller:

> Context gathered for issue `$ARGUMENTS`. The task context is available in the issue notes.

Do not output the context brief itself — the calling agent should read it from the issue when needed.

---

## Common Issues

- **Issue not claimable** — If the issue status is `in_progress`, `done`, `closed`, or any other non-claimable state, report:

  > Error: Issue `$ARGUMENTS` has status `<status>` and cannot be claimed. Only issues with a claimable status (e.g., `open`, `backlog`, `todo`) can be gathered.

  Do not proceed with context gathering.
