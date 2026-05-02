---
name: blueprint
description: "Set up a structured AI-assisted development workflow for your project. Installs real Claude Code agent roles (PM, Tech Lead, Dev, QA, DevOps), a four-track process, and a /run skill that executes tasks autonomously through the full agent pipeline. Run on any new or existing project. Re-run anytime to update."
---

# blueprint

## What this does

**blueprint** sets up a structured AI-assisted development workflow for your project. It installs:

- **Agent roles** — PM, Tech Lead, Dev, QA, DevOps (real Claude Code subagents)
- **Four-track process** — Track 0 — Hotfix (emergency fix), Track 1 — Major (brainstorm + spec + full review), Track 2 — Standard (lightweight brainstorm via `simple`, scoped review on changed files), Track 3 — Non-Code (docs, planning, research)
- **`/run` skill** — executes any task autonomously through the full agent pipeline
- **project-doctor** — full codebase audit for Track 1, scoped to changed files for Track 2
- **Tech stack configuration** — layer, test commands, verification strategy, and deploy setup tailored to your actual project

Use `/run <task>` to execute a task autonomously. Use `/run --auto <task>` to also auto-accept review gates.

Once set up, `/run` drives the full pipeline: classify → spec → plan → implement → verify → review — without you having to manage each step.

**Re-run `/blueprint` anytime:** after a stack change, to refresh stale config, or to pull improvements from a newer skill version. Re-runs are safe — manually added content is never overwritten.

---

## Step 1 — Introduce yourself

Print the following before doing anything else:

> **blueprint** sets up a structured AI-assisted development workflow for your project. It installs real Claude Code agent roles (PM, Tech Lead, Dev, QA, DevOps), a four-track process, deep code review via project-doctor, and configures everything specifically for your tech stack.
>
> Once set up, use `/run <task>` to execute any task through the full autonomous pipeline — the agents will classify, spec, plan, implement, verify, and review without you having to drive each step. Use `/run --auto <task>` to also auto-accept review gates (clarifying questions are always asked).
>
> Re-run `/blueprint` anytime: after a stack change, to update stale config, or to pull in improvements from a newer skill version. Re-runs are safe — manually added content is never overwritten.

---

## Step 2 — Detect mode

Check the project directory:

- **Re-run**: `.blueprint/` directory already exists → tell user: *"Re-running blueprint — updating managed sections, leaving manual content untouched."*
- **Existing project, first run**: Source files exist but no `.blueprint/`
- **New project, stack known**: User has told you the stack or it is obvious from files already present
- **New project, stack unknown**: Ask — *"What technology stack are you planning to use? (Say 'unknown' to set up the structural scaffold now and complete stack config later.)"*
  - If unknown → jump to the **Pending layer** section at the bottom of this skill, then stop.

---

## Step 3 — Detect and install dependencies

Read `~/.claude/plugins/installed_plugins.json`.

### superpowers
- Key containing `superpowers` exists → already installed, skip
- Missing → ask: *"superpowers is not installed. It provides the brainstorming, TDD, debugging, and planning workflows used by this process. Install at user-scope (recommended — available in all your projects) or project-scope (this project only)?"*
  - User-scope: run `/plugin install superpowers`
  - Project-scope: note it for manual install guidance in the report

### frontend-design
- Key containing `frontend-design` exists → skip
- Missing AND project has a UI/frontend component → ask: *"frontend-design is not installed. It provides structured UI design workflows. Install at user-scope or project-scope?"*
  - Install at chosen scope

### simple
- Key containing `simple` exists → skip
- Missing → ask: *"simple is not installed. It provides lightweight brainstorming for Track 2 — Standard tasks (quick intent discovery, trade-off proposals, and direction approval before specifying). Install at user-scope (recommended) or project-scope?"*
  - User-scope: run `/plugin install https://github.com/roin-orca/skills.git simple`
  - Project-scope: note it for manual install guidance in the report

### Playwright MCP
- Only ask if project is browser-based (has pages/, a frontend, or renders HTML)
- Key containing `playwright` exists in installed_plugins.json → skip
- Missing → ask: *"This looks like a browser-based project. Playwright MCP enables browser-based testing and visual verification. Install it?"*
  - If yes → guide user: `/plugin install playwright-mcp` or equivalent

---

## Step 4 — Read the project

Read these files if they exist (do not error if absent):

**Config files:** `package.json`, `astro.config.*`, `next.config.*`, `vite.config.*`, `nuxt.config.*`, `svelte.config.*`, `pyproject.toml`, `requirements.txt`, `setup.py`, `go.mod`, `Cargo.toml`, `Gemfile`, `composer.json`

**CI/deploy files:** `Dockerfile`, `.github/workflows/`, `Makefile`, `wrangler.toml`, `vercel.json`, `netlify.toml`

**Project docs:** `README.md`

**Source structure:** examine `src/` (or `app/`, `lib/`, `cmd/` etc.) for pages, API routes, components, server-side code, test files

Determine:

| Signal | What to look for |
|---|---|
| Stack name | Framework + output type e.g. `astro-static`, `nextjs`, `django-rest`, `go-api`, `react-vite` |
| Test commands | Scripts in `package.json`, `pytest`, `cargo test`, `go test`, `make test` |
| Browser-based? | Has pages/, renders HTML, has a dev server |
| Deploy target | `wrangler.toml` → Cloudflare, `vercel.json` → Vercel, `netlify.toml` → Netlify, `Dockerfile` → container, CI workflow → CI-based |
| Solo or team? | Check for `.github/CODEOWNERS`, team size signals in README |

Summarise your findings to the user in a short bullet list and ask for confirmation before writing any files.

---

## Step 5 — Configure roles

Present the default roles and offer customization before writing any files.

**Default roles:**

| Role | Default responsibilities |
|------|--------------------------|
| PM | Task classification, specs, acceptance criteria, closeout |
| Tech Lead | Architecture, delegation, review, PR readiness |
| Dev | Implementation, tests, verification docs |
| QA | Independent verification using project verification profile |
| DevOps | Deployment and deployment verification |

Check `.claude/agents/` for existing role files. If found, display the current managed section content so the user can see what is already configured.

Ask:

> *"Would you like to customize agent roles? You can update responsibilities for any default role, or add new ones. Enter your changes or say 'skip' to use defaults."*

Collect and record for use in Step 7:
- **Modified default roles** — updated responsibility text for any of the five default roles
- **New roles** — name (lowercase-hyphenated, becomes the filename) + responsibility description

On re-run: read existing managed sections and display them before asking, so the user sees current configuration. Content outside managed sections is never shown and never modified.

---

## Step 6 — Generate the layer

Create `.blueprint/layers/<stack-name>/` with four files tailored to this exact project.

**README.md** — one paragraph describing the stack and what makes it distinct

**testing-defaults.md** — the actual test/build commands for this stack. Not generic. Use the real commands detected in Step 4. Examples:
- Astro static: `npm run build`
- Next.js: `npm run build`, `npm run lint`
- Django: `python -m pytest`, `python -m mypy .`
- Go API: `go test ./...`, `go vet ./...`

**verification-defaults.md** — the appropriate QA mode. Examples:
- Browser-based with Playwright MCP: `Primary QA mode: Playwright MCP (mcp__playwright__*) — navigate, screenshot, snapshot, console check`
- Python API: `Primary QA mode: pytest + manual API endpoint verification`
- Static site: `Primary QA mode: build audit + Playwright MCP visual review`

**deployment-defaults.md** — the actual deploy steps for the detected deploy target. Examples:
- Cloudflare Pages: `npm run build → dist/ → auto-deploy on push to main via Cloudflare Pages`
- Vercel: `npm run build → auto-deploy on push via Vercel`
- Docker: `docker build → docker push → deploy to container host`
- Unknown: `[Document your deploy steps here]`

---

## Step 7 — Write all files

### Managed section pattern

For every file section that may be regenerated on re-run, wrap it:
```
<!-- scaffold:begin managed <name> -->
...generated content...
<!-- scaffold:end managed <name> -->
```

Re-run rules:
- **Inside managed sections** → always regenerate
- **Outside managed sections** → never touch
- **Missing files** → create
- **Files without managed sections that already exist** → skip, note in report

For agent files with YAML frontmatter, the same pattern applies using YAML `#` comments:

```yaml
# scaffold:begin managed frontmatter
name: pm
description: ...
# scaffold:end managed frontmatter
```

Re-run rules:
- **Inside `# scaffold:begin/end managed frontmatter`** → always regenerate
- **Markers absent** → skip (user has customised the frontmatter)

### Files to write

Write each file below. Substitute `[GENERATED: ...]` placeholders with content derived from Step 4 analysis.

---

#### `CLAUDE.md`

Append a single include line to the project's existing `CLAUDE.md` (create the file if it doesn't exist). Do NOT write track instructions or managed sections into CLAUDE.md itself.

```markdown
@.blueprint/BLUEPRINT.md
```

If the line `@.blueprint/BLUEPRINT.md` already exists in `CLAUDE.md`, skip — do not duplicate it.

---

#### `.blueprint/BLUEPRINT.md`

```markdown
# Blueprint — Project Operating Guide

Always classify work into a track before acting. Reference tracks by number and title.

**Track 0 — Hotfix** — Production incident. Speed over process. No spec phase.
Pipeline: Tech Lead triage → Dev fix → QA smoke test → Ship (if uncommitted) → DevOps deploy → PM documents after.

**Track 1 — Major** — Significant change requiring brainstorming, PRD, spec, and architectural review. Mandatory `/project-doctor` review before merge.
Pipeline: PM (brainstorm + spec) → Tech Lead (plan) → Dev (implement) → QA → Tech Lead review (incl. project-doctor) → PM closeout (scope validation) → Ship (if uncommitted) → DevOps (if needed).

**Track 2 — Standard** — Normal feature or fix. Lightweight brainstorming via `simple` skill (always interactive). Scoped review on changed files only.
Pipeline: PM (simple brainstorm + spec) → Tech Lead (plan if needed) → Dev (implement) → QA → Tech Lead review (scoped project-doctor) → PM closeout → Ship (if uncommitted) → DevOps (if needed).

**Track 3 — Non-Code** — Documentation, planning, research. No code changes.
Pipeline: PM owns or delegates entirely.

Use `/run <task>` to execute any track autonomously. Use `/run --auto <task>` to also auto-accept review gates.
Use the process docs and project artifacts in `.blueprint/` before making assumptions.
```

---

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
2. Check `.blueprint/workflow/run-mode.md` — content is `interactive` or `autonomous`.
3. Classify the track (Tracks 1–3 only — `/run` never invokes PM first for Track 0 — Hotfix):
   - **Track 1 — Major**: significant change needing brainstorming, PRD, spec, and architectural review
   - **Track 2 — Standard**: normal feature or fix
   - **Track 3 — Non-Code**: documentation, planning, research — no code changes

   **Classification heuristics:**
   | Signal | Track 1 | Track 2 |
   |--------|---------|---------|
   | New subsystem or module | yes | — |
   | Touches > 5 existing files | likely | — |
   | Crosses architectural boundaries | yes | — |
   | Single concern, clear scope | — | yes |
   | User said "quick", "small", or "simple" | — | yes |
   | Ambiguous scope | default to Track 1 | — |
4. For **Track 1 — Major**: invoke `superpowers:brainstorming`. If the task involves UI/UX work (new pages, components, visual changes, user flows), also invoke `frontend-design` to inform the spec with design decisions. IMPORTANT: stop when the spec is written — do NOT invoke `writing-plans` or `executing-plans`. In **autonomous mode**: auto-accept all review and approval gates in brainstorming; still ask clarifying questions. Write `.blueprint/workflow/handoff-pm.md` and exit. The pipeline handles planning and implementation.
5. For **Track 2 — Standard**: invoke `simple` for lightweight brainstorming (intent discovery, trade-offs, direction approval). **The `simple` brainstorm is always interactive — never auto-accept direction choices, even in autonomous mode.** Then write spec and acceptance criteria based on the approved direction. If the task involves UI/UX work, invoke `frontend-design` to inform the spec with design decisions before writing acceptance criteria. Also note in the handoff doc whether the change is small-scope (single concern, few files) — this determines whether Tech Lead plans or Dev implements directly. Write `.blueprint/workflow/handoff-pm.md`.
6. For **Track 3 — Non-Code**: own the task entirely or delegate as appropriate. Write `.blueprint/workflow/handoff-pm.md` with status `complete` when done.

**When invoked as the closeout stage** (final stage of pipeline — including Track 0 — Hotfix, where you run last):

1. Read all `.blueprint/workflow/handoff-*.md` files.
2. **Scope validation (skip for Track 0 — Hotfix, since no initial spec exists):** Compare the actual files changed (from `handoff-dev.md` and any fix-loop handoff docs) against the original acceptance criteria in `handoff-pm.md`. If the changes are materially broader than the spec — new files not anticipated in the acceptance criteria, different components touched, behavior beyond what was specified — note `scope-expanded` in the closeout summary and list what expanded. If changes are within scope, note `scope: as-specified`.
   - In **autonomous mode**: `scope-expanded` is informational only — does not block closeout.
   - In **interactive mode**: if scope expanded, ask the user to confirm the expanded scope is acceptable before completing closeout.
3. Write the final run summary to `.blueprint/workflow/handoff-closeout.md`. For **Track 0 — Hotfix**: the summary should document the incident, what was fixed, and how it was deployed — not a spec-driven pipeline summary.
4. Print this summary to the user:

```
## Run complete ✓

Task: <task>
Track: Track N — Title
Mode: Interactive | Autonomous
Loops: N fix iterations — [what was found and fixed per loop, or "none"]
Scope: as-specified | scope-expanded — [if expanded: brief note of what grew]
Artifacts: [list what was produced: spec, plan, implementation, verification report]
Status: all acceptance criteria met
```

**Handoff doc to write:** `.blueprint/workflow/handoff-pm.md`

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
- Small-scope: true | false (Track 2 only — true if single concern, few files, clear direction)
- Acceptance criteria: [listed inline]
- Spec location: [if written to a file, link it]

## Notes for next stage
[Anything tech-lead needs to know]
```
<!-- scaffold:end managed pm-role -->
```

---

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

1. Read `.blueprint/workflow/handoff-pm.md` for the spec and acceptance criteria.
2. Check `.blueprint/workflow/run-mode.md` for mode.
3. **For Track 1 — Major**: invoke `superpowers:writing-plans` using the spec as input.
4. **For Track 2 — Standard**: check `handoff-pm.md` for the `small-scope` flag.
   - If `small-scope: true` — skip formal planning. Write a brief implementation note (what to change, where) directly in the handoff doc. No `writing-plans` invocation.
   - If `small-scope: false` or not set — invoke `superpowers:writing-plans` as normal.
5. In **autonomous mode**: auto-accept plan review gates.
6. When planning finishes, do NOT surface the execution choice question ("subagent-driven or inline?") to the user — the `/run` pipeline controls Dev invocation. Write `.blueprint/workflow/handoff-tech-lead.md` and exit.

**Your role in the review stage (invoked after QA):**

1. Read `.blueprint/workflow/handoff-qa.md` for verification results.
2. Read `.blueprint/workflow/handoff-pm.md` for acceptance criteria.
3. Check `.blueprint/workflow/run-mode.md` for mode.
4. **For Track 1 — Major**: invoke `/project-doctor` (full codebase audit) before assessing PR readiness.
5. **For Track 2 — Standard**: invoke `/project-doctor` scoped to changed files only. Pass the list of changed files from `handoff-dev.md` — review only those files, not the full codebase.
6. **Escalation check**: If the implementation grew beyond Track 2 scope (new subsystem introduced, architectural boundaries crossed, > 5 files changed that weren't anticipated in the spec), set status to `escalate-to-track-1` in the handoff doc. The pipeline will restart as Track 1.
7. Assess whether the implementation meets acceptance criteria and is ready to merge.
8. In **autonomous mode**: auto-accept review gates.
9. Write `.blueprint/workflow/handoff-review.md`.

**Handoff doc to write** (planning stage): `.blueprint/workflow/handoff-tech-lead.md`

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
[Anything dev needs to know beyond the plan]
```

**Handoff doc to write** (review stage): `.blueprint/workflow/handoff-review.md`

```markdown
# Handoff: Tech Lead (Review)

## Status
complete | issues-found | escalate-to-track-1 | blocked

## Track
Track N — Title

## Summary
[Review outcome, project-doctor result if Track 1 or Track 2. If escalating: explain why scope exceeds Track 2.]

## Outputs
- PR readiness: ready | not ready
- Issues found: [listed inline, empty if none]
- project-doctor scope: full (Track 1) | changed-files-only (Track 2)

## Notes for next stage
[Deployment notes for DevOps, or closeout notes for PM]
```
<!-- scaffold:end managed tech-lead-role -->
```

---

#### `.claude/agents/dev.md`

```markdown
---
# scaffold:begin managed frontmatter
name: dev
description: Implements assigned changes, writes tests, and documents verification steps. Use for code implementation tasks.
# scaffold:end managed frontmatter
---
<!-- scaffold:begin managed dev-role -->
You are the Dev agent in the autonomous development pipeline.

**Your role when invoked by `/run`:**

1. Check if this is a **fix loop**: if `.blueprint/workflow/handoff-qa.md` exists and has status `issues-found`, read it for the list of issues to fix. Otherwise read `.blueprint/workflow/handoff-tech-lead.md` for the implementation plan (or implementation note, if Tech Lead skipped formal planning for small-scope Track 2).
2. Check `.blueprint/workflow/run-mode.md` for mode.
3. If a formal plan exists: invoke `superpowers:executing-plans` with the plan. **Automatically select subagent-driven execution — do not ask the user to choose.**
   If only an implementation note exists (small-scope Track 2): implement directly from the note — no `executing-plans` invocation needed.
4. In **autonomous mode**: auto-accept execution checkpoints and review gates within executing-plans.
5. Write `.blueprint/workflow/handoff-dev.md`.

**Handoff doc to write:** `.blueprint/workflow/handoff-dev.md`

```markdown
# Handoff: Dev

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
<!-- scaffold:end managed dev-role -->
```

---

#### `.claude/agents/qa.md`

```markdown
---
# scaffold:begin managed frontmatter
name: qa
description: Independently verifies changes meet acceptance criteria. Use after implementation to verify correctness and run tests.
# scaffold:end managed frontmatter
---
<!-- scaffold:begin managed qa-role -->
You are the QA agent in the autonomous development pipeline.

**Your role when invoked by `/run`:**

1. Read `.blueprint/workflow/handoff-dev.md` for what was implemented and how to verify it.
2. Read `.blueprint/workflow/handoff-pm.md` for the acceptance criteria.
3. Read the project verification profile in `.blueprint/project-artifacts/verification/` — follow the strategy and required evidence documented there.
4. Run all automated tests (commands from `.blueprint/project-artifacts/testing/test-commands.md`).
5. **Determine if this is a UI change.** A change is a UI change if it touches templates, components, pages, stylesheets, or anything that affects what the user sees in a browser. If uncertain, treat it as a UI change.
6. **For UI changes — MANDATORY Playwright MCP verification:**
   - Start the dev server (or use the build preview).
   - Use `mcp__playwright__browser_navigate` to open every affected page.
   - Use `mcp__playwright__browser_snapshot` to inspect the DOM and confirm UI changes render correctly.
   - Use `mcp__playwright__browser_take_screenshot` to capture visual evidence.
   - Use `mcp__playwright__browser_console_messages` with level `error` to confirm zero console errors.
   - Check every view mentioned in the acceptance criteria.
   - **This is NOT optional.** Tests passing alone is insufficient for UI changes. You MUST use Playwright MCP to visually confirm the app works in a browser.
7. **For non-UI changes** (backend, API, config, tooling): run tests and verify acceptance criteria programmatically. No browser verification required.
8. Write `.blueprint/workflow/handoff-qa.md`.

**Critical:** If issues are found, list each one precisely and specifically in the handoff doc so dev knows exactly what to fix. Vague issue descriptions cause fix loops to fail.

**Critical:** Do not mark status as `complete` for UI changes without Playwright evidence. The handoff doc must include which pages were navigated and what was confirmed via snapshot or screenshot.

**Handoff doc to write:** `.blueprint/workflow/handoff-qa.md`

```markdown
# Handoff: QA

## Status
complete | issues-found

## Track
Track N — Title

## Summary
[What was verified, what passed, what failed]

## Outputs
- Tests run: [list with pass/fail]
- Acceptance criteria checked: [each criterion with pass/fail]
- Browser verification (if UI change): [pages navigated, what was confirmed via Playwright snapshot/screenshot]

## Issues found
[If status is issues-found: list each issue with exact reproduction steps. Empty if complete.]

## Notes for next stage
[For tech-lead review: overall quality assessment]
```
<!-- scaffold:end managed qa-role -->
```

---

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

1. Read `.blueprint/workflow/handoff-review.md` to confirm the implementation is PR-ready.
2. Use deploy steps from `.blueprint/project-artifacts/deployment/deploy.md`.
3. Execute the deployment.
4. Verify the deployment succeeded.
5. Write `.blueprint/workflow/handoff-devops.md`.

**Issue classification:** If deployment fails, determine the cause:
- **Code issue** (tests pass locally but fail in deploy environment, runtime error): set status `issues-found` and note `issue-type: code` — the pipeline will route back to dev.
- **Infrastructure/deploy issue** (network, permissions, environment config): set status `issues-found` and note `issue-type: deploy` — the pipeline will retry DevOps.
- **External blocker** (third-party service down, credential expired): set status `blocked`.

**Handoff doc to write:** `.blueprint/workflow/handoff-devops.md`

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

---

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
Follow your responsibilities above. Check `.blueprint/workflow/run-mode.md` for mode. In autonomous mode, auto-accept review gates. Write a handoff doc to `.blueprint/workflow/handoff-<role-name>.md` when your stage is complete.

**Handoff doc to write:** `.blueprint/workflow/handoff-<role-name>.md`

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

New role files follow the same re-run rules: the managed sections (both frontmatter and body) are regenerated, content outside them is never touched.

---

#### `.blueprint/project-artifacts/testing/test-commands.md`

```markdown
# Test Commands

<!-- scaffold:begin managed detected-test-commands -->
[GENERATED: actual test/build commands for the detected stack]
<!-- scaffold:end managed detected-test-commands -->
```

---

#### `.blueprint/project-artifacts/verification/strategy.md`

```markdown
# Verification Strategy

<!-- scaffold:begin managed strategy-layer-defaults -->
[GENERATED: QA mode appropriate for detected stack]
<!-- scaffold:end managed strategy-layer-defaults -->

Also specify required evidence and smoke vs deeper verification expectations.
```

---

#### `.blueprint/project-artifacts/verification/browser-playwright.md` *(browser-based projects only)*

```markdown
# Browser Playwright

Playwright is available via MCP — use `mcp__playwright__*` tools directly. No npm install needed.

## How to use

1. Start the dev server: [GENERATED: actual dev command] or build: [GENERATED: build + preview commands]
2. Navigate: `mcp__playwright__browser_navigate`
3. Screenshot: `mcp__playwright__browser_take_screenshot`
4. Snapshot / assert: `mcp__playwright__browser_snapshot`
5. Console errors: `mcp__playwright__browser_console_messages`
6. Responsive: `mcp__playwright__browser_resize` to 375px and 768px

## What to verify

[GENERATED: project-specific checklist — pages to check, key user flows, design system elements to validate]
```

---

#### `.blueprint/project-artifacts/deployment/deploy.md`

```markdown
# Deploy

<!-- scaffold:begin managed deployment-defaults -->
[GENERATED: deploy steps for detected deploy target]
<!-- scaffold:end managed deployment-defaults -->
```

---

#### `.blueprint/project-artifacts/deployment/environments.md`

```markdown
# Environments

[GENERATED: detected environment names and URLs, or stub: "Document environments, URLs, gates, and owners here."]
```

---

#### `.blueprint/project-artifacts/deployment/rollback.md`

```markdown
# Rollback

[GENERATED: rollback steps for detected deploy target, or stub: "Document rollback steps and failure signals here."]
```

---

#### `.blueprint/project-artifacts/github/pr-checks.md` *(team projects only — skip for solo)*

```markdown
# PR Checks

List required CI and review gates.
```

---

#### `.blueprint/project-artifacts/review/review-strategy.md`

```markdown
# Review Strategy

<!-- scaffold:begin managed review-gate -->
Track 1 requires full `/project-doctor` codebase audit before merge.
Track 2 requires `/project-doctor` scoped to changed files only before merge.

Tech Lead may escalate Track 2 → Track 1 if implementation exceeds original scope.
<!-- scaffold:end managed review-gate -->
```

---

#### `.blueprint/project-overrides/project-profile.md` *(skip if already exists with content)*

```markdown
# Project Profile

## Product purpose
[GENERATED: read from README, package.json description, About pages, or app entry points]

## App type
[GENERATED: static site / SSR web app / REST API / CLI / desktop / mobile]

## Languages and frameworks
[GENERATED: from package.json, config files, src structure]

## Repo structure
[GENERATED: key directories and their purpose, e.g. src/pages/, src/api/, src/components/]

## Deploy targets and environments
[GENERATED: from detected deploy config]

## Test strategy
[GENERATED: from detected test runner and scripts]

## QA strategy
[GENERATED: from verification layer defaults]

## Risk areas
[GENERATED: layout/global files, auth, payment, data layer, shared utilities — whatever applies to this project]
```

---

#### `.blueprint/project-overrides/stack-overrides.md`

```markdown
# Stack Overrides

<!-- scaffold:begin managed stack-override-hints -->
Active layer: [GENERATED: stack-name]
Detected likely layer: [GENERATED: stack-name]
Override only what differs from the layer defaults.
<!-- scaffold:end managed stack-override-hints -->
```

---

#### `.blueprint/project-overrides/active-layer.txt`

Write the detected stack name, e.g.: `astro-static`

---

#### `docs/process/tracks.md` *(skip if already exists)*

```markdown
# Tracks

Use `/run <task>` to execute any task autonomously. Use `/run --auto <task>` to also auto-accept review gates.

---

## Track 0 — Hotfix

**When:** Production incident. Speed is critical.

**Pipeline:**
Tech Lead (triage + fix plan) → Dev (implement fix) → QA (smoke test) → Ship (if uncommitted) → DevOps (deploy) → PM (document after)

**Notes:** No spec phase. PM runs at closeout only to document what happened.

---

## Track 1 — Major

**When:** Significant change requiring product brainstorming, PRD, spec, and architectural review.

**Pipeline:**
PM (brainstorm + spec, stop before writing-plans) → Tech Lead (writing-plans) → Dev (executing-plans) → QA (full verification) → Tech Lead (final review + mandatory project-doctor) → PM (closeout — scope validation, gates Ship) → Ship (if uncommitted) → DevOps (if deployment required)

**Notes:** `/project-doctor` is mandatory before Tech Lead gives final approval.

---

## Track 2 — Standard

**When:** Normal feature or bug fix.

**Pipeline:**
PM (lightweight brainstorm via `simple` — always interactive + spec + acceptance criteria) → Tech Lead (writing-plans if not small-scope, skip if small-scope) → Dev (executing-plans or direct implementation) → QA (tests + Playwright for UI changes) → Tech Lead (final review + scoped project-doctor on changed files only) → PM (closeout — scope validation, gates Ship) → Ship (if uncommitted) → DevOps (if deployment required)

**Notes:**
- `simple` brainstorm is always interactive, even in `--auto` mode.
- Tech Lead skips formal planning for small-scope changes (single concern, few files).
- project-doctor runs only on changed files, not the full codebase.
- If Tech Lead finds the change grew beyond Track 2 scope, it escalates to Track 1.

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

---

#### `.blueprint/workflow/.gitkeep`

Create this file with empty content. This creates the workflow directory for runtime handoff docs.

Then add `.blueprint/workflow/` to the project's `.gitignore` if a `.gitignore` exists:

```
# Blueprint workflow runtime artifacts
.blueprint/workflow/
```

If no `.gitignore` exists, note in the Step 9 report under "Needs attention": "Add `.blueprint/workflow/` to `.gitignore` to exclude runtime handoff docs from version control."

---

#### `.claude/skills/run/SKILL.md`

```markdown
---
name: run
description: Executes a development task through the full autonomous agent pipeline. Use /run <task> for interactive mode or /run --auto <task> to auto-accept review gates.
---

# /run

Execute a development task through the full autonomous agent pipeline.

**Usage:**
- `/run <task description>` — Interactive: questions asked, review gates required
- `/run --auto <task description>` — Autonomous: review/approval gates auto-accepted. Brainstorming direction choices (`simple` and `superpowers:brainstorming`) always remain interactive — the user always picks the direction.

## What this does

Classifies your task into a track, then runs it through the appropriate agents in sequence: PM → Tech Lead → Dev → QA → Tech Lead review → PM closeout (scope validation) → Ship → DevOps. Issues are fixed automatically by looping back to the appropriate stage. Only genuine blockers or input requests pause the pipeline.

## Before starting

**Track classification for Track 0:** Before spawning any agent, determine if this is a Track 0 — Hotfix. Signals: the task description mentions a production incident, uses the word "hotfix", says something is down or broken in production, or explicitly asks for an emergency fix. If Track 0, skip PM and spawn Tech Lead first (see pipelines below). For all other tasks, spawn PM first to classify the track.

1. Create `.blueprint/workflow/` if it doesn't exist.
2. Delete any existing handoff docs: remove all files matching `.blueprint/workflow/handoff-*.md` and `.blueprint/workflow/run-mode.md`.
3. Write `.blueprint/workflow/run-mode.md`:
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
| `skipped` | (Ship stage only) Print `✓ skipped — no uncommitted changes`, continue to next stage |
| `scope-expanded` | (PM closeout only) Print `⚠ SCOPE EXPANDED: <summary>`. In autonomous mode: informational, continue. In interactive mode: surface to user for confirmation before completing. |
| `escalate-to-track-1` | (Tech Lead review only) Print `⚡ ESCALATING TO TRACK 1: <reason>`. Restart pipeline as Track 1 from the beginning. |

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
- `⚠ SCOPE EXPANDED` for scope-expanded (informational in autonomous mode)

Example:
```
[1/8] PM — Track 1 — Major detected. Writing spec...         ✓
[2/8] Tech Lead — Planning implementation...                  ✓
[3/8] Dev — Implementing...                                   ✓
[4/8] QA — Verifying...                                       ✗ 2 issues found
[3/8] Dev — Fixing issues (attempt 1/3)...                    ✓
[4/8] QA — Re-verifying...                                    ✓
[5/8] Tech Lead — Final review + project-doctor...            ✓
[6/8] PM — Closeout (scope validated)...                      ✓
[7/8] Ship — Committing and creating PR...                    ✓
[8/8] DevOps — Deploying...                                   ✓
```

## Auto-heal routing

When a stage returns `issues-found`, route back as follows:

| Stage with issues | Route back to |
|---|---|
| QA | Dev → QA |
| Tech Lead (review) | Dev → QA → Tech Lead (review) |
| Tech Lead (review) — `escalate-to-track-1` | Restart pipeline as Track 1 from the beginning (PM brainstorm + spec) |
| project-doctor (inside Tech Lead review) | Dev → QA → Tech Lead (review, includes project-doctor) |
| Ship | **BLOCKED** — surface to user (commit/push failures require manual intervention) |
| DevOps — `issue-type: code` | Dev → QA → Tech Lead (review) → PM closeout → Ship → DevOps |
| DevOps — `issue-type: deploy` | DevOps retry |

To determine DevOps issue type: read the `## Issues found` section of `handoff-devops.md` for the `issue-type:` line.

## Pipelines by track

PM classifies the track in its first stage. Read `handoff-pm.md` after the PM stage to determine which pipeline to follow.

**Track 0 — Hotfix** (skip PM spec, start with Tech Lead):
```
[1] Tech Lead — triage, plan the fix
[2] Dev — implement fix
[3] QA — smoke test
[4] Ship — commit + push (if uncommitted changes exist)
[5] DevOps — deploy
[6] PM — document what happened (closeout only)
```

**Track 1 — Major:**
```
[1] PM — brainstorm + spec (stop before writing-plans)
[2] Tech Lead — write-plans
[3] Dev — executing-plans
[4] QA — full verification
[5] Tech Lead — final review + project-doctor
[6] PM — closeout (scope validation — gates Ship)
[7] Ship — commit + push/PR (if uncommitted changes exist)
[8] DevOps — deploy (if task requires deployment)
```

**Track 2 — Standard:**
```
[1] PM — lightweight brainstorm (simple, always interactive) + spec + acceptance criteria
[2] Tech Lead — write-plans (skip if small-scope — PM hands off directly to Dev)
[3] Dev — executing-plans (or direct implementation if no formal plan)
[4] QA — tests + mandatory Playwright for UI changes
[5] Tech Lead — final review + scoped project-doctor (changed files only)
[6] PM — closeout (scope validation — gates Ship)
[7] Ship — commit + push/PR (if uncommitted changes exist)
[8] DevOps — deploy (if task requires deployment)
```

Note: If Tech Lead review returns `escalate-to-track-1`, the pipeline restarts as Track 1 from the beginning.

**Track 3 — Non-Code:**
```
[1] PM — owns entirely, no further stages unless delegated
```

For Track 0: spawn Tech Lead first with the task and note it is a hotfix. PM runs only at closeout.

For Tracks 1 and 2: spawn PM first. Read `handoff-pm.md` to confirm track before proceeding.

For Track 3: spawn PM with the task. PM will own it and write handoff-pm.md when done.

## Ship stage

The Ship stage commits verified work and makes it available for deployment or merge. It runs after Tech Lead review passes.

**Skip condition:** Run `git status` first. If there are no uncommitted changes (working tree clean, nothing staged), skip this stage entirely — set status to `skipped` in the handoff doc and continue to the next stage.

**When there are uncommitted changes:**

1. Run `git status` to identify changed/new files.
2. Stage relevant files (`git add` — prefer specific files over `git add -A`; never commit `.env`, credentials, or `.blueprint/workflow/` files).
3. Derive a commit message from the PM spec (`handoff-pm.md` summary + acceptance criteria). The message should explain *why* the change was made, not just list files.
4. Commit with the derived message.
5. Determine push target:
   - If on a feature branch: push to remote with `-u` flag.
   - If on `main`/`master` and the project is solo (no CODEOWNERS, no PR checks configured): push directly.
   - If on `main`/`master` and the project is team-based: create a feature branch first (`git checkout -b <branch-name>`), push, then create a PR using `gh pr create`.
6. Write `.blueprint/workflow/handoff-ship.md`.

**If commit or push fails:** Set status to `blocked` with the error details. The pipeline will surface this to the user (Ship failures are not auto-healable).

**Handoff doc to write:** `.blueprint/workflow/handoff-ship.md`

```markdown
# Handoff: Ship

## Status
complete | skipped | blocked

## Track
Track N — Title

## Summary
[What was committed: commit message used, branch pushed to, PR created (if any). If skipped: "No uncommitted changes — nothing to ship."]

## Outputs
- Commit: [short SHA + message, or "n/a" if skipped]
- Branch: [branch name]
- PR: [URL if created, "n/a" if pushed directly, "n/a" if skipped]
- Pushed to remote: yes | no | skipped

## Issues found
[If blocked: the exact error from git/gh. Empty otherwise.]

## Notes for next stage
[For DevOps: branch/PR details. For PM closeout: what was shipped.]
```

## DevOps stage

Only run the DevOps stage if the task actually requires deployment. Signs a deployment is needed:
- The task description mentions deploy, release, or shipping
- The plan (handoff-tech-lead.md) includes deploy steps
- Tech Lead review (handoff-review.md) notes deployment is required

If deployment is not required, skip DevOps and go directly to PM closeout.
```

---

## Step 8 — Install project-doctor

Read `~/.claude/plugins/installed_plugins.json` and check for any key containing `project-doctor`.

- **Found** → already installed, skip. Do not write any skill file.
- **Not found** → tell the user:

  > `project-doctor` is not installed. Run the following to install it:
  > ```
  > /plugin install https://github.com/soguy/skills.git project-doctor
  > ```
  > Re-run `/blueprint` after installing to complete setup.

  Then **stop** — do not write a local skill file or embed content inline.

Wire `review-strategy.md` managed section to point to `/project-doctor` regardless of install status.

---

## Step 9 — Report

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
Use `/project-doctor` for deep review before merge (runs automatically in Track 1 and Track 2 via /run).
Re-run `/blueprint` anytime to refresh configuration, add roles, or update to a newer version.
```

---

## Pending layer (new project, unknown stack)

When the user says the stack is unknown or cannot be detected, create only the structural files:

- `CLAUDE.md` — append `@.blueprint/BLUEPRINT.md` include line (create file if needed)
- `.blueprint/BLUEPRINT.md` — full track instructions content as defined in Step 7
- `.claude/agents/pm.md`, `tech-lead.md`, `dev.md`, `qa.md`, `devops.md` — full content as defined in Step 7 (with YAML frontmatter managed sections and pipeline-aware bodies), no role customization prompt in pending mode
- `.claude/skills/run/SKILL.md` — full /run skill as defined in Step 7
- `.blueprint/workflow/.gitkeep` — creates the workflow directory
- `docs/process/tracks.md` — full content as above
- `.blueprint/project-overrides/project-profile.md` — all sections present but values set to `[TBD — re-run /blueprint once stack is chosen]`
- `.blueprint/layers/pending/README.md` — content: `Stack not yet determined. Re-run /blueprint once the technology stack is chosen.`
- `.blueprint/project-overrides/active-layer.txt` — content: `pending`

Do NOT create: test-commands, verification strategy, deployment files, browser-playwright, stack-overrides.

Then tell the user:

> Structural scaffold complete. Re-run `/blueprint` once you've chosen your technology stack to generate the layer, test commands, verification strategy, deployment config, and project profile.
