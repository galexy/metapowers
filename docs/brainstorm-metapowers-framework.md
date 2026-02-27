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
series of nested feedback loops. We extend their model into a concrete four-level
loop structure that maps to how product development actually works -- from vision
through delivery.

### Loop Overview

```
Release Loop:    PRFAQ --> [Planning Loop] --> ... --> Release
                              |         ^
                              v         |
Planning Loop:   Decompose --> Analyze --> Prioritize --> [Execution Loops]
                                                            |         ^
                                                            v         |
Execution Loop:                                   Design --> Implement --> Verify
                                                               |         ^
                                                               v         |
Coding Loop:                                            Edit --> Test --> Fix
```

Key structural properties:
- Outer loops run less frequently; inner loops spin fast.
- Inner loops may run in **parallel** (multiple stories, multiple features).
- Each loop level consumes artifacts from its parent and produces artifacts that
  flow back up.
- The loop structure is the **frozen spot**. The framework owns it.

### Release Loop

**Cadence:** Infrequent. Corresponds to a major product release, a significant
capability milestone, or a strategic pivot. May map to work behind feature flags
rather than an actual shipping release.

**Phases:**

1. **Envision.** Author a PRFAQ (Press Release / FAQ) that articulates the customer
   value narrative for this release. This is a collaborative, iterative act -- the
   product developer is crafting a story about why this release matters to
   customers, what problems it solves, and what the experience looks like.

2. **Plan.** Descend into the Planning Loop to decompose the vision into
   deliverable work.

3. **Integrate.** Assemble completed features into a cohesive release. Validate
   that the delivered capabilities match the PRFAQ narrative.

4. **Ship.** Release preparation, deployment, and post-release stabilization.

**Lifecycle notes:**
- After initial product launch, multiple PRFAQ cycles may be active simultaneously
  for independent major capabilities.
- A single PRFAQ may span multiple planning cycles as the vision is progressively
  realized.
- The PRFAQ is a living artifact -- it evolves as planning and execution reveal
  what's feasible and what's valuable.

**Artifacts produced:** PRFAQ, release plan, release notes.

**STATE INTEGRATION POINT:** The PRFAQ and release plan must be persisted in the
system of record. This is the first point where adapter choice matters -- a PRFAQ
might live as a Google Doc, a markdown file, or a Notion page. The framework
doesn't care, but it must be reachable.

### Planning Loop

**Cadence:** Per-release, but may iterate multiple times as understanding deepens.

**Input:** PRFAQ (from Release Loop).

**Phases:**

1. **Decompose.** Break the PRFAQ vision into features and capabilities. Map
   features to **epics** -- large collections of user stories that collectively
   implement a capability. A feature may span multiple epics. An epic is a
   collection of specific use cases that collectively deliver a coherent piece of
   functionality.

   Within each epic, define **user stories** -- units of work that can be
   accomplished within a single coding session. If a story is too large, subdivide
   into sub-epics with more atomic units. This phase also identifies operational
   tasks (infrastructure, configuration, tooling) alongside feature work.

   Each user story has **clear acceptance criteria** that define user-observable
   behavior. Well-formed acceptance criteria include not just the happy path but
   well-thought-out error handling conditions. Edge cases are less likely to be
   enumerated at this stage.

2. **Analyze.** (Optional but valuable.) Examine dependencies across user stories,
   epics, and features. Assess cohesiveness of the system and the capabilities
   being defined. This is where a **holistic view** of the product becomes
   essential.

   This phase may surface:
   - Cross-epic and cross-feature dependencies not visible at the story level
   - Architectural implications of what's being delivered
   - Integration points that need coordination
   - Technical risk areas that need spikes or proof-of-concepts

   The analysis may require **technical architectural review** -- thinking through
   the system-level implications of the planned work, especially when
   interdependencies between stories are significant.

   **Output:** Dependency graphs, architectural notes, and potentially additional
   work items (technical stories, spike tasks, infrastructure work) that weren't
   visible during decomposition.

3. **Prioritize.** Sequence the work based on dependencies, risk, and value.
   Determine which stories can proceed in parallel and which must be serialized.

4. **Execute.** Descend into Execution Loops -- potentially many in parallel, one
   per user story or small group of related stories. This is typically the handoff
   point where engineers (human or AI) begin development.

5. **Review.** As execution loops complete, assess progress against the epic and
   feature goals. Determine if additional planning iterations are needed.

**Artifacts produced:** Epics, user stories with acceptance criteria, dependency
graphs, architectural analysis documents, task breakdowns.

**STATE INTEGRATION POINT:** This is the heaviest integration point. Epics and
stories need to exist in an issue tracker (GitHub Issues, Linear, Jira). Design
and analysis documents need to be in a document store (markdown, Google Docs,
Confluence). Dependencies need to be expressed in whatever system tracks them.
The adapter layer must support all of these artifact types and their relationships.

**HANDOFF POINT:** The transition from Planning to Execution is one of the most
significant handoff points in the entire process. The system of record is the
primary mechanism for this handoff -- the executing agent (or engineer) picks up
a story from the tracker, reads its acceptance criteria, and begins work. The
quality of this handoff depends entirely on the state adapter's ability to
surface the right context.

### Execution Loop

**Cadence:** Per-story. Multiple execution loops may run in parallel.

**Input:** A user story with acceptance criteria (from Planning Loop).

**Phases:**

1. **Design.** Think through how to implement the story. Consider the approach,
   affected code areas, integration points, and potential risks.

2. **Implement.** Write the code. Descend into the Coding Loop for the
   edit-test-fix cycle.

3. **Verify.** Confirm the implementation meets acceptance criteria. This includes:
   - Automated tests (unit, integration) developed alongside the code
   - Code that is working and capable of integrating into the mainline
   - Potentially: BDD-style tests that automate the user acceptance path
   - Potentially: proof that agents have manually tested acceptance paths
     (calling API endpoints, running CLI commands, driving a headless browser)

4. **Complete.** The story is done. Artifacts flow back to the Planning Loop.

**THIS IS THE PRIMARY CUSTOMIZATION POINT.** The framework explicitly does not
dictate how the Execution Loop's phases are implemented. This is by design -- it
is the hottest of the hot spots. Examples of how a user might customize it:

- **Team brainstorm strategy:** Spawn a team of agents -- one group brainstorms
  the design approach, another group thinks through testing (white box and black
  box perspectives). Design and test spec work products are intermediate outputs
  that can optionally be reviewed before development begins. This is akin to
  design reviews and test spec reviews before committing to implementation.

- **TDD strategy:** Write tests first from the acceptance criteria, then implement
  until tests pass. The verification phase is inherently satisfied by the process.

- **Spike-then-formalize strategy:** Build a rough prototype first, demo it for
  feedback, then formalize or discard.

- **Pair programming strategy:** Human and AI collaborate interactively throughout
  design and implementation, with autonomous verification.

**Artifacts produced:** Design documents, test specifications, implementation code,
automated tests, verification results.

**STATE INTEGRATION POINT:** As implementation proceeds, the story's status in the
issue tracker must be updated (in progress, in review, done). Design documents and
test specs are intermediate artifacts that may live in the document store. Code
changes flow through git. Pull requests may be created for review. Each of these
is a separate adapter concern.

**COMMUNICATION POINT:** During execution, the agent (or engineer) may need to
communicate status, ask questions, or surface blockers. The system of record is the
primary channel for this -- comments on issues, PR discussions, status updates. The
flow policy determines whether these communications are informational (notify) or
blocking (approve). Optionally, other communication mechanisms (messaging systems
between agents or personas) may be used, but these are implementation details of
specific phase strategies, not constraints of the framework.

### Coding Loop

**Cadence:** Rapid. Many iterations per story.

**Phases:**

1. **Edit.** Make a code change.
2. **Test.** Run relevant tests or checks.
3. **Fix.** Address failures. If the fix is non-trivial, loop back to Edit.

This is the tightest loop -- it runs almost entirely autonomously. Human
involvement is rare here and typically only occurs when the agent is stuck.

**Artifacts produced:** Code changes, test results (transient).

---

### Parallelism and Independence

A critical structural property: **inner loops are independent of each other.**

- Multiple Release Loops can run concurrently for independent product lines or
  major capabilities.
- Multiple Planning Loop iterations can run if the release has separable feature
  tracks.
- Multiple Execution Loops run in parallel -- one per story (or small story group).
  This is the primary unit of parallelism in development.
- Coding Loop iterations are sequential within a single execution context.

The framework must support this. The loop engine doesn't just manage a single
stack of nested loops -- it manages a **tree** of potentially concurrent loop
instances, each with its own state.

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
  release.envision -> release.plan:         { mode: hitl_collaborate }
  release.integrate -> release.ship:        { mode: hitl_approve }

  planning.decompose -> planning.analyze:   { mode: autonomous }
  planning.analyze -> planning.prioritize:  { mode: hitl_notify }
  planning.prioritize -> planning.execute:  { mode: hitl_approve }
  planning.execute -> planning.review:      { mode: autonomous }

  execution.design -> execution.implement:  { mode: autonomous }
  execution.implement -> execution.verify:  { mode: autonomous }
  execution.verify -> execution.complete:   { mode: autonomous }

  coding.*:                                 { mode: autonomous }
```

```yaml
# Tight oversight profile
policies:
  release.*:                                { mode: hitl_collaborate }

  planning.decompose -> planning.analyze:   { mode: hitl_approve }
  planning.analyze -> planning.prioritize:  { mode: hitl_approve }
  planning.prioritize -> planning.execute:  { mode: hitl_approve }
  planning.execute -> planning.review:      { mode: hitl_approve }

  execution.design -> execution.implement:  { mode: hitl_approve }
  execution.implement -> execution.verify:  { mode: autonomous }
  execution.verify -> execution.complete:   { mode: hitl_approve }

  coding.*:                                 { mode: autonomous }
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
update_status(ref, status) -> void
```

Artifact types are semantic, not format-specific:
- `prfaq` -- press release / FAQ narrative
- `release_plan` -- high-level release plan
- `epic` -- large work unit grouping related stories
- `story` -- user story with acceptance criteria
- `task` -- operational or technical task
- `design_doc` -- architectural or functional design document
- `test_spec` -- test plan or specification
- `dependency_graph` -- cross-story/epic dependency analysis
- `implementation` -- code changes (branches, commits, PRs)
- `review` -- code or design review
- `decision` -- architectural decision record
- `verification_result` -- test results, acceptance proof

**Adapter implementations:**
- `markdown_files`: reads/writes markdown in a directory structure
- `github_issues`: maps stories, epics, tasks to GitHub Issues with labels
- `github_pr`: maps implementations and reviews to Pull Requests
- `linear`: maps stories/epics/tasks to Linear tickets with relationships
- `beads`: maps context and working state to Beads
- `google_docs`: maps PRFAQs, design docs, and prose to Google Docs
- `git`: maps implementations to branches/commits

**Composability by artifact type:**

```yaml
adapters:
  prfaq:                google_docs
  release_plan:         markdown_files
  epic:                 github_issues
  story:                github_issues
  task:                 github_issues
  design_doc:           markdown_files     # docs/design/
  test_spec:            markdown_files     # docs/tests/
  dependency_graph:     markdown_files     # docs/analysis/
  implementation:       git
  review:               github_pr
  decision:             markdown_files     # docs/decisions/
  verification_result:  markdown_files     # docs/verification/
```

A user who wants everything local uses `markdown_files` for all types. A team
using Linear and Google Docs routes accordingly. The loop engine and phase
strategies are unaffected by this choice.

**The system of record is the sum of all configured adapters.** It doesn't live in
one place -- it's the composition of wherever each artifact type is stored. The
framework's job is to abstract over this composition so that phase strategies and
the loop engine interact with artifacts uniformly regardless of backing store.

**Minimum requirements for any adapter composition:**
- Every artifact type used by the configured loop levels must have an adapter
- Adapters must support `link()` to express cross-artifact relationships (e.g.,
  story-to-epic, story-to-design-doc, story-to-PR)
- Adapters must support `update_status()` for artifact types that have lifecycle
  states (stories, tasks, epics)
- Cross-adapter linking must work: a story in GitHub Issues must be linkable to a
  design doc in markdown. The framework needs a **unified reference format** that
  adapters can resolve regardless of backend.

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
  release.envision:           { skill: "prfaq_author" }
  release.plan:               { skill: "release_planning" }
  release.integrate:          { skill: "release_integration" }
  release.ship:               { skill: "release_delivery" }

  planning.decompose:         { skill: "epic_story_decomposition" }
  planning.analyze:           { skill: "dependency_analysis" }
  planning.prioritize:        { skill: "work_prioritization" }
  planning.execute:           { skill: "story_dispatch" }
  planning.review:            { skill: "progress_review" }

  execution.design:           { skill: "story_design" }
  execution.implement:        { skill: "story_implement" }
  execution.verify:           { skill: "story_verify" }
  execution.complete:         { skill: "story_complete" }

  coding.edit:                { skill: "edit_code" }
  coding.test:                { skill: "run_tests" }
  coding.fix:                 { skill: "fix_from_failure" }
```

```yaml
# Team-of-agents execution variant
strategies:
  execution.design:           { skill: "team_brainstorm_design" }
  execution.implement:        { skill: "team_implement" }
  execution.verify:           { skill: "team_verify_whitebox_blackbox" }
```

```yaml
# TDD execution variant
strategies:
  execution.design:           { skill: "tdd_test_design_from_acceptance" }
  execution.implement:        { skill: "tdd_red_green_refactor" }
  execution.verify:           { skill: "tdd_coverage_and_acceptance" }
```

```yaml
# Spike-then-formalize execution variant
strategies:
  execution.design:           { skill: "spike_scope" }
  execution.implement:        { skill: "spike_prototype" }
  execution.verify:           { skill: "spike_evaluate_and_formalize" }
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
    for artifact in result.artifacts:
      ref = adapter.write(artifact.type, artifact.id, artifact.content)
      adapter.update_status(ref, artifact.status)
      for link in artifact.links:
        adapter.link(ref, link.target, link.relation)

    # 5. Evaluate transition based on result + flow policy
    transition_key = loop_level + "." + phase + " -> " + next_phase(state)
    policy = config.policies[transition_key]

    match policy.mode:
      autonomous       -> state.advance(result)
      hitl_notify      -> notify_user(result); state.advance(result)
      hitl_approve     -> decision = ask_user(result); state.apply(decision)
      hitl_collaborate -> state.apply(collaborate_with_user(result))

    # 6. Handle loop transitions
    match result.status:
      done       -> state.exit_loop()           # escalate to outer
      needs_more -> state.continue_loop()       # stay in current loop
      descend    -> state.enter_inner_loop()    # descend to inner loop
      parallel   -> state.fork_inner_loops(result.work_items)  # fan-out
      blocked    -> state.pause(); notify_user(result)
      failed     -> state.handle_failure(policy)

    # 7. Handle parallel completion (join)
    if state.has_pending_parallel:
      completed = state.collect_completed_loops()
      if state.all_parallel_complete:
        state.join_and_advance(completed)
```

## State Integration Point Map

The following table identifies every point in the loop structure where the state
adapter is invoked. These are the places where the choice of backing store matters
and where adapter quality directly affects the workflow.

| Loop Level | Phase          | Artifacts Written            | Status Updates    | Links Created                    |
|------------|----------------|------------------------------|-------------------|----------------------------------|
| Release    | Envision       | prfaq                        | --                | --                               |
| Release    | Plan           | release_plan                 | --                | release_plan -> prfaq            |
| Release    | Integrate      | verification_result          | release_plan      | --                               |
| Release    | Ship           | --                           | release_plan      | --                               |
| Planning   | Decompose      | epic, story, task            | --                | story -> epic, task -> epic      |
| Planning   | Analyze        | dependency_graph, decision   | story (re-ranked) | dependency links across stories  |
| Planning   | Analyze        | story, task (new ones)       | --                | new stories -> epic              |
| Planning   | Prioritize     | --                           | story (ordered)   | --                               |
| Planning   | Execute        | --                           | story (assigned)  | --                               |
| Planning   | Review         | --                           | epic, story       | --                               |
| Execution  | Design         | design_doc, test_spec        | story (in prog)   | design_doc -> story              |
| Execution  | Implement      | implementation               | story (in prog)   | implementation -> story          |
| Execution  | Verify         | verification_result, review  | story (in review) | verification -> story, PR -> story |
| Execution  | Complete       | --                           | story (done)      | --                               |
| Coding     | Edit           | implementation (transient)   | --                | --                               |
| Coding     | Test           | verification_result (trans.) | --                | --                               |
| Coding     | Fix            | implementation (transient)   | --                | --                               |

## Communication and Handoff Model

**Primary principle:** The system of record IS the communication mechanism.

Handoff between phases and between loop levels happens through artifacts in the
state adapters. When the Planning Loop creates a story in GitHub Issues, that IS
the handoff to the Execution Loop -- the executing agent picks up the story from
the tracker.

This has several implications:

1. **HiTL observation is adapter-mediated.** A human reviewing progress looks at
   the system of record -- the issue tracker, the document store, the PR list.
   The framework doesn't need a separate "dashboard" if the adapters keep the
   system of record current.

2. **HiTL feedback flows through the same adapters.** A human commenting on a
   GitHub Issue or a PR is providing feedback that the framework can read through
   the adapter. The flow policy determines whether the framework pauses to solicit
   this feedback or proceeds autonomously.

3. **Agent-to-agent communication is an implementation detail.** If a phase
   strategy spawns multiple agents (e.g., the team brainstorm strategy), how those
   agents communicate internally is up to the strategy. The framework doesn't
   constrain it. What matters is that the strategy's outputs flow back through the
   state adapter.

4. **Notifications are a flow policy concern.** Whether a human is notified of
   phase transitions (Slack message, email, GitHub notification) is part of the
   flow policy's `hitl_notify` and `hitl_approve` modes. The notification
   mechanism itself could be another pluggable adapter, but it's a sub-concern
   of flow policy, not a fourth axis.

## Orthogonality Verification

| Change desired                      | Axis touched              | Everything else unchanged                  |
|-------------------------------------|---------------------------|--------------------------------------------|
| More autonomy in execution loop     | Flow Policy               | State, Strategies, Loop structure          |
| Tighter review at planning phase    | Flow Policy               | State, Strategies, Loop structure          |
| Move stories to Linear              | State Adapter (story)     | Flow, Strategies, Loop structure           |
| PRFAQs in Google Docs               | State Adapter (prfaq)     | Flow, Strategies, Loop structure           |
| Switch to TDD execution             | Phase Strategy (exec.*)   | Flow, State, Loop structure                |
| Team brainstorm before coding       | Phase Strategy (exec.*)   | Flow, State, Loop structure                |
| Add deployment loop level           | Loop structure + defaults | Existing policies/adapters/strategies      |
| Change design review to async       | Flow Policy (one trans.)  | State, Strategies, Loop structure          |
| Agent-to-agent messaging in exec    | Phase Strategy internals  | Flow, State, Loop structure, other phases  |

## Defaults (Works Out of the Box)

A zero-configuration installation should provide:

- **Flow Policy**: release/planning loops use `hitl_collaborate` or `hitl_approve`;
  execution loop uses `hitl_approve` for design, `autonomous` for implement/verify;
  coding loop is fully `autonomous`
- **State Adapter**: `markdown_files` for all artifact types, stored in
  `.metapowers/` or a configurable project directory
- **Phase Strategies**: basic skill implementations for every phase at every loop
  level
- **Loop Structure**: the four-level model (release/planning/execution/coding)

Users progressively specialize. Start with defaults, then:
1. Tune the flow policy to match your autonomy preference
2. Swap state adapters as your tooling preferences emerge
3. Replace phase strategies to match your development style
4. Optionally: add custom loop levels (deployment, incident response)

## Open Questions

- **Configuration format**: YAML? TOML? JSON? A `metapowers.yml` in the project
  root? Or integrated into `.claude/settings.json`?

- **Skill discovery**: How does the engine find and validate phase strategy skills?
  A registry? Convention-based naming? A manifest file?

- **Loop structure extensibility**: The four-level model is a sensible default, but
  should users be able to define custom loop levels? (e.g., a "deployment" loop, an
  "incident response" loop, a "sprint" wrapper). How are custom levels registered
  and wired into the engine?

- **Artifact schema**: How structured should artifacts be? Fully schemaless (pass
  opaque content through adapters)? Loosely typed (semantic types with optional
  schemas)? Strictly typed? User stories have a known structure (description,
  acceptance criteria); PRFAQs have a known structure (press release, FAQ). Should
  the framework enforce or merely suggest these schemas?

- **Unified reference format**: When a story in GitHub Issues references a design
  doc in markdown, how does `link()` work across adapter boundaries? We likely need
  a canonical reference format like `adapter_type:artifact_type:id` that any adapter
  can resolve.

- **Parallel execution model**: When the Planning Loop fans out to multiple
  Execution Loops, how does the engine manage them? Threads? Subprocesses? Claude
  Code subagents? Is there a max concurrency? How are results collected (join)?

- **Error recovery**: When a phase strategy fails, how does the engine decide
  between retry, escalate, and abort? Is this a sub-concern of flow policy or a
  separate concern?

- **Context passing**: What information flows between phases beyond artifacts?
  Session history? Previous decisions? Iteration counts? Is there a "context
  object" that the engine maintains and passes to every strategy invocation?

- **Composable strategies**: Can a phase strategy itself be a mini-loop? (e.g.,
  the "team brainstorm" strategy internally runs design-agents and test-agents in
  sequence). Is this just nesting, or does the framework need explicit support for
  strategy composition?

- **Entry points**: Can a user enter the framework at any loop level? (e.g., skip
  the release loop and start at planning with a pre-existing set of stories; or
  start at execution with a single story). What state must be bootstrapped?

- **Adapter minimum requirements**: What is the minimum interface a new adapter
  must implement? Which methods are optional? (e.g., `link()` might not be
  meaningful for all backends; `update_status()` might not apply to document-type
  artifacts).

## Inspirations and References

- **Gene Kim & Steve Yegge**, *Vibe Coding* -- nested feedback loops as the
  fundamental structure of software development
- **Wolfgang Pree**, *Design Patterns for Object-Oriented Software Development* --
  frozen spots and hot spots in framework design
- **Ralph Johnson & Brian Foote**, "Designing Reusable Classes" -- inversion of
  control as the defining characteristic of frameworks
- **Martin Fowler**, "InversionOfControl" -- the Hollywood Principle
- **Amazon's PRFAQ process** -- working backwards from the customer narrative to
  drive product development
- **MFC (Microsoft Foundation Classes)** -- document/view architecture as a
  canonical example of framework-with-extension-points
- **Superpowers plugin** -- example of the monolithic approach we're decomposing
- **GSD plugin** -- example of subagent-based execution with hardcoded flow
