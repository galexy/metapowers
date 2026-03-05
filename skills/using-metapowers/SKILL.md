---
name: using-metapowers
description: "Use when the user wants to start a development workflow, manage a release, or work through the SDLC. Metapowers provides a pluggable framework with configurable strategies and state adapters."
---

# Using Metapowers

Metapowers is a **framework** for AI-assisted software development. Unlike
monolithic workflow plugins, it separates three concerns:

1. **Loop structure** (frozen): The nested feedback loops that define the SDLC
2. **Strategy skills** (hot spot): Pluggable implementations for each phase
3. **State adapters** (hot spot): Configurable storage for artifacts

## Available Loop Skills

| Skill | Description | Entry Command |
|-------|-------------|---------------|
| `metapowers:release-loop` | Release cycle: Envision → Plan → Integrate → Ship | `/release` |

## Configuration

Metapowers reads configuration from `metapowers.yml` in the project root. If not
found, it uses built-in defaults.

To customize, copy the default config:
```
cp <plugin-root>/config/default.yml ./metapowers.yml
```

Then edit `metapowers.yml` to:
- **Swap strategies**: Change which skill handles each phase
- **Change adapters**: Route artifacts to different storage backends
- **Set project metadata**: Name, release directory, etc.

## How It Works

When a loop skill (like `release-loop`) runs:

1. It reads the config to find which strategy skill handles the current phase
2. It invokes that strategy skill via the Skill tool
3. The strategy skill collaborates with the user to produce artifacts
4. Control returns to the loop, which stores artifacts via the configured adapter
5. The loop advances to the next phase

**The loop owns the process. Strategies own the work. Adapters own the storage.**

## Creating Custom Strategies

A strategy skill is a normal SKILL.md that follows the strategy interface:

1. **DO** collaborate with the user to produce the phase's artifact
2. **DO** signal completion clearly when done
3. **DO NOT** decide what phase comes next (the loop handles transitions)
4. **DO NOT** store artifacts directly (the loop handles storage)
5. **DO NOT** invoke other loop or strategy skills

To use a custom strategy, create it as a skill and update `metapowers.yml`:

```yaml
strategies:
  release.envision: "my-plugin:lean-canvas-author"  # instead of prfaq-author
```
