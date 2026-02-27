# Research: Building Metapowers as Claude Code Skills

## How Claude Code Skills Work

### The Mechanics

A Claude Code skill is a **prompt template** stored as a Markdown file with YAML
frontmatter. It is not executable code -- it is instructions that Claude follows
when the skill is triggered.

**File structure:**
```
plugin-name/
├── .claude-plugin/
│   └── plugin.json            # Plugin metadata (name, author, version)
├── skills/
│   └── skill-name/
│       └── SKILL.md           # YAML frontmatter + markdown instructions
├── commands/                   # User-invokable commands (thin skill wrappers)
│   └── command-name.md
├── hooks/                      # Lifecycle hooks (session start, etc.)
│   ├── hooks.json
│   └── session-start
└── references/                 # Supplementary docs loaded on demand
```

**SKILL.md format:**
```markdown
---
name: skill-name
description: "Trigger description -- when Claude should use this skill"
---

# Skill Title

Instructions for Claude to follow when this skill is invoked.
```

**Key mechanics:**
1. The `description` field is always in Claude's context (~100 words). It
   determines WHEN Claude invokes the skill.
2. The full SKILL.md body is loaded only when the skill is invoked (~500 lines max).
3. Skills are invoked via the `Skill` tool -- either automatically (Claude decides
   based on description) or explicitly (user types `/command-name`).
4. Commands (in `commands/`) are thin wrappers: they invoke a skill and have
   `disable-model-invocation: true` so only users can trigger them.

### How Skills Chain

Skills chain by **instructing Claude to invoke another skill by name.** This is
textual, not programmatic. Examples from superpowers:

```markdown
# In brainstorming skill:
Invoke the writing-plans skill to create a detailed implementation plan.
Do NOT invoke any other skill. writing-plans is the next step.

# In plan file headers:
> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans

# In subagent-driven-development:
Use superpowers:finishing-a-development-branch
```

Chaining is always **explicit and hardcoded** in the skill text. There is no
dynamic dispatch mechanism built into Claude Code's skill system.

### How State Passes Between Skills

In superpowers, state passes through **files at known paths:**

| Artifact | Path | Format |
|----------|------|--------|
| Design doc | `docs/plans/YYYY-MM-DD-<topic>-design.md` | Markdown |
| Implementation plan | `docs/plans/YYYY-MM-DD-<feature-name>.md` | Markdown |
| Task tracking | TodoWrite (Claude's built-in task list) | Internal |
| Code | Git worktree on a feature branch | Files + commits |
| Review feedback | Subagent response text | Ephemeral |

**Critical observation:** The path convention (`docs/plans/YYYY-MM-DD-...`) is
hardcoded in the skill text. If you want artifacts stored elsewhere, you fork the
skill. This is Problem #2 from our brainstorm doc.

### How Superpowers Enforces Its Process

Superpowers uses several enforcement mechanisms:

1. **SessionStart hook**: Injects `using-superpowers` skill content into every
   session's context automatically.

2. **Hard gates**: `<HARD-GATE>` tags with explicit prohibitions ("Do NOT invoke
   any implementation skill until design is approved").

3. **Mandatory checklists**: Numbered steps that must be completed in order,
   tracked via TodoWrite.

4. **Rationalization prevention tables**: Explicit counter-arguments to common
   shortcuts ("This is too simple to test" → "Simple code breaks").

5. **Process flow diagrams**: Graphviz `digraph` blocks showing valid state
   transitions with exactly one terminal state.

6. **Explicit terminal states**: Every skill names its successor. There is no
   ambiguity about what comes next.

## The Pluggability Challenge

The core question for metapowers: **Can a skill dynamically dispatch to another
skill based on configuration, rather than hardcoding the next skill's name?**

### Analysis

Skills are prompts. Claude can:
- Read files (including config files)
- Parse YAML/JSON content
- Use the `Skill` tool with a dynamically determined skill name
- Make decisions based on what it reads

Therefore, dynamic dispatch IS possible. The pattern:

```markdown
## Phase Dispatch

1. Read the metapowers configuration file
2. Look up the strategy skill for the current phase
3. Invoke that skill using the Skill tool

Example: if config says `release.envision: metapowers:prfaq-author`,
then invoke `metapowers:prfaq-author`.
```

This is the key architectural insight: **Claude itself is the dispatch mechanism.**
We don't need a programmatic router -- we need a skill that instructs Claude to
read a config and act on it.

### Risks and Concerns

1. **Reliability**: Claude must correctly read the config, extract the right skill
   name, and invoke it. Prompt engineering must be tight.

2. **Context budget**: Each skill invocation loads ~500 lines into context. A loop
   skill that invokes a strategy skill that invokes an adapter skill could consume
   significant context.

3. **Chaining depth**: Claude Code skills don't have a formal "return" mechanism.
   When a strategy skill completes, control doesn't automatically return to the
   loop skill. The loop skill must instruct Claude on what happens after the
   strategy completes.

4. **State continuity**: Across skill invocations, Claude's context is continuous
   (within a session). But if skills are invoked via subagents, each subagent has
   fresh context.

### Proposed Pattern: Config-Driven Dispatch

Instead of:
```markdown
# Hardcoded (superpowers style)
Invoke the writing-plans skill.
```

We use:
```markdown
# Config-driven (metapowers style)
Read `metapowers.yml` and find the strategy for `release.envision`.
Invoke that strategy skill using the Skill tool.
When the strategy completes, return here and continue with the next phase.
```

The config file becomes the **wiring diagram** that connects the frozen loop
structure to the hot-spot strategy skills.

## Cross-Cutting Concern: System of Record (State Adapter)

### The Problem

Superpowers hardcodes artifact paths:
```markdown
Write the design to docs/plans/YYYY-MM-DD-<topic>-design.md
```

Metapowers needs this to be configurable. The same process should work whether
artifacts live in markdown files, GitHub Issues, or Linear tickets.

### Approach: Adapter Instructions in Config

The config file specifies not just WHICH adapter, but includes enough context for
Claude to know HOW to use it:

```yaml
adapters:
  prfaq:
    type: markdown_files
    path: "docs/releases/{release_name}/prfaq.md"
  story:
    type: github_issues
    labels: ["user-story"]
    template: |
      ## Acceptance Criteria
      {acceptance_criteria}
```

The loop skill reads this config and constructs appropriate instructions for
artifact storage. The strategy skill doesn't need to know about storage -- it
produces content, and the loop skill handles persistence.

### Approach: Adapter as Reference File

Alternatively, each adapter type has a reference file that describes how to
perform CRUD operations:

```
references/
├── adapters/
│   ├── markdown-files.md     # How to read/write markdown artifacts
│   ├── github-issues.md      # How to read/write GitHub Issues
│   └── linear.md             # How to read/write Linear tickets
```

The loop skill loads the appropriate reference based on config. This keeps adapter
knowledge out of the loop skill itself.

### Recommendation for Spike

For the tracer bullet, use the simpler approach: **adapter config with inline
path templates.** The loop skill reads the config, determines the path/mechanism,
and instructs Claude directly. This tests the core hypothesis (configurable
storage) without building a full adapter system.

## Tracer Bullet Design

### What We're Testing

1. **Dynamic dispatch**: Can a loop skill read a config and invoke a strategy skill
   by name?
2. **Configurable storage**: Can artifact storage be driven by config rather than
   hardcoded paths?
3. **Separation of concerns**: Can the process (loop) skill be independent of the
   strategy (phase) skill?

### Scope

One loop level (Release Loop), one phase (Envision), one strategy (PRFAQ Author),
one adapter (markdown files). Enough to prove the pattern works end-to-end.

### File Structure

```
metapowers/
├── .claude-plugin/
│   └── plugin.json
├── skills/
│   ├── release-loop/
│   │   └── SKILL.md              # The release loop process (frozen spot)
│   ├── prfaq-author/
│   │   └── SKILL.md              # Default PRFAQ authoring strategy (hot spot)
│   └── using-metapowers/
│       └── SKILL.md              # Bootstrap skill (like using-superpowers)
├── commands/
│   └── release.md                # /release entry point
├── references/
│   └── adapters/
│       └── markdown-files.md     # Markdown adapter reference
└── config/
    └── default.yml               # Default configuration
```

### Config Schema (default.yml)

```yaml
# Metapowers Configuration
# This file wires together the loop structure, strategies, and adapters.

# Strategy skills for each loop.phase
strategies:
  release.envision:     metapowers:prfaq-author
  release.plan:         metapowers:release-planning    # not yet implemented
  release.integrate:    metapowers:release-integration  # not yet implemented
  release.ship:         metapowers:release-delivery     # not yet implemented

# Adapter configuration for each artifact type
adapters:
  prfaq:
    type: markdown_files
    path_template: "docs/releases/{release_name}/prfaq.md"
  release_plan:
    type: markdown_files
    path_template: "docs/releases/{release_name}/release-plan.md"
  epic:
    type: markdown_files
    path_template: "docs/releases/{release_name}/epics/{epic_name}.md"
  story:
    type: markdown_files
    path_template: "docs/releases/{release_name}/stories/{story_id}.md"

# Project metadata
project:
  name: ""                        # Filled in per-project
  releases_dir: "docs/releases"
```

### Interface Points

The following are the explicit interface boundaries in the spike:

**1. Loop Skill → Config File** (read interface)
- Loop skill reads `metapowers.yml` (project root) or falls back to
  `config/default.yml` (plugin default)
- Extracts: strategy skill name for current phase, adapter config for artifact type

**2. Loop Skill → Strategy Skill** (dispatch interface)
- Loop skill invokes strategy skill via `Skill` tool
- Passes context: release name, current phase, input artifacts (references)
- Expects: strategy skill produces artifact content and signals completion

**3. Strategy Skill → Artifact Output** (content interface)
- Strategy skill produces content (e.g., a PRFAQ document)
- Strategy skill does NOT decide where to store it
- Strategy skill signals what it produced (artifact type, suggested name)

**4. Loop Skill → Adapter** (storage interface)
- Loop skill reads adapter config for the artifact type
- Loop skill stores the artifact at the configured location
- For markdown_files: writes to the path_template with variables resolved
- For github_issues: would create an issue with labels (future)

**5. Loop Skill → Next Phase** (transition interface)
- After strategy completes and artifact is stored, loop skill advances
- Reads config for next phase's strategy
- Invokes next strategy skill
- (For spike: only Envision phase is implemented; subsequent phases are stubs)

### What Success Looks Like

The spike succeeds if:

1. A user can type `/release` and the release-loop skill activates
2. The release-loop skill reads `metapowers.yml` and determines it should invoke
   `metapowers:prfaq-author` for the Envision phase
3. The prfaq-author skill runs, collaborating with the user to produce a PRFAQ
4. Control returns to the release-loop skill
5. The release-loop skill reads the adapter config and stores the PRFAQ at the
   configured path
6. The release-loop skill recognizes that subsequent phases aren't yet implemented
   and notifies the user

### What Success Would Look Like for Pluggability

After the spike, we should be able to:
1. Create an alternative strategy skill (e.g., `metapowers:lean-canvas-author`)
2. Change one line in `metapowers.yml` (`release.envision: metapowers:lean-canvas-author`)
3. Re-run `/release` and get the new strategy without touching the loop skill

Similarly:
1. Change the adapter config from `markdown_files` to `github_issues`
2. Re-run and get the PRFAQ created as a GitHub Issue instead of a markdown file
