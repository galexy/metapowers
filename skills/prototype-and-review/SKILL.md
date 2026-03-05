---
name: prototype-and-review
description: Implement a task by coordinating an agent team with parallel prototypes, multi-area code review, fix cycles, PR creation, and comparative evaluation. Use when the user wants to explore multiple implementation approaches for a spike, prototype, or feature.
argument-hint: [beads-issue-id]
---

# Prototype and Review Workflow

Implement the beads issue `$ARGUMENTS` by coordinating an agent team. Create parallel prototypes, review them, fix issues, create PRs, and evaluate.

## Phase 0: Context Gathering

1. Run `bd show $ARGUMENTS` to understand the task, its dependencies, and deliverables.
2. Read any referenced architecture docs, specs, or parent issues.
3. Identify the distinct prototype approaches to explore (minimum 2). Look for:
   - The recommended approach from the architecture/design docs
   - The fallback or alternative approach
   - Any other viable approaches worth validating
4. Claim the issue: `bd update $ARGUMENTS --status=in_progress`

## Phase 1: Task Breakdown

1. Create a beads subtask for each prototype:
   ```bash
   bd create --title="Prototype X: <approach>" --description="<details>" --type=task --priority=1
   bd dep add <subtask-id> $ARGUMENTS --type=parent-child
   ```

2. For each prototype, create child subtasks for the implementation steps. Link them with parent-child and blocking dependencies. The agent will update these as it progresses.

3. Create an evaluation task blocked by all prototype tasks:
   ```bash
   bd create --title="Evaluate prototypes and recommend direction" --description="<criteria>" --type=task --priority=1
   bd dep add <eval-id> $ARGUMENTS --type=parent-child
   bd dep add <eval-id> <prototype-1-id> --type=blocks
   bd dep add <eval-id> <prototype-2-id> --type=blocks
   ```

## Phase 2: Parallel Implementation

1. **Pre-create git worktrees** for each prototype (the `isolation: "worktree"` parameter may not work reliably in all environments):
   ```bash
   git worktree add /tmp/<prototype-name> -b <branch-name>
   cd /tmp/<prototype-name> && pnpm install
   ```

2. **Create a team** via TeamCreate.

3. **Launch implementation agents** in parallel — one per prototype. Use `subagent_type: "general-purpose"` with `mode: "bypassPermissions"` and `run_in_background: true`. Do NOT use `isolation: "worktree"`.

   Each agent's prompt MUST include:
   - `cd /tmp/<worktree-path>` as the FIRST instruction
   - The branch name it's on
   - The beads task IDs to claim and close
   - Full technical context (dependencies, config, architecture decisions)
   - Specific deliverable requirements
   - Commands to verify work (tsc, test runner)
   - Commit instructions

4. **Verify worktree isolation** after launch — confirm `git worktree list` shows separate entries and `git branch --show-current` on main hasn't changed.

5. Wait for all implementation agents to complete.

## Phase 3: Code Review

Launch review agents in parallel for each prototype. Use `subagent_type: "feature-dev:code-reviewer"` with `run_in_background: true`.

Group the 7 review areas into 3 focused agents per prototype:

### Review Agent 1: Style, Formatting, and Cruft (Areas 1, 5, 6)

**Area 1 — Coding Style:**
- Consistent naming (camelCase for vars/functions, PascalCase for types/interfaces)
- Idiomatic TypeScript (proper types, no unnecessary `any`, good generics use)
- Clean imports (no unused, consistent ordering)
- Function size and decomposition
- Readability

**Area 5 — Debugging Cruft:**
- Remove stray `console.log` not part of intentional output
- Remove commented-out code
- Remove temporary files, scratch code, TODO comments that should be issues
- Check for leftover debugging flags

**Area 6 — Consistent Formatting:**
- Indentation, semicolons, quotes, trailing commas, brace style, line length

### Review Agent 2: Efficiency and Error Handling (Areas 2, 4)

**Area 2 — Efficiency and Algorithm Appropriateness:**
- No Rube Goldberg solutions
- Appropriate data structures
- No unnecessary allocations, redundant iterations, or O(n^2) where O(n) is possible
- No shortcuts that sacrifice correctness

**Area 4 — Robust Error Handling:**
- Network failures, malformed input, empty input
- Null/undefined guards
- Consistent error handling pattern
- No unhandled promise rejections
- Edge cases considered

### Review Agent 3: Types/Lint and Test Coverage (Areas 3, 7)

**Area 3 — Error/Warning Cleanup:**
- Run `tsc --noEmit` and report errors
- Run the test suite and report failures
- Check for `@ts-ignore`, `@ts-expect-error`, `as any`

**Area 7 — Functional Test Coverage:**
- Boundary conditions (empty, single, very large)
- Error conditions
- 0, 1, and many instances
- Complex feature interactions
- Critical success criteria explicitly tested (black box)
- Consider property-based tests where they add clarity
- All tests passing

### Review Output Format

Each reviewer rates issues as:
- **MUST FIX**: Genuinely problematic (not a nitpick)
- **NITPICK**: Minor style preference

Only MUST FIX items are reported (with file:line citations).

## Phase 4: Fix Cycle

1. **Collate findings** from all reviewers, deduplicating overlapping issues.
2. **Create a beads fix task** per prototype with the full list of MUST FIX items.
3. **Dispatch fixes** to the implementation agents via SendMessage with specific instructions for each fix.
4. Wait for fixes to complete.
5. **Run verification review** (round 2) — quick pass to confirm all fixes applied, tests pass, tsc clean.
6. **Repeat** if round 2 finds new issues. Stop when reviews are clean.

## Phase 5: PR Creation

For each prototype:
1. Push the branch: `cd /tmp/<worktree> && git push -u origin <branch>`
2. Create a PR with `gh pr create` including:
   - Summary of what was built and key findings
   - Test plan (test count, tsc status, review status)
3. Create a beads PR task if gating on user approval.

## Phase 6: Comparative Evaluation

1. Claim the evaluation beads task.
2. Launch 3 evaluation agents in parallel (`subagent_type: "feature-dev:code-explorer"`), each analyzing from a different perspective:

   - **Performance & Scalability**: Parse time, memory, scaling behavior, budget compliance
   - **Correctness & Reliability**: Accuracy, edge cases, failure modes, authoritative vs reconstructed data
   - **Maintainability & Architecture Fit**: Code complexity, LOC, heuristic count, extensibility, dependency risk, test confidence

3. Collate evaluations into a recommendation with justification.
4. Post evaluation summaries as comments on the respective PRs.

## Phase 7: Resolution

1. Based on evaluation, recommend which prototype to merge.
2. Close the rejected PR(s) with evaluation summary.
3. **Do NOT close the parent beads issue** — gate on user approval.
4. Update beads: close subtasks, close evaluation task.

## Phase 8: Cleanup

1. Shut down all agents.
2. Delete the team via TeamDelete.
3. Remove worktrees: `git worktree remove /tmp/<name>`
4. Push final beads state to main.
5. Verify: `git status` shows clean, up to date with origin.

## Important Rules

- Always use beads (`bd`) for task tracking — never markdown TODOs or TaskCreate
- Always pre-create worktrees manually — don't rely on `isolation: "worktree"`
- Break implementation tasks into beads subtasks BEFORE delegating to agents
- Shut down agents promptly when their work is done
- Fix stale `isActive` flags in team config if TeamDelete fails
- Never close the parent issue without user approval
