# Markdown Files Adapter

Reference documentation for the `markdown_files` state adapter.

## Overview

The markdown files adapter stores artifacts as markdown files in a directory
structure within the project repository. This is the simplest adapter and the
default for all artifact types.

## Configuration

```yaml
adapters:
  <artifact_type>:
    type: markdown_files
    path_template: "path/to/{variable}/filename.md"
```

## Path Template Variables

| Variable | Source | Example |
|----------|--------|---------|
| `{release_name}` | Release context (set at loop start) | `v2-auth-overhaul` |
| `{epic_name}` | Epic context (set during planning) | `user-onboarding` |
| `{story_id}` | Story identifier | `AUTH-042` |
| `{date}` | Current date | `2026-02-26` |
| `{doc_name}` | Document name (from strategy) | `api-design` |

## Operations

### write(artifact_type, content, variables)

1. Resolve the `path_template` by replacing `{variable}` placeholders with the
   provided variable values.
2. Create parent directories if they don't exist (`mkdir -p`).
3. Write the content to the resolved path.
4. Stage and commit the file to git with a descriptive message.

### read(artifact_type, variables)

1. Resolve the `path_template`.
2. Read the file at the resolved path.
3. Return the content. If the file doesn't exist, return null.

### list(artifact_type, variables)

1. Resolve the path template up to the first unresolved variable.
2. Glob for files matching the pattern.
3. Return the list of resolved paths and their metadata (modification time).

### update_status(ref, status)

For markdown files, status is tracked as YAML frontmatter in the file:

```markdown
---
status: in_progress
updated: 2026-02-26
---

# Document content...
```

### link(from_ref, to_ref, relation)

For markdown files, links are expressed as markdown links within the document
or as a `related` section in YAML frontmatter:

```markdown
---
related:
  - type: epic
    ref: docs/releases/v2/epics/user-onboarding.md
    relation: parent
---
```
