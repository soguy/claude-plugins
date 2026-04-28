# Design: Blueprint — Real Claude Code Subagents

**Date:** 2026-04-28  
**Status:** Approved

## Problem

The blueprint skill creates `.claude/agents/*.md` files that describe roles but are not recognised by Claude Code as real subagents. A valid Claude Code subagent file requires YAML frontmatter (`name`, `description`) at the top of the file. Without it, the files are plain documentation — Claude Code does not auto-delegate to them and they cannot be `@`-mentioned.

## Goal

Make every agent file blueprint generates a real Claude Code subagent, auto-discoverable by Claude Code's delegation system, while preserving the existing managed-section re-run contract.

## Scope

One file changes: `plugins/blueprint/.claude/skills/blueprint/SKILL.md`.

- Step 7 agent file templates (5 default roles)
- Pending layer agent file stubs (same 5 roles)
- Custom role generation logic (Step 5 / Step 7)

No other files, steps, or logic change.

## Design

### Frontmatter format

Each agent file gets YAML frontmatter prepended, using YAML `#` comments as managed-section markers:

```markdown
---
# scaffold:begin managed frontmatter
name: pm
description: Owns task classification, specs, acceptance criteria, and closeout. Use for planning, requirements analysis, and task triage.
# scaffold:end managed frontmatter
---
<!-- scaffold:begin managed pm-role -->
Owns task classification, specs, acceptance criteria, and closeout.
Run `superpowers:brainstorming` before Track 1 specs.
For UI work, run `frontend-design` before Tech Lead handoff.
<!-- scaffold:end managed pm-role -->
```

### Re-run contract

Extends the existing managed-section rules:

| State | Behaviour |
|---|---|
| `# scaffold:begin/end managed frontmatter` present | Regenerate entire frontmatter block on re-run |
| Markers absent | Skip — user has customised, never overwrite |
| File missing | Create with full frontmatter + managed body |

This is the same contract as HTML comment managed sections, adapted for YAML syntax.

### Role descriptions

| File | `name` | `description` |
|---|---|---|
| `pm.md` | `pm` | Owns task classification, specs, acceptance criteria, and closeout. Use for planning, requirements analysis, and task triage. |
| `tech-lead.md` | `tech-lead` | Owns architecture decisions, delegation, code review, and PR readiness. Use for technical design, review, and merge decisions. |
| `dev-worker.md` | `dev-worker` | Implements assigned changes, writes tests, and documents verification steps. Use for code implementation tasks. |
| `qa-verifier.md` | `qa-verifier` | Independently verifies changes meet acceptance criteria. Use after implementation to verify correctness and run tests. |
| `devops.md` | `devops` | Owns deployment and deployment verification. Use when deploying or troubleshooting infrastructure. |

### Tools and model

- **tools:** omitted (Claude Code defaults to full tool access)
- **model:** omitted (inherits from parent session)

Users who want role-specific tools or models add them outside the managed frontmatter markers — re-runs will not overwrite them.

### Custom roles (Step 5)

For each new role the user defines, blueprint generates a `description` derived from the user-provided responsibility text and wraps the entire frontmatter in `# scaffold:begin/end managed frontmatter` markers, same as default roles.

### Pending layer

The pending layer creates the same five agent files. Each gets the same frontmatter treatment — full frontmatter with managed markers, body content set to role defaults (no customisation prompt in pending mode).

## What does NOT change

- Step 1–6, Step 8, Step 9 — unchanged
- Existing managed section logic for HTML comment blocks — unchanged
- Re-run behaviour for all non-agent files — unchanged
- Body content of role files — unchanged
