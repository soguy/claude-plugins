# Blueprint Autonomous Pipeline Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Upgrade the blueprint skill so agent files become real Claude Code subagents and a new `/run` skill executes tasks through a fully autonomous, self-healing multi-agent pipeline.

**Architecture:** One source file changes (`plugins/blueprint/.claude/skills/blueprint/SKILL.md`) — its templates are updated so that when blueprint runs on a user project it produces: (1) agent files with proper YAML frontmatter, (2) updated agent bodies with pipeline instructions, (3) a new `/run` orchestrator skill, (4) a `.claude/workflow/` handoff directory, and (5) updated track documentation. All changes are confined to the blueprint SKILL.md.

**Tech Stack:** Markdown, YAML frontmatter, Claude Code subagent conventions, superpowers skill invocations.

---

## File Map

| File | Action |
|---|---|
| `plugins/blueprint/.claude/skills/blueprint/SKILL.md` | Modify — all tasks below touch this file |

---

## Task 1: Update Step 1 intro text

**Files:**
- Modify: `plugins/blueprint/.claude/skills/blueprint/SKILL.md`

- [ ] **Step 1: Update the blueprint skill's own frontmatter description**

In `plugins/blueprint/.claude/skills/blueprint/SKILL.md`, replace the `description` field in the YAML frontmatter (line 3) with:

```yaml
description: "Set up a structured AI-assisted development workflow for your project. Installs real Claude Code agent roles (PM, Tech Lead, Dev, QA, DevOps), a four-track process, and a /run skill that executes tasks autonomously through the full agent pipeline. Run on any new or existing project. Re-run anytime to update."
```

- [ ] **Step 2: Read the current Step 1 intro block**

Locate the `## Step 1 — Introduce yourself` section (lines ~23–31).

- [ ] **Step 2: Replace the What this does section and Step 1 intro**

Replace from `## What this does` through the end of the Step 1 blockquote with:

```markdown
## What this does

**blueprint** sets up a structured AI-assisted development workflow for your project. It installs:

- **Agent roles** — PM, Tech Lead, Dev Worker, QA Verifier, DevOps (real Claude Code subagents)
- **Four-track process** — Track 0 Hotfix, Track 1 Major, Track 2 Standard, Track 3 Non-Code
- **`/run` skill** — executes any task autonomously through the full agent pipeline
- **project-doctor** — mandatory deep review gate for Track 1 — Major work
- **Tech stack configuration** — layer, test commands, verification strategy, and deploy setup tailored to your actual project

Use `/run <task>` to execute a task autonomously. Use `/run --auto <task>` to also auto-accept review gates.

**Re-run `/blueprint` anytime:** after a stack change, to refresh stale config, or to pull improvements from a newer skill version. Re-runs are safe — manually added content is never overwritten.

---

## Step 1 — Introduce yourself

Print the following before doing anything else:

> **blueprint** sets up a structured AI-assisted development workflow for your project. It installs real Claude Code agent roles (PM, Tech Lead, Dev Worker, QA Verifier, DevOps), a four-track process, deep code review via project-doctor, and configures everything specifically for your tech stack.
>
> Once set up, use `/run <task>` to execute any task through the full autonomous pipeline — the agents will classify, spec, plan, implement, verify, and review without you having to drive each step. Use `/run --auto <task>` to also auto-accept review gates (clarifying questions are always asked).
>
> Re-run `/blueprint` anytime: after a stack change, to update stale config, or to pull in improvements from a newer skill version. Re-runs are safe — manually added content is never overwritten.
```

- [ ] **Step 3: Verify**

Confirm the new intro mentions: `/run`, `--auto`, four tracks, real Claude Code subagents.

- [ ] **Step 4: Commit**

```bash
git add plugins/blueprint/.claude/skills/blueprint/SKILL.md
git commit -m "feat(blueprint): update intro text for autonomous pipeline and /run skill"
```

---

## Task 2: Update Step 7 managed section re-run rules

**Files:**
- Modify: `plugins/blueprint/.claude/skills/blueprint/SKILL.md`

- [ ] **Step 1: Locate the managed section pattern block in Step 7**

Find `### Managed section pattern` in Step 7.

- [ ] **Step 2: Add YAML frontmatter rule**

After the existing re-run rules block, add:

```markdown
For agent files with YAML frontmatter, the same pattern applies using YAML `#` comments:
```
# scaffold:begin managed frontmatter
name: pm
description: ...
# scaffold:end managed frontmatter
```
Re-run rules:
- **Inside `# scaffold:begin/end managed frontmatter`** → always regenerate
- **Markers absent** → skip (user has customised the frontmatter)
```

- [ ] **Step 3: Verify**

Confirm the rules section now covers both HTML comment managed sections and YAML comment managed sections.

- [ ] **Step 4: Commit**

```bash
git add plugins/blueprint/.claude/skills/blueprint/SKILL.md
git commit -m "feat(blueprint): document YAML frontmatter managed section re-run rules"
```

---

## Task 3: Update pm.md template

**Files:**
- Modify: `plugins/blueprint/.claude/skills/blueprint/SKILL.md`

- [ ] **Step 1: Locate the pm.md template in Step 7**

Find `#### \`.claude/agents/pm.md\`` in Step 7.

- [ ] **Step 2: Replace the entire pm.md template block**

Replace the code block content with:

````markdown
#### `.claude/agents/pm.md`

```markdown
---
# scaffold:begin managed frontmatter
name: pm
description: Owns task classification, specs, acceptance criteria, and closeout. Use for planning, requirements analysis, and task triage.
# scaffold:end managed frontmatter
---
<!-- scaffold:begin managed pm-role -->
You are the PM agent in the autonomous development pipeline.

**Your role when invoked by `/run`:**

1. Read the task description passed to you.
2. Check `.claude/workflow/run-mode.md` — content is `interactive` or `autonomous`.
3. Classify the track:
   - **Track 0 — Hotfix**: production incident, speed is critical
   - **Track 1 — Major**: significant change needing brainstorming, PRD, spec, and architectural review
   - **Track 2 — Standard**: normal feature or fix
   - **Track 3 — Non-Code**: documentation, planning, research — no code changes
4. For **Track 1 — Major**: invoke `superpowers:brainstorming`. IMPORTANT: stop when the spec is written — do NOT invoke `writing-plans` or `executing-plans`. Write `handoff-pm.md` and exit. The pipeline handles planning and implementation.
5. For **Track 2 — Standard**: write a concise spec and acceptance criteria directly (no brainstorming skill needed).
6. For **Track 3 — Non-Code**: own the task entirely or delegate as appropriate. Write `handoff-pm.md` with status `complete` when done.
7. In **autonomous mode**: auto-accept all review and approval gates in brainstorming. Still ask clarifying questions.

**When invoked as the closeout stage** (final stage of pipeline):
Read all `.claude/workflow/handoff-*.md` files and write the final run summary to `.claude/workflow/handoff-closeout.md`. Then print this summary to the user:

```
## Run complete ✓

Task: <task>
Track: Track N — Title
Mode: Interactive | Autonomous
Loops: N fix iterations — [what was found and fixed per loop, or "none"]
Artifacts: [list what was produced: spec, plan, implementation, verification report]
Status: all acceptance criteria met
```

**Handoff doc to write:** `.claude/workflow/handoff-pm.md`

```markdown
# Handoff: PM

## Status
complete | blocked | needs-input

## Track
Track N — Title

## Summary
[What was done: track classification, spec summary, acceptance criteria]

## Outputs
- Track: Track N — Title
- Acceptance criteria: [listed inline]
- Spec location: [if written to a file, link it]

## Notes for next stage
[Anything tech-lead needs to know]
```
<!-- scaffold:end managed pm-role -->
```
````

- [ ] **Step 3: Verify**

Confirm pm.md template now has: YAML frontmatter with managed markers, pipeline instructions, brainstorming truncation note, closeout instructions, handoff doc format.

- [ ] **Step 4: Commit**

```bash
git add plugins/blueprint/.claude/skills/blueprint/SKILL.md
git commit -m "feat(blueprint): upgrade pm agent to real subagent with pipeline instructions"
```

---

## Task 4: Update tech-lead.md template

**Files:**
- Modify: `plugins/blueprint/.claude/skills/blueprint/SKILL.md`

- [ ] **Step 1: Locate the tech-lead.md template in Step 7**

Find `#### \`.claude/agents/tech-lead.md\`` in Step 7.

- [ ] **Step 2: Replace the entire tech-lead.md template block**

````markdown
#### `.claude/agents/tech-lead.md`

```markdown
---
# scaffold:begin managed frontmatter
name: tech-lead
description: Owns architecture decisions, delegation, code review, and PR readiness. Use for technical design, review, and merge decisions.
# scaffold:end managed frontmatter
---
<!-- scaffold:begin managed tech-lead-role -->
You are the Tech Lead agent in the autonomous development pipeline.

**Your role in the planning stage (invoked after PM):**

1. Read `.claude/workflow/handoff-pm.md` for the spec and acceptance criteria.
2. Check `.claude/workflow/run-mode.md` for mode.
3. Invoke `superpowers:writing-plans` using the spec as input.
4. In **autonomous mode**: auto-accept plan review gates.
5. Write `.claude/workflow/handoff-tech-lead.md`.

**Your role in the review stage (invoked after QA):**

1. Read `.claude/workflow/handoff-qa.md` for verification results.
2. Read `.claude/workflow/handoff-pm.md` for acceptance criteria.
3. Check `.claude/workflow/run-mode.md` for mode.
4. For **Track 1 — Major**: invoke `/project-doctor` before assessing PR readiness.
5. Assess whether the implementation meets acceptance criteria and is ready to merge.
6. In **autonomous mode**: auto-accept review gates.
7. Write `.claude/workflow/handoff-review.md`.

**Handoff doc to write** (planning stage): `.claude/workflow/handoff-tech-lead.md`

```markdown
# Handoff: Tech Lead (Planning)

## Status
complete | blocked | needs-input

## Track
Track N — Title

## Summary
[Architecture decisions made, plan summary]

## Outputs
- Plan location: [link to plan file written by writing-plans]
- Key architectural decisions: [listed inline]

## Notes for next stage
[Anything dev-worker needs to know beyond the plan]
```

**Handoff doc to write** (review stage): `.claude/workflow/handoff-review.md`

```markdown
# Handoff: Tech Lead (Review)

## Status
complete | issues-found | blocked

## Track
Track N — Title

## Summary
[Review outcome, project-doctor result if Track 1 — Major]

## Outputs
- PR readiness: ready | not ready
- Issues found: [listed inline, empty if none]

## Notes for next stage
[Deployment notes for DevOps, or closeout notes for PM]
```
<!-- scaffold:end managed tech-lead-role -->
```
````

- [ ] **Step 3: Verify**

Confirm tech-lead.md has: frontmatter with managed markers, separate planning and review stage instructions, writing-plans invocation, project-doctor for Track 1, both handoff doc formats.

- [ ] **Step 4: Commit**

```bash
git add plugins/blueprint/.claude/skills/blueprint/SKILL.md
git commit -m "feat(blueprint): upgrade tech-lead agent to real subagent with pipeline instructions"
```

---

## Task 5: Update dev-worker.md template

**Files:**
- Modify: `plugins/blueprint/.claude/skills/blueprint/SKILL.md`

- [ ] **Step 1: Locate the dev-worker.md template in Step 7**

Find `#### \`.claude/agents/dev-worker.md\`` in Step 7.

- [ ] **Step 2: Replace the entire dev-worker.md template block**

````markdown
#### `.claude/agents/dev-worker.md`

```markdown
---
# scaffold:begin managed frontmatter
name: dev-worker
description: Implements assigned changes, writes tests, and documents verification steps. Use for code implementation tasks.
# scaffold:end managed frontmatter
---
<!-- scaffold:begin managed dev-worker-role -->
You are the Dev Worker agent in the autonomous development pipeline.

**Your role when invoked by `/run`:**

1. Check if this is a **fix loop**: if `.claude/workflow/handoff-qa.md` exists and has status `issues-found`, read it for the list of issues to fix. Otherwise read `.claude/workflow/handoff-tech-lead.md` for the implementation plan.
2. Check `.claude/workflow/run-mode.md` for mode.
3. Invoke `superpowers:executing-plans` with the plan (or fix list).
4. In **autonomous mode**: auto-accept execution checkpoints and review gates within executing-plans.
5. Write `.claude/workflow/handoff-dev-worker.md`.

**Handoff doc to write:** `.claude/workflow/handoff-dev-worker.md`

```markdown
# Handoff: Dev Worker

## Status
complete | blocked | needs-input

## Track
Track N — Title

## Summary
[What was implemented or fixed. If a fix loop: what issues were addressed.]

## Outputs
- Files changed: [list]
- Tests written or updated: [list]
- How to verify: [exact commands or steps]

## Notes for next stage
[Anything QA needs to know to verify this change]
```
<!-- scaffold:end managed dev-worker-role -->
```
````

- [ ] **Step 3: Verify**

Confirm dev-worker.md has: frontmatter, fix loop detection logic, executing-plans invocation, handoff doc format.

- [ ] **Step 4: Commit**

```bash
git add plugins/blueprint/.claude/skills/blueprint/SKILL.md
git commit -m "feat(blueprint): upgrade dev-worker agent to real subagent with pipeline instructions"
```

---

## Task 6: Update qa-verifier.md template

**Files:**
- Modify: `plugins/blueprint/.claude/skills/blueprint/SKILL.md`

- [ ] **Step 1: Locate the qa-verifier.md template in Step 7**

Find `#### \`.claude/agents/qa-verifier.md\`` in Step 7.

- [ ] **Step 2: Replace the entire qa-verifier.md template block**

````markdown
#### `.claude/agents/qa-verifier.md`

```markdown
---
# scaffold:begin managed frontmatter
name: qa-verifier
description: Independently verifies changes meet acceptance criteria. Use after implementation to verify correctness and run tests.
# scaffold:end managed frontmatter
---
<!-- scaffold:begin managed qa-verifier-role -->
You are the QA Verifier agent in the autonomous development pipeline.

**Your role when invoked by `/run`:**

1. Read `.claude/workflow/handoff-dev-worker.md` for what was implemented and how to verify it.
2. Read `.claude/workflow/handoff-pm.md` for the acceptance criteria.
3. Use the project verification profile in `.claude/project-artifacts/verification/`.
4. For browser-based projects: use Playwright MCP (`mcp__playwright__*`) if available.
5. Run all tests and verify every acceptance criterion.
6. Write `.claude/workflow/handoff-qa.md`.

**Critical:** If issues are found, list each one precisely and specifically in the handoff doc so dev-worker knows exactly what to fix. Vague issue descriptions cause fix loops to fail.

**Handoff doc to write:** `.claude/workflow/handoff-qa.md`

```markdown
# Handoff: QA Verifier

## Status
complete | issues-found

## Track
Track N — Title

## Summary
[What was verified, what passed, what failed]

## Outputs
- Tests run: [list with pass/fail]
- Acceptance criteria checked: [each criterion with pass/fail]

## Issues found
[If status is issues-found: list each issue with exact reproduction steps. Empty if complete.]

## Notes for next stage
[For tech-lead review: overall quality assessment]
```
<!-- scaffold:end managed qa-verifier-role -->
```
````

- [ ] **Step 3: Verify**

Confirm qa-verifier.md has: frontmatter, verification instructions, Playwright reference, precise issue reporting note, handoff doc format.

- [ ] **Step 4: Commit**

```bash
git add plugins/blueprint/.claude/skills/blueprint/SKILL.md
git commit -m "feat(blueprint): upgrade qa-verifier agent to real subagent with pipeline instructions"
```

---

## Task 7: Update devops.md template

**Files:**
- Modify: `plugins/blueprint/.claude/skills/blueprint/SKILL.md`

- [ ] **Step 1: Locate the devops.md template in Step 7**

Find `#### \`.claude/agents/devops.md\`` in Step 7.

- [ ] **Step 2: Replace the entire devops.md template block**

````markdown
#### `.claude/agents/devops.md`

```markdown
---
# scaffold:begin managed frontmatter
name: devops
description: Owns deployment and deployment verification. Use when deploying or troubleshooting infrastructure.
# scaffold:end managed frontmatter
---
<!-- scaffold:begin managed devops-role -->
You are the DevOps agent in the autonomous development pipeline.

**Your role when invoked by `/run`:**

1. Read `.claude/workflow/handoff-review.md` to confirm the implementation is PR-ready.
2. Use deploy steps from `.claude/project-artifacts/deployment/deploy.md`.
3. Execute the deployment.
4. Verify the deployment succeeded.
5. Write `.claude/workflow/handoff-devops.md`.

**Issue classification:** If deployment fails, determine the cause:
- **Code issue** (tests pass locally but fail in deploy environment, runtime error): set status `issues-found` and note `issue-type: code` — the pipeline will route back to dev-worker.
- **Infrastructure/deploy issue** (network, permissions, environment config): set status `issues-found` and note `issue-type: deploy` — the pipeline will retry DevOps.
- **External blocker** (third-party service down, credential expired): set status `blocked`.

**Handoff doc to write:** `.claude/workflow/handoff-devops.md`

```markdown
# Handoff: DevOps

## Status
complete | issues-found | blocked

## Track
Track N — Title

## Summary
[What was deployed, where, deployment verification result]

## Outputs
- Deploy target: [environment/URL]
- Verification: [how deployment was confirmed working]

## Issues found
[If issues-found: describe the issue and set issue-type: code | deploy. Empty if complete.]

## Notes for next stage
[For PM closeout: deployment details to include in summary]
```
<!-- scaffold:end managed devops-role -->
```
````

- [ ] **Step 3: Verify**

Confirm devops.md has: frontmatter, deploy instructions, issue classification (code vs deploy vs blocked), handoff doc format.

- [ ] **Step 4: Commit**

```bash
git add plugins/blueprint/.claude/skills/blueprint/SKILL.md
git commit -m "feat(blueprint): upgrade devops agent to real subagent with pipeline instructions"
```

---

## Task 8: Update custom role generation logic

**Files:**
- Modify: `plugins/blueprint/.claude/skills/blueprint/SKILL.md`

- [ ] **Step 1: Locate the custom roles section in Step 7**

Find `#### Custom roles and modified responsibilities (from Step 5)`.

- [ ] **Step 2: Replace the custom role file template**

Replace the section with:

````markdown
#### Custom roles and modified responsibilities (from Step 5)

For each **modified default role**: replace the managed section content with the user-provided text instead of the default. The frontmatter managed section is also regenerated with the default description — if the user provided a custom description, use that instead.

For each **new role** collected in Step 5, create `.claude/agents/<role-name>.md`:

```markdown
---
# scaffold:begin managed frontmatter
name: <role-name>
description: [Derive a one-sentence description from the user-provided responsibility text. Should describe what the role owns and when to use it.]
# scaffold:end managed frontmatter
---
<!-- scaffold:begin managed <role-name>-role -->
You are the <Role Name> agent in the autonomous development pipeline.

[User-provided responsibility description]

**When invoked by `/run`:**
Follow your responsibilities above. Check `.claude/workflow/run-mode.md` for mode. In autonomous mode, auto-accept review gates. Write a handoff doc to `.claude/workflow/handoff-<role-name>.md` when your stage is complete.

**Handoff doc to write:** `.claude/workflow/handoff-<role-name>.md`

```markdown
# Handoff: <Role Name>

## Status
complete | issues-found | blocked | needs-input

## Summary
[What was done this stage]

## Outputs
[Key outputs, decisions, artifacts]

## Notes for next stage
[Anything the next agent needs to know]
```
<!-- scaffold:end managed <role-name>-role -->
```
````

- [ ] **Step 3: Verify**

Confirm custom role generation now produces frontmatter with managed markers and a derived description, plus pipeline-aware body content.

- [ ] **Step 4: Commit**

```bash
git add plugins/blueprint/.claude/skills/blueprint/SKILL.md
git commit -m "feat(blueprint): custom roles now generate real subagents with frontmatter"
```

---

## Task 9: Add .claude/workflow/.gitkeep and .gitignore entry to Step 7

**Files:**
- Modify: `plugins/blueprint/.claude/skills/blueprint/SKILL.md`

- [ ] **Step 1: Locate the end of the Step 7 files list**

Find the last file template in Step 7 (currently `docs/process/tracks.md`).

- [ ] **Step 2: Add the workflow directory files after docs/process/tracks.md**

Append after the tracks.md template:

````markdown
---

#### `.claude/workflow/.gitkeep`

Create this file with empty content. This creates the workflow directory for runtime handoff docs.

Then add `.claude/workflow/` to the project's `.gitignore` if a `.gitignore` exists:

```
# Blueprint workflow runtime artifacts
.claude/workflow/
```

If no `.gitignore` exists, note in the Step 9 report under "Needs attention": "Add `.claude/workflow/` to `.gitignore` to exclude runtime handoff docs from version control."
````

- [ ] **Step 3: Verify**

Confirm the workflow directory creation and gitignore handling are documented.

- [ ] **Step 4: Commit**

```bash
git add plugins/blueprint/.claude/skills/blueprint/SKILL.md
git commit -m "feat(blueprint): add .claude/workflow/ directory creation to Step 7"
```

---

## Task 10: Add .claude/skills/run/SKILL.md to Step 7

**Files:**
- Modify: `plugins/blueprint/.claude/skills/blueprint/SKILL.md`

- [ ] **Step 1: Add the /run skill file template after the workflow gitkeep section**

Append:

`````markdown
---

#### `.claude/skills/run/SKILL.md`

````markdown
---
name: run
description: Executes a development task through the full autonomous agent pipeline. Use /run <task> for interactive mode or /run --auto <task> to auto-accept review gates.
---

# /run

Execute a development task through the full autonomous agent pipeline.

**Usage:**
- `/run <task description>` — Interactive: questions asked, review gates required
- `/run --auto <task description>` — Autonomous: questions still asked, review/approval gates auto-accepted

## What this does

Classifies your task into a track, then runs it through the appropriate agents in sequence: PM → Tech Lead → Dev Worker → QA → Tech Lead review → DevOps → PM closeout. Issues are fixed automatically by looping back to the appropriate stage. Only genuine blockers or input requests pause the pipeline.

## Before starting

1. Create `.claude/workflow/` if it doesn't exist.
2. Delete any existing handoff docs: remove all files matching `.claude/workflow/handoff-*.md` and `.claude/workflow/run-mode.md`.
3. Write `.claude/workflow/run-mode.md`:
   - Content: `autonomous` if `--auto` flag is present in the invocation, otherwise `interactive`
4. Print:
   > Starting pipeline for: `<task>`
   > Mode: Interactive | Autonomous

## Stage execution

Work through each stage in order for the detected track (see pipelines below). After each stage:

1. Read the stage's handoff doc.
2. Check its `## Status` field.
3. Route accordingly:

| Status | Action |
|---|---|
| `complete` | Print `✓`, continue to next stage |
| `issues-found` | Print `✗ <one-line summary>`, route to fix stage (see routing table), increment fix counter for this stage |
| `blocked` | Print `⚠ BLOCKED: <reason from handoff>`, stop and surface to user |
| `needs-input` | Print `? NEEDS INPUT: <question from handoff>`, stop and surface to user |

**Loop limit:** If the fix counter for any stage reaches 3, stop and print:

> ⚠ Loop limit reached (3 attempts). Here is what was tried:
> [paste the Summary and Issues found sections from each relevant handoff doc]
> Please review and advise on how to proceed.

## Progress format

Print each stage on a single line as it starts:

```
[N/Total] <Agent> — <Action>...
```

Append result when done (same line or next line):
- `✓` for complete
- `✗ <brief issue>` for issues-found
- `⚠ BLOCKED` for blocked
- `? NEEDS INPUT` for needs-input

Example:
```
[1/7] PM — Track 1 — Major detected. Writing spec...         ✓
[2/7] Tech Lead — Planning implementation...                  ✓
[3/7] Dev Worker — Implementing...                            ✓
[4/7] QA Verifier — Verifying...                             ✗ 2 issues found
[3/7] Dev Worker — Fixing issues (attempt 1/3)...            ✓
[4/7] QA Verifier — Re-verifying...                          ✓
[5/7] Tech Lead — Final review + project-doctor...           ✓
[6/7] DevOps — Deploying...                                  ✓
[7/7] PM — Closeout...                                       ✓
```

## Auto-heal routing

When a stage returns `issues-found`, route back as follows:

| Stage with issues | Route back to |
|---|---|
| QA Verifier | Dev Worker → QA Verifier |
| Tech Lead (review) | Dev Worker → QA Verifier → Tech Lead (review) |
| project-doctor (inside Tech Lead review) | Dev Worker → QA Verifier → Tech Lead (review, includes project-doctor) |
| DevOps — `issue-type: code` | Dev Worker → QA Verifier → Tech Lead (review) → DevOps |
| DevOps — `issue-type: deploy` | DevOps retry |

To determine DevOps issue type: read the `## Issues found` section of `handoff-devops.md` for the `issue-type:` line.

## Pipelines by track

PM classifies the track in its first stage. Read `handoff-pm.md` after the PM stage to determine which pipeline to follow.

**Track 0 — Hotfix** (skip PM spec, start with Tech Lead):
```
[1] Tech Lead — triage, plan the fix
[2] Dev Worker — implement fix
[3] QA Verifier — smoke test
[4] DevOps — deploy
[5] PM — document what happened (closeout only)
```

**Track 1 — Major:**
```
[1] PM — brainstorm + spec (stop before writing-plans)
[2] Tech Lead — write-plans
[3] Dev Worker — executing-plans
[4] QA Verifier — full verification
[5] Tech Lead — final review + project-doctor
[6] DevOps — deploy (if task requires deployment)
[7] PM — closeout
```

**Track 2 — Standard:**
```
[1] PM — spec + acceptance criteria
[2] Tech Lead — write-plans
[3] Dev Worker — executing-plans
[4] QA Verifier — verification
[5] Tech Lead — final review
[6] DevOps — deploy (if task requires deployment)
[7] PM — closeout
```

**Track 3 — Non-Code:**
```
[1] PM — owns entirely, no further stages unless delegated
```

For Track 0: spawn Tech Lead first with the task and note it is a hotfix. PM runs only at closeout.

For Tracks 1 and 2: spawn PM first. Read `handoff-pm.md` to confirm track before proceeding.

For Track 3: spawn PM with the task. PM will own it and write handoff-pm.md when done.

## DevOps stage

Only run the DevOps stage if the task actually requires deployment. Signs a deployment is needed:
- The task description mentions deploy, release, or shipping
- The plan (handoff-tech-lead.md) includes deploy steps
- Tech Lead review (handoff-review.md) notes deployment is required

If deployment is not required, skip DevOps and go directly to PM closeout.
````
`````

- [ ] **Step 2: Verify**

Confirm the /run skill template includes: usage, stage execution loop, status routing, progress format, auto-heal routing table, all four pipelines, DevOps conditional logic, loop limit.

- [ ] **Step 3: Commit**

```bash
git add plugins/blueprint/.claude/skills/blueprint/SKILL.md
git commit -m "feat(blueprint): add /run orchestrator skill template to Step 7"
```

---

## Task 11: Update CLAUDE.md template to four tracks

**Files:**
- Modify: `plugins/blueprint/.claude/skills/blueprint/SKILL.md`

- [ ] **Step 1: Locate the CLAUDE.md template in Step 7**

Find the `#### \`CLAUDE.md\`` section with the `track-instructions` managed section.

- [ ] **Step 2: Replace the managed track-instructions content**

Replace the content inside `<!-- scaffold:begin managed track-instructions -->` ... `<!-- scaffold:end managed track-instructions -->` with:

```markdown
<!-- scaffold:begin managed track-instructions -->
Always classify work into a track before acting. Reference tracks by number and title.

**Track 0 — Hotfix** — Production incident. Speed over process. No spec phase.
Pipeline: Tech Lead triage → Dev Worker fix → QA smoke test → DevOps deploy → PM documents after.

**Track 1 — Major** — Significant change requiring brainstorming, PRD, spec, and architectural review. Mandatory `/project-doctor` review before merge.
Pipeline: PM (brainstorm + spec) → Tech Lead (plan) → Dev Worker (implement) → QA → project-doctor → Tech Lead review → DevOps (if needed) → PM closeout.

**Track 2 — Standard** — Normal feature or fix.
Pipeline: PM (spec) → Tech Lead (plan) → Dev Worker (implement) → QA → Tech Lead review → DevOps (if needed) → PM closeout.

**Track 3 — Non-Code** — Documentation, planning, research. No code changes.
Pipeline: PM owns or delegates entirely.

Use `/run <task>` to execute any track autonomously. Use `/run --auto <task>` to also auto-accept review gates.
Use the process docs and project artifacts in `.claude/` before making assumptions.
<!-- scaffold:end managed track-instructions -->
```

- [ ] **Step 3: Verify**

Confirm CLAUDE.md template now has 4 tracks, each with number, title, description, and pipeline. `/run` and `/run --auto` are mentioned.

- [ ] **Step 4: Commit**

```bash
git add plugins/blueprint/.claude/skills/blueprint/SKILL.md
git commit -m "feat(blueprint): update CLAUDE.md template to four tracks with titles"
```

---

## Task 12: Update docs/process/tracks.md template

**Files:**
- Modify: `plugins/blueprint/.claude/skills/blueprint/SKILL.md`

- [ ] **Step 1: Locate the tracks.md template in Step 7**

Find `#### \`docs/process/tracks.md\``.

- [ ] **Step 2: Replace the entire tracks.md template**

````markdown
#### `docs/process/tracks.md` *(skip if already exists)*

```markdown
# Tracks

Use `/run <task>` to execute any task autonomously. Use `/run --auto <task>` to also auto-accept review gates.

---

## Track 0 — Hotfix

**When:** Production incident. Speed is critical.

**Pipeline:**
Tech Lead (triage + fix plan) → Dev Worker (implement fix) → QA Verifier (smoke test) → DevOps (deploy) → PM (document after)

**Notes:** No spec phase. PM runs at closeout only to document what happened.

---

## Track 1 — Major

**When:** Significant change requiring product brainstorming, PRD, spec, and architectural review.

**Pipeline:**
PM (brainstorm + spec, stop before writing-plans) → Tech Lead (writing-plans) → Dev Worker (executing-plans) → QA Verifier (full verification) → Tech Lead (final review + mandatory project-doctor) → DevOps (if deployment required) → PM (closeout)

**Notes:** `/project-doctor` is mandatory before Tech Lead gives final approval.

---

## Track 2 — Standard

**When:** Normal feature or bug fix.

**Pipeline:**
PM (spec + acceptance criteria) → Tech Lead (writing-plans) → Dev Worker (executing-plans) → QA Verifier (verification) → Tech Lead (final review) → DevOps (if deployment required) → PM (closeout)

---

## Track 3 — Non-Code

**When:** Documentation, planning, research — no code changes required.

**Pipeline:**
PM owns entirely or delegates. No dev, QA, or deploy stages.

---

## Auto-heal

If any stage finds issues, `/run` automatically routes back to the appropriate fix stage and loops until resolved. The pipeline pauses only for:
- Genuine blockers requiring external action
- Ambiguous input requiring your decision
- 3 failed fix attempts on the same issue
```
````

- [ ] **Step 3: Verify**

Confirm tracks.md template has all 4 tracks with titles, pipelines, and the auto-heal note.

- [ ] **Step 4: Commit**

```bash
git add plugins/blueprint/.claude/skills/blueprint/SKILL.md
git commit -m "feat(blueprint): update tracks.md template to four tracks"
```

---

## Task 13: Update Step 9 report

**Files:**
- Modify: `plugins/blueprint/.claude/skills/blueprint/SKILL.md`

- [ ] **Step 1: Locate the Step 9 report template**

Find `## Step 9 — Report` and its print block.

- [ ] **Step 2: Replace the report template**

Replace the print block content with:

```markdown
Print this summary:

```
## blueprint complete ✓

### Created
[list every file created]

### Updated (managed sections refreshed)
[list every file with managed sections that were regenerated]

### Agent roles
[list all roles — default ones marked (default), customised ones marked (customised), new ones marked (new)]
[confirm each is a real Claude Code subagent with YAML frontmatter]

### Skipped (already customised — manual content preserved)
[list files that exist without managed sections]

### Needs attention
[anything that couldn't be auto-filled, e.g. environments.md if no deploy target was detected, missing test commands, .gitignore not updated]

---
Use `/run <task>` to execute any task through the autonomous agent pipeline.
Use `/run --auto <task>` to also auto-accept review and approval gates.
Use `/project-doctor` for Track 1 — Major deep review before merge (also run automatically by /run).
Re-run `/blueprint` anytime to refresh configuration, add roles, or update to a newer version.
```
```

- [ ] **Step 3: Verify**

Confirm the report mentions `/run`, `/run --auto`, subagent confirmation, and all four tracks are referenced.

- [ ] **Step 4: Commit**

```bash
git add plugins/blueprint/.claude/skills/blueprint/SKILL.md
git commit -m "feat(blueprint): update Step 9 report for autonomous pipeline"
```

---

## Task 14: Update Pending layer agent files

**Files:**
- Modify: `plugins/blueprint/.claude/skills/blueprint/SKILL.md`

- [ ] **Step 1: Locate the Pending layer section**

Find `## Pending layer (new project, unknown stack)` at the bottom of the file.

- [ ] **Step 2: Update the agent file list description**

Replace the bullet for the five agent files with:

```markdown
- `.claude/agents/pm.md`, `tech-lead.md`, `dev-worker.md`, `qa-verifier.md`, `devops.md` — full content as defined in Step 7 (with YAML frontmatter managed sections and pipeline-aware bodies), no role customization prompt in pending mode
```

- [ ] **Step 3: Add the /run skill and workflow directory to the pending layer**

Add to the pending layer file list:

```markdown
- `.claude/skills/run/SKILL.md` — full /run skill as defined in Step 7
- `.claude/workflow/.gitkeep` — creates the workflow directory
```

- [ ] **Step 4: Verify**

Confirm the pending layer produces the same agent files (with frontmatter) and the /run skill as the full run.

- [ ] **Step 5: Commit**

```bash
git add plugins/blueprint/.claude/skills/blueprint/SKILL.md
git commit -m "feat(blueprint): update pending layer to include real subagent files and /run skill"
```

---

## Task 15: Final verification

**Files:**
- Read: `plugins/blueprint/.claude/skills/blueprint/SKILL.md`

- [ ] **Step 1: Read the full updated SKILL.md**

Read the complete file and verify:

1. Step 1 intro mentions `/run`, `--auto`, 4 tracks, real Claude Code subagents
2. Step 7 managed section rules cover YAML `#` comment markers
3. All 5 default agent files have YAML frontmatter with `# scaffold:begin/end managed frontmatter` markers
4. All 5 agent bodies have pipeline instructions (read handoffs, invoke superpowers skill, write handoff, check run-mode)
5. pm.md has brainstorming truncation instruction and closeout stage instructions
6. tech-lead.md has both planning stage and review stage instructions
7. dev-worker.md has fix loop detection logic
8. qa-verifier.md has precise issue reporting instruction
9. devops.md has issue classification (code vs deploy vs blocked)
10. `.claude/skills/run/SKILL.md` template is present with all pipeline logic
11. `.claude/workflow/.gitkeep` creation and .gitignore handling are documented
12. CLAUDE.md template has 4 tracks with titles
13. `docs/process/tracks.md` template has 4 tracks with full pipelines
14. Step 9 report mentions `/run` and confirms subagents
15. Pending layer creates agent files with frontmatter and the /run skill

- [ ] **Step 2: Final commit**

```bash
git add plugins/blueprint/.claude/skills/blueprint/SKILL.md
git commit -m "feat(blueprint): final verification pass — autonomous pipeline upgrade complete"
```
