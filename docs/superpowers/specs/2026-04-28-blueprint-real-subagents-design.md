# Design: Blueprint — Autonomous Agent Pipeline with Real Claude Code Subagents

**Date:** 2026-04-28  
**Status:** Approved

---

## Problem

The blueprint skill creates `.claude/agents/*.md` files that:
1. Are not recognised by Claude Code as real subagents (missing YAML frontmatter)
2. Are purely role documentation — agents don't chain into an autonomous pipeline
3. Require the user to manually invoke each role step by step

There is also no opt-in trigger — any user in the project could inadvertently activate the workflow.

---

## Goals

1. Make every blueprint agent file a real Claude Code subagent (YAML frontmatter)
2. Add a `/run` trigger skill that executes the full pipeline autonomously
3. Keep workflow opt-in — other users unaffected unless they invoke `/run`
4. Integrate superpowers skills (`brainstorming`, `writing-plans`, `executing-plans`) as the engine inside each agent stage
5. Auto-heal: pipeline loops on issues without user intervention unless truly blocked
6. Full transparency: user sees live progress and a final summary of everything that happened

---

## Scope

One source file changes (`SKILL.md`). The generated files it writes into user projects are updated or newly added:

| File | Change |
|---|---|
| `plugins/blueprint/.claude/skills/blueprint/SKILL.md` | Agent templates, intro text, tracks, report, pending layer |
| *(generated)* `CLAUDE.md` | 4 tracks with titles |
| *(generated)* `docs/process/tracks.md` | 4 tracks with full stage sequences |
| *(new, generated)* `.claude/skills/run/SKILL.md` | The `/run` orchestrator skill |
| *(new, generated)* `.claude/workflow/.gitkeep` | Runtime handoff directory |

---

## Design

### 1. Naming

| Skill | Purpose |
|---|---|
| `/blueprint` | Sets up and updates the AI-assisted development framework |
| `/run <task>` | Executes a task through the full autonomous agent pipeline |

---

### 2. Tracks

Four tracks, always referenced with both number and title:

| Track | Title | When | Pipeline |
|---|---|---|---|
| Track 0 | Hotfix | Production incident — speed over process | Tech Lead triage → Dev Worker fix → QA smoke test → DevOps deploy → PM documents after |
| Track 1 | Major | Significant change requiring product brainstorming, PRD, spec, and architectural review | PM (brainstorm + spec) → Tech Lead (plan) → Dev Worker (implement) → QA → project-doctor → Tech Lead review → DevOps → PM closeout |
| Track 2 | Standard | Normal feature or fix | PM (spec) → Tech Lead (plan) → Dev Worker (implement) → QA → Tech Lead review → DevOps (if needed) → PM closeout |
| Track 3 | Non-Code | Documentation, planning, research — no code changes | PM owns or delegates entirely |

Track is classified by the PM agent at the start of every `/run`. For Track 0, PM is skipped and Tech Lead begins immediately.

---

### 3. Agent frontmatter

Each `.claude/agents/*.md` file gets YAML frontmatter using YAML `#` comments as managed-section markers:

```markdown
---
# scaffold:begin managed frontmatter
name: pm
description: Owns task classification, specs, acceptance criteria, and closeout. Use for planning, requirements analysis, and task triage.
# scaffold:end managed frontmatter
---
<!-- scaffold:begin managed pm-role -->
...
<!-- scaffold:end managed pm-role -->
```

**Re-run contract:**
- `# scaffold:begin/end managed frontmatter` present → regenerate on re-run
- Markers absent → skip (user has customised)
- `tools` field omitted → full tool access (Claude Code default)
- `model` field omitted → inherits from parent session

**Role descriptions:**

| File | `name` | `description` |
|---|---|---|
| `pm.md` | `pm` | Owns task classification, specs, acceptance criteria, and closeout. Use for planning, requirements analysis, and task triage. |
| `tech-lead.md` | `tech-lead` | Owns architecture decisions, delegation, code review, and PR readiness. Use for technical design, review, and merge decisions. |
| `dev-worker.md` | `dev-worker` | Implements assigned changes, writes tests, and documents verification steps. Use for code implementation tasks. |
| `qa-verifier.md` | `qa-verifier` | Independently verifies changes meet acceptance criteria. Use after implementation to verify correctness and run tests. |
| `devops.md` | `devops` | Owns deployment and deployment verification. Use when deploying or troubleshooting infrastructure. |

Custom roles (defined in Step 5): blueprint generates a description derived from the user-provided responsibility text, wrapped in the same managed frontmatter markers.

---

### 4. Agent body updates

Each agent's body is updated with specific pipeline instructions. Each agent:
1. Reads its input handoff doc(s) from `.claude/workflow/`
2. Checks `.claude/workflow/run-mode.md` for auto-approve behaviour
3. Invokes its superpowers skill
4. Writes its output handoff doc to `.claude/workflow/`

| Agent | Superpowers skill | Input | Output |
|---|---|---|---|
| `pm` | `brainstorming` — truncated: stop before invoking `writing-plans` | Task from `/run` | `handoff-pm.md` |
| `tech-lead` (planning) | `writing-plans` — takes spec from PM handoff as input | `handoff-pm.md` | `handoff-tech-lead.md` |
| `dev-worker` | `executing-plans` — takes plan from Tech Lead handoff | `handoff-tech-lead.md` (first run) or `handoff-qa.md` (fix loop) | `handoff-dev-worker.md` |
| `qa-verifier` | Own verification logic (tests, Playwright, etc.) | `handoff-dev-worker.md` | `handoff-qa.md` |
| `tech-lead` (review) | `project-doctor` if Track 1 — Major | `handoff-qa.md` | `handoff-review.md` |
| `devops` | Deploy commands per project config | `handoff-review.md` | `handoff-devops.md` |
| `pm` (closeout) | — | All handoff docs | `handoff-closeout.md` + final summary printed to user |

**Brainstorming truncation:** PM agent body includes explicit instruction: "When brainstorming produces a spec, stop there. Do not invoke `writing-plans` or `executing-plans`. Write `handoff-pm.md` with the spec and acceptance criteria, then exit."

**Auto-approve in agents:** Each agent body includes: "Check `.claude/workflow/run-mode.md`. If content is `autonomous`, auto-accept all review and approval gates within your superpowers skill invocation. Questions still require user input."

---

### 5. Handoff document format

All handoff docs written to `.claude/workflow/`. Directory is gitignored (runtime artifacts).

```markdown
# Handoff: <Stage Name>

## Status
complete | issues-found | blocked | needs-input

## Track
Track N — Title

## Summary
One paragraph of what was done this stage.

## Outputs
- Key decisions, files changed, test results, acceptance criteria — listed inline
- Links to any artifacts written (spec, plan, etc.)

## Notes for next stage
Anything the next agent needs to know that isn't obvious from the outputs.
```

---

### 6. The `/run` skill

Blueprint writes `.claude/skills/run/SKILL.md` to the project.

**Invocation:**
- `/run <task>` — Interactive: questions asked, reviews required
- `/run --auto <task>` — Autonomous: questions still asked, review/approval gates auto-accepted

**On start:**
1. Create `.claude/workflow/` if missing
2. Clear handoff docs from any previous run
3. Write `.claude/workflow/run-mode.md`: `interactive` or `autonomous`
4. Print pipeline header

**Stage execution loop:**

```
For each stage in pipeline (determined by track):
  1. Print: [N/Total] Agent — Action...
  2. Spawn agent via Agent tool
  3. Agent writes handoff doc
  4. Read handoff doc status:
     - complete      → print ✓, continue to next stage
     - issues-found  → print ✗, auto-route to fix stage, increment fix counter
     - blocked       → print ⚠, pause, surface to user
     - needs-input   → print ?, pause, surface to user
  5. If fix counter reaches 3 → pause, surface to user with full attempt history
```

**Live progress format:**
```
[1/7] PM — Track 1 — Major detected. Writing spec...        ✓
[2/7] Tech Lead — Planning implementation...                 ✓
[3/7] Dev Worker — Implementing...                           ✓
[4/7] QA Verifier — Verifying...                            ✗ 2 issues found
[3/7] Dev Worker — Fixing issues (attempt 1/3)...           ✓
[4/7] QA Verifier — Re-verifying...                         ✓
[5/7] Tech Lead — Final review + project-doctor...          ✓
[6/7] DevOps — Deploying...                                 ✓
[7/7] PM — Closeout...                                      ✓
```

**Auto-heal routing:**

| Stage that finds issues | Routes back to |
|---|---|
| QA Verifier | Dev Worker |
| Tech Lead review | Dev Worker → QA → Tech Lead |
| project-doctor | Dev Worker → QA → Tech Lead → project-doctor |
| DevOps (code issue) | Dev Worker → QA → Tech Lead → DevOps |
| DevOps (deploy issue) | DevOps retry |

**Final summary** (printed by PM closeout):
```
## Run complete ✓

Task: <task>
Track: Track N — Title
Loops: N fix iterations (if any — what was found, what was fixed)
Artifacts: spec, plan, implementation, verification report
Status: all acceptance criteria met
```

---

### 7. Run mode and auto-approve scope

`--auto` flag auto-accepts review/approval gates **only**. It never skips:
- Clarifying questions (brainstorming still asks these)
- `blocked` or `needs-input` status (always surfaces to user)
- `issues-found` status (always triggers auto-heal loop, never silently passed)
- Loop limit reached (always surfaces to user)

---

### 8. Blueprint SKILL.md changes

| Section | Change |
|---|---|
| Step 1 — Intro text | Updated: explains `/run`, autonomous pipeline, `--auto` flag, 4 tracks |
| Step 7 — Agent templates | Frontmatter added (managed markers) + body updated per Section 4 |
| Step 7 — New files | `.claude/skills/run/SKILL.md` and `.claude/workflow/.gitkeep` added |
| Step 7 — `CLAUDE.md` template | 4 tracks with titles and stage sequences |
| Step 7 — `docs/process/tracks.md` | 4 tracks with full stage sequences |
| Step 9 — Report | Updated: lists `/run` as execution trigger, explains `--auto` |
| Pending layer | Same frontmatter treatment for all 5 default agent files |

---

### 9. What does NOT change

- Steps 2–6 (detect mode, dependencies, read project, configure roles, generate layer)
- Step 8 (project-doctor install check)
- Managed section re-run contract for all non-frontmatter content
- All non-agent generated files (testing, verification, deployment, review artifacts)
