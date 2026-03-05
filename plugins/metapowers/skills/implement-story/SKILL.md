---
name: implement-story
description: Implement a user story or feature by coordinating a coder and tester through design, implementation with TDD, multi-area code review, and PR creation. Use when the user wants to implement a story, feature, or task from beads.
argument-hint: [beads-issue-id]
---

# Implement Story Workflow

Implement the beads issue `$ARGUMENTS` by coordinating a coder and tester through design collaboration, TDD implementation, code review, and PR creation.

---

## Phase 0: Context Gathering

1. Run `bd show $ARGUMENTS` to understand the task, its dependencies, parent epic, and acceptance criteria.
2. Read any referenced architecture docs, specs, PRD, or parent issues.
3. Identify the relevant codebase areas — read key files to understand existing patterns, conventions, and types.
4. Claim the issue: `bd update $ARGUMENTS --status=in_progress`

## Phase 1: Design Collaboration

### Setup

1. Create a team via TeamCreate.

2. Create beads subtasks:
   ```bash
   bd create --title="Design: Implementation plan for <story>" --description="<details>" --type=task --priority=1
   bd create --title="Design: Test plan for <story>" --description="<details>" --type=task --priority=1
   bd create --title="Design: PR for review" --description="Gate design completion on PR approval" --type=task --priority=1
   ```
   Link as children of `$ARGUMENTS`. Make the PR task blocked by both design tasks.

3. Create a working branch and worktree:
   ```bash
   git worktree add /tmp/<story-branch> -b <story-branch>
   cd /tmp/<story-branch> && pnpm install
   ```

### Launch Design Agents

Launch two agents in parallel (`subagent_type: "general-purpose"`, `mode: "bypassPermissions"`, `run_in_background: true`). Both work in the same worktree since they're writing separate design docs (no conflict risk).

**Coder Agent** — writes the implementation design:

The coder's prompt MUST instruct them to:
- Work in `/tmp/<story-branch>`
- Claim their beads design task
- Read the architecture docs, existing code, and types relevant to the story
- Write a design document at a specified path (e.g., `.specs/<story>/implementation-design.md`) covering:
  - **Approach**: High-level strategy and rationale
  - **Files to create/modify**: With specific descriptions of changes
  - **Interfaces and types**: Key type definitions and contracts
  - **Data flow**: How data moves through the system
  - **Dependencies**: External libraries or internal modules needed
  - **Edge cases**: Known complications and how they'll be handled
  - **Open questions**: Anything needing clarification
- After writing, send the design to the tester agent for feedback via SendMessage
- Iterate based on tester feedback (at least 1 round)
- Commit the design doc when both agents agree it's ready

**Tester Agent** — writes the test plan:

The tester's prompt MUST instruct them to:
- Work in `/tmp/<story-branch>`
- Claim their beads design task
- Read the architecture docs, existing code, existing tests, and the story's acceptance criteria
- Write a test plan at a specified path (e.g., `.specs/<story>/test-plan.md`) covering:
  - **BDD Acceptance Criteria**: Expand the story's acceptance criteria into Given/When/Then scenarios. Cover happy paths, edge cases, and error conditions.
  - **Unit Tests**: Key units to test, with descriptions of what each test validates
  - **Integration Tests**: How components interact — test the boundaries between modules, services, and layers. Cover data flow across module boundaries.
  - **End-to-End Tests**: Tests that mimic actual user usage patterns. Describe realistic scenarios a user would perform, end to end through the public API or UI.
  - **Boundary Conditions**: 0, 1, many instances; empty inputs; maximum sizes
  - **Error Conditions**: Network failures, malformed input, missing data, timeouts
  - **Performance Criteria**: Any performance budgets from the architecture doc
  - **Test Data**: Fixtures, mocks, or factories needed
- After writing, send the test plan to the coder agent for feedback via SendMessage
- Iterate based on coder feedback (at least 1 round) — ensure the test plan aligns with the implementation design
- Commit the test plan when both agents agree it's ready

### Design Feedback Loop

The agents should exchange feedback via SendMessage:
1. Coder sends implementation design to tester
2. Tester sends test plan to coder
3. Each reviews the other's doc and sends feedback
4. Both iterate until aligned (minimum 1 feedback round)
5. Both commit their docs and close their design tasks

### Design PR

Once both design docs are committed:
1. Push the branch and create a PR with both design docs
2. PR body should summarize the design decisions and link to the beads issue
3. Close the PR beads task — it's now gated on user review

**PAUSE HERE.** Notify the user that the design PR is ready for review. Wait for:
- User approval (PR merged) → proceed to Phase 2
- User feedback (PR comments) → relay feedback to the coder and tester agents (re-launch if needed) for iteration, then update the PR

## Phase 2: TDD Implementation

### Setup

1. Create beads subtasks for implementation:
   ```bash
   bd create --title="Implement: <story>" --description="<details, reference design doc>" --type=task --priority=1
   bd create --title="Test: <story>" --description="<details, reference test plan>" --type=task --priority=1
   ```
   Link as children of `$ARGUMENTS`.

2. The implementation happens on the same branch as the design docs (or a new branch if preferred). Pre-create a worktree if the old one was cleaned up:
   ```bash
   git worktree add /tmp/<story-branch> -b <story-branch>
   ```

### Launch Implementation Agents

Launch two agents in parallel. Both work in the same worktree — coordinate via SendMessage to avoid file conflicts.

**Coder Agent** — implements with TDD discipline:

The coder's prompt MUST instruct them to:
- Work in `/tmp/<story-branch>`
- Read the implementation design doc and test plan first
- Claim their beads implementation task
- **Follow TDD — write tests BEFORE code, incrementally:**
  1. Pick the first unit of work from the design
  2. Write a failing test for it (red)
  3. Write the minimal code to make it pass (green)
  4. Refactor if needed
  5. Commit the test + code together
  6. Move to the next unit of work
  7. Repeat until the implementation is complete
- Do NOT write all tests at once then all code — interleave them
- Coordinate with the tester agent: notify when each increment is committed so the tester can verify and add integration/e2e tests on top
- Run tests after each increment to ensure nothing is broken
- Run `tsc --noEmit` periodically

**Tester Agent** — develops integration and e2e tests:

The tester's prompt MUST instruct them to:
- Work in `/tmp/<story-branch>`
- Read the test plan first
- Claim their beads test task
- Begin writing integration and e2e tests based on the test plan
- Coordinate with the coder: wait for notifications about completed increments before writing integration tests that depend on those modules
- Write tests incrementally as the coder completes units:
  1. As units become available, write integration tests for module boundaries
  2. Write e2e tests that exercise realistic user scenarios
  3. Verify BDD acceptance criteria are covered
  4. Run the full test suite after each addition
- Flag any implementation issues discovered during testing — send to coder via SendMessage
- Commit tests incrementally

### Implementation Coordination

The coder and tester coordinate via SendMessage:
1. Coder completes a unit → notifies tester with what's ready
2. Tester writes integration/e2e tests for that unit → notifies coder of any issues found
3. Coder fixes issues → notifies tester
4. Repeat until implementation is complete
5. Both close their beads tasks when done

## Phase 3: Code Review

Launch review agents in parallel for the implementation. Use `subagent_type: "feature-dev:code-reviewer"` with `run_in_background: true`.

Group the 7 review areas into 3 focused agents:

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
- Run the test suite and report failures — **execute the tests and include the output**
- Check for `@ts-ignore`, `@ts-expect-error`, `as any`

**Area 7 — Functional Test Coverage:**
- Boundary conditions (empty, single, very large)
- Error conditions
- 0, 1, and many instances
- Complex feature interactions
- Critical success criteria from acceptance criteria explicitly tested (black box)
- Integration tests cover module boundaries
- E2e tests mimic realistic user scenarios
- BDD acceptance criteria are all covered
- Consider property-based tests where they add clarity
- All tests passing

### Review Output Format

Each reviewer rates issues as:
- **MUST FIX**: Genuinely problematic (not a nitpick)
- **NITPICK**: Minor style preference

Only MUST FIX items are reported (with file:line citations).

**CRITICAL**: Review Agent 3 MUST execute the test suite and include the output. Test failures are MUST FIX items that get sent back to the implementation agents.

## Phase 4: Fix Cycle

1. **Collate findings** from all reviewers, deduplicating overlapping issues.
2. **Create a beads fix task** with the full list of MUST FIX items.
3. **Dispatch fixes** to the coder and tester agents (re-launch if needed) via SendMessage:
   - Code issues → coder
   - Test issues → tester
   - Both must run full test suite after fixes
4. Wait for fixes to complete.
5. **Run verification review** (round 2) — quick pass to confirm all fixes applied, tests pass, tsc clean. **Execute the tests again.**
6. **Repeat** if verification finds new issues. Stop when reviews are clean and all tests pass.

## Phase 5: Implementation PR

1. Push the branch: `cd /tmp/<worktree> && git push -u origin <branch>`
2. Create a PR with `gh pr create` including:
   - Summary of what was implemented
   - Reference to the design PR and beads issue
   - Test results (test count, pass rate)
   - Review status (rounds completed, all clean)
3. Notify the user the implementation PR is ready for review.

**PAUSE HERE.** Wait for user approval. Relay any PR feedback to agents for fixes.

## Phase 6: Resolution

Once the user approves and merges the PR:
1. Close all remaining beads subtasks.
2. **Do NOT close the parent beads issue** — gate on explicit user approval.

## Phase 7: Cleanup

1. Shut down all agents.
2. Delete the team via TeamDelete.
3. Remove worktrees: `git worktree remove /tmp/<name>`
4. Push final beads state to main.
5. Verify: `git status` shows clean, up to date with origin.

## Important Rules

- Always use beads (`bd`) for task tracking — never markdown TODOs or TaskCreate
- Always pre-create worktrees manually — don't rely on `isolation: "worktree"`
- Break tasks into beads subtasks BEFORE delegating to agents
- Shut down agents promptly when their work is done
- Fix stale `isActive` flags in team config if TeamDelete fails
- Never close the parent issue without user approval
- **TDD is mandatory**: tests before code, incrementally, not all at once
- **Tests must be executed** during review — not just read, but run
- Design phase gates on user PR approval before implementation begins
- Coder and tester must collaborate via SendMessage, not work in isolation
