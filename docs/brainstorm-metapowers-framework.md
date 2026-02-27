# Metapowers: A Framework for AI-Assisted Software Development

## The Problem

Existing Claude Code skills and plugins (Superpowers, GSD, etc.) each define an
entire SDLC workflow monolithically. They bundle three concerns together:

1. **The loop structure** (what phases exist and how they connect)
2. **The state management** (how work products are stored and passed between phases)
3. **The operational logic** (what actually happens inside each phase)

This creates three concrete problems:

**Problem 1: Flow control is baked in.** Each plugin hardcodes whether transitions
are autonomous or human-in-the-loop. A plugin designed for high autonomy can't
easily be tuned toward tighter human oversight, or vice versa. The conceptual flow
and the control policy are fused.

**Problem 2: State is coupled to a storage mechanism.** Plugins typically store
workflow state as markdown files in a specific directory structure. This makes it
difficult to use GitHub Issues for tasks, Linear for project management, Beads for
context, or Google Docs for design prose -- or to mix and match across these backends.

**Problem 3: Phase implementations are hardcoded.** The execution logic at each
step is wired directly into the workflow. Changing the implementation phase from
"autonomous coding" to "test-driven development" or "spike-then-formalize" requires
forking the entire plugin, not swapping a single component.

These plugins aren't frameworks. They're recipes.

## What Makes a Framework

Drawing from MFC, Rails, Spring, React, and Django -- a framework has four
essential qualities:

1. **Inversion of Control.** The framework calls your code, not the other way
   around. The Hollywood Principle: "Don't call us, we'll call you."

2. **Frozen spots and hot spots.** The framework has invariant structure (frozen
   spots) that you don't rewrite, and well-defined extension points (hot spots)
   where you plug in domain-specific behavior. (Wolfgang Pree's terminology.)

3. **Orthogonality.** Different concerns are independently extensible. Changing
   one axis of behavior doesn't force changes on another.

4. **Default behavior.** The framework works out of the box with sensible defaults.
   You progressively specialize only what you need to change.

The test: *Can I extend one aspect without rewriting another? Does the framework
call me, or do I drive it?* If the answer to both is no, it's a recipe, not a
framework.

## The Nested Loop Model

Gene Kim and Steve Yegge describe software development in *Vibe Coding* as a
series of nested feedback loops. This is the conceptual core we're building on:

```
Outer Loop:   Understand --> Design --> Implement --> Verify --> Deliver
                                          |    ^
                                          v    |
Inner Loop:                        Code --> Run --> Evaluate
                                            |    ^
                                            v    |
Micro Loop:                           Edit --> Test --> Fix
```

Each loop level has:
- **Entry conditions**: when you descend into this loop
- **Phase sequence**: the steps within the loop
- **Transition decisions**: continue the current loop, descend to an inner loop,
  or escalate to the outer loop
- **Exit conditions**: when the loop's work is done and control returns outward

The loop structure itself is the **frozen spot**. The framework owns it. Individual
skills never decide what phase comes next -- they do their work, produce artifacts,
and return control to the engine.

## Architecture: Three Orthogonal Axes

```
                    +---------------------------+
                    |   Loop Engine (frozen)     |
                    |   enter --> dispatch -->   |
                    |   evaluate --> transition  |
                    +-----+--------+--------+---+
                          |        |        |
              +-----------+        |        +----------+
              v                    v                   v
     +---------------+    +----------------+    +---------------+
     |  Flow Policy  |    | State Adapter  |    |Phase Strategy |
     |   (Axis 1)    |    |   (Axis 2)     |    |   (Axis 3)    |
     +---------------+    +----------------+    +---------------+
       Problem #1           Problem #2           Problem #3
```

### Axis 1: Flow Policy

Governs what happens *between* phases -- not what happens inside them.

Each transition has a mode:
- **autonomous**: proceed without human input
- **hitl_notify**: inform the human, then proceed
- **hitl_approve**: pause for human approval before proceeding
- **hitl_collaborate**: work interactively with the human

Example configurations:

```yaml
# High autonomy profile
policies:
  outer.understand -> outer.design:     { mode: hitl_approve }
  outer.design -> outer.implement:      { mode: hitl_approve }
  outer.implement -> outer.verify:      { mode: autonomous }
  inner.code -> inner.run:              { mode: autonomous }
  inner.run -> inner.evaluate:          { mode: autonomous }
  inner.evaluate -> outer.verify:       { mode: hitl_notify }
  micro.*:                              { mode: autonomous }

# Tight oversight profile
policies:
  outer.*:                              { mode: hitl_approve }
  inner.evaluate -> outer.*:            { mode: hitl_approve }
  inner.code -> inner.run:              { mode: autonomous }
  micro.*:                              { mode: autonomous }
```

Key insight: the loop structure is identical in both profiles. Only the transition
policy changes. A user can tune the autonomy dial without touching strategies or
state management.

Additional policy parameters per transition:
- `max_iterations`: cap on loop cycles before forced escalation
- `escalation_condition`: when to break out of the current loop level
- `timeout`: wall-clock limit before pausing for human input

### Axis 2: State Adapter

The framework reads and writes artifacts. It doesn't care how or where.

**Interface:**

```
read(artifact_type, id) -> content
write(artifact_type, id, content) -> ref
list(artifact_type, query?) -> ref[]
link(from_ref, to_ref, relation) -> void
```

Artifact types are semantic, not format-specific:
- `requirement`
- `design_doc`
- `task`
- `test_spec`
- `implementation`
- `review`
- `decision`

**Adapter implementations:**
- `markdown_files`: reads/writes markdown in a directory structure
- `github_issues`: maps tasks and requirements to GitHub Issues
- `github_pr`: maps reviews to Pull Requests
- `linear`: maps tasks to Linear tickets
- `beads`: maps context to Beads
- `git`: maps implementations to branches/commits

**Composability by artifact type:**

```yaml
adapters:
  requirement:      github_issues
  design_doc:       markdown_files    # prose in docs/design/
  task:             linear            # work items in Linear
  test_spec:        markdown_files    # test plans in docs/tests/
  implementation:   git
  review:           github_pr
  decision:         markdown_files    # ADRs in docs/decisions/
```

A user who wants everything local uses `markdown_files` for all types. A user on a
team routes tasks to Linear and reviews to GitHub PRs. The loop engine and phase
strategies are unaffected by this choice.

### Axis 3: Phase Strategy

Each phase in each loop level has a pluggable implementation -- a skill that
performs the actual work.

**Interface contract for a phase strategy:**

A phase strategy skill receives:
- The current context (loop level, phase, iteration count)
- Input artifacts (read via the state adapter)
- The flow policy for reference (but not for decision-making)

A phase strategy skill returns:
- Output artifacts (written via the state adapter)
- A result status: `done`, `needs_more`, `blocked`, `failed`
- Optional: suggested next action (advisory, not authoritative)

The engine -- not the strategy -- decides what happens next based on the result
status and the flow policy.

**Example strategy configurations:**

```yaml
# Default strategies
strategies:
  outer.understand:   { skill: "requirements_elicitation" }
  outer.design:       { skill: "architecture_sketch" }
  outer.implement:    { skill: "task_decomposition" }
  outer.verify:       { skill: "integration_review" }
  outer.deliver:      { skill: "release_prep" }

  inner.code:         { skill: "write_code" }
  inner.run:          { skill: "run_and_observe" }
  inner.evaluate:     { skill: "evaluate_result" }

  micro.edit:         { skill: "edit_code" }
  micro.test:         { skill: "run_unit_test" }
  micro.fix:          { skill: "fix_from_failure" }
```

```yaml
# TDD-heavy variant (only inner loop changes)
strategies:
  inner.code:         { skill: "tdd_write_test_first" }
  inner.run:          { skill: "tdd_run_tests" }
  inner.evaluate:     { skill: "tdd_coverage_check" }
```

```yaml
# Spike-then-formalize variant
strategies:
  inner.code:         { skill: "spike_prototype" }
  inner.run:          { skill: "demo_to_user" }
  inner.evaluate:     { skill: "formalize_or_discard" }
```

## The Loop Engine (Pseudocode)

```
loop_engine(context, config):
  state = load_or_init_state(context)

  while not state.complete:
    phase = state.current_phase
    loop_level = state.current_loop

    # 1. Load the strategy for this phase
    strategy = config.strategies[loop_level + "." + phase]

    # 2. Read input artifacts through the adapter
    artifacts = adapter.read(strategy.input_types, state.current_refs)

    # 3. Invoke the phase strategy (INVERSION OF CONTROL)
    result = invoke_skill(strategy.skill, artifacts, context)

    # 4. Write output artifacts through the adapter
    adapter.write(result.artifact_type, result.id, result.content)

    # 5. Evaluate transition based on result + flow policy
    transition_key = loop_level + "." + phase + " -> " + next_phase(state)
    policy = config.policies[transition_key]

    match policy.mode:
      autonomous      -> state.advance(result)
      hitl_notify     -> notify_user(result); state.advance(result)
      hitl_approve    -> decision = ask_user(result); state.apply(decision)
      hitl_collaborate -> state.apply(collaborate_with_user(result))

    # 6. Handle loop transitions
    match result.status:
      done       -> state.exit_loop()           # escalate to outer
      needs_more -> state.continue_loop()       # stay in current loop
      descend    -> state.enter_inner_loop()    # descend to inner loop
      blocked    -> state.pause(); notify_user(result)
      failed     -> state.handle_failure(policy)
```

## Orthogonality Verification

| Change desired                  | Axis touched              | Everything else unchanged              |
|---------------------------------|---------------------------|----------------------------------------|
| More autonomy in inner loop     | Flow Policy               | State, Strategies, Loop structure      |
| Less autonomy in outer loop     | Flow Policy               | State, Strategies, Loop structure      |
| Move tasks to Linear            | State Adapter (task)      | Flow, Strategies, Loop structure       |
| Store design docs in Google     | State Adapter (design_doc)| Flow, Strategies, Loop structure       |
| Switch to TDD                   | Phase Strategy (inner.*)  | Flow, State, Loop structure            |
| Add spike-first exploration     | Phase Strategy (inner.*)  | Flow, State, Loop structure            |
| Add a deployment loop level     | Loop structure + defaults | Existing policies/adapters/strategies  |
| Change review to pair-program   | Phase Strategy (verify)   | Flow, State, Loop structure            |

## Defaults (Works Out of the Box)

A zero-configuration installation should provide:

- **Flow Policy**: outer loops use `hitl_approve`, inner/micro loops use `autonomous`
- **State Adapter**: `markdown_files` for all artifact types, stored in `.claude/workflow/`
- **Phase Strategies**: basic skill implementations for every phase
- **Loop Structure**: the three-level nested model (outer/inner/micro)

Users progressively specialize. Start with defaults, then:
1. Tune the flow policy to match your autonomy preference
2. Swap state adapters as your tooling preferences emerge
3. Replace phase strategies to match your development style

## Open Questions

- **Configuration format**: YAML? TOML? JSON? A `metapowers.yml` in the project
  root? Or integrated into `.claude/settings.json`?

- **Skill discovery**: How does the engine find and validate phase strategy skills?
  A registry? Convention-based naming? A manifest file?

- **Loop structure extensibility**: The three-level model (outer/inner/micro) is a
  sensible default, but should users be able to define custom loop levels? (e.g.,
  a "sprint" loop wrapping the outer loop, or a "refactor" loop between inner and
  micro)

- **Artifact schema**: How structured should artifacts be? Fully schemaless (pass
  opaque content through adapters)? Loosely typed (semantic types with optional
  schemas)? Strictly typed?

- **Cross-adapter linking**: When a task in Linear references a design doc in
  markdown, how does `link()` work across adapter boundaries? Do we need a unified
  reference format?

- **Error recovery**: When a phase strategy fails, how does the engine decide
  between retry, escalate, and abort? Is this a fourth axis (error policy) or a
  sub-concern of flow policy?

- **Context passing**: What information flows between phases beyond artifacts?
  Session history? Previous decisions? Iteration counts? Is there a "context"
  adapter separate from state?

- **Composable strategies**: Can a phase strategy itself be composed of sub-strategies?
  (e.g., "implement" = decompose + for-each-task(code + test)). Is this just the
  inner loop, or is there a strategy composition mechanism?

## Inspirations and References

- **Gene Kim & Steve Yegge**, *Vibe Coding* -- nested feedback loops as the
  fundamental structure of software development
- **Wolfgang Pree**, *Design Patterns for Object-Oriented Software Development* --
  frozen spots and hot spots in framework design
- **Ralph Johnson & Brian Foote**, "Designing Reusable Classes" -- inversion of
  control as the defining characteristic of frameworks
- **Martin Fowler**, "InversionOfControl" -- the Hollywood Principle
- **MFC (Microsoft Foundation Classes)** -- document/view architecture as a
  canonical example of framework-with-extension-points
- **Superpowers plugin** -- example of the monolithic approach we're decomposing
- **GSD plugin** -- example of subagent-based execution with hardcoded flow
