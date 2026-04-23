---
name: blueprint
description: "Set up a structured AI-assisted development workflow for your project. Installs agent roles (PM, Tech Lead, Dev, QA, DevOps), a three-track process, project-doctor deep review, and configures everything for your tech stack. Run on any new or existing project. Re-run anytime to update."
---

# blueprint

## What this does

**blueprint** sets up a structured AI-assisted development workflow for your project. It installs:

- **Agent roles** — PM, Tech Lead, Dev Worker, QA Verifier, DevOps
- **Three-track process** — lightweight for small tasks, rigorous for large ones
- **project-doctor** — mandatory deep review gate for Track 1 work
- **Tech stack configuration** — layer, test commands, verification strategy, and deploy setup tailored to your actual project

Once set up, Claude will automatically follow this process on every task: classify it, spec it if needed, implement it, verify it, and review it before merge.

**Re-run `/blueprint` anytime:** after a stack change, to refresh stale config, or to pull improvements from a newer skill version. Re-runs are safe — manually added content is never overwritten.

---

## Step 1 — Introduce yourself

Print the following before doing anything else:

> **blueprint** sets up a structured AI-assisted development workflow for your project. It installs agent roles (PM, Tech Lead, Dev, QA, DevOps), a three-track process (lightweight for small tasks, rigorous for large ones), deep code review via project-doctor, and configures everything specifically for your tech stack.
>
> Once set up, Claude will automatically follow this process on every task — classify it, spec it if needed, implement it, verify it, and review it before merge.
>
> Re-run `/blueprint` anytime: after a stack change, to update stale config, or to pull in improvements from a newer skill version. Re-runs are safe — manually added content is never overwritten.

---

## Step 2 — Detect mode

Check the project directory:

- **Re-run**: `.claude/` directory already exists → tell user: *"Re-running blueprint — updating managed sections, leaving manual content untouched."*
- **Existing project, first run**: Source files exist but no `.claude/`
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
| Dev Worker | Implementation, tests, verification docs |
| QA Verifier | Independent verification using project verification profile |
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

Create `.claude/layers/<stack-name>/` with four files tailored to this exact project.

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

### Files to write

Write each file below. Substitute `[GENERATED: ...]` placeholders with content derived from Step 4 analysis.

---

#### `CLAUDE.md`

```markdown
# Claude Project Operating Guide

<!-- scaffold:begin managed track-instructions -->
Always classify work into Track 1, Track 2, or Track 3 before acting.

**Track 1** — Risky or architectural change. Mandatory `/project-doctor` review before merge.
**Track 2** — Normal feature or fix. Standard QA, no deep review required.
**Track 3** — No code change. Documentation, planning, research.

Use the process docs and project artifacts in `.claude/` before making assumptions.
For Track 1, `/project-doctor` is mandatory before final merge readiness.
<!-- scaffold:end managed track-instructions -->
```

---

#### `.claude/agents/pm.md`

```markdown
<!-- scaffold:begin managed pm-role -->
Owns task classification, specs, acceptance criteria, and closeout.
Run `superpowers:brainstorming` before Track 1 specs.
For UI work, run `frontend-design` before Tech Lead handoff.
<!-- scaffold:end managed pm-role -->
```

---

#### `.claude/agents/tech-lead.md`

```markdown
<!-- scaffold:begin managed tech-lead-role -->
Owns architecture decisions, delegation, review, tests, PR readiness, and final merge handoff.
Must enforce dependencies and mandatory `/project-doctor` review for Track 1.
<!-- scaffold:end managed tech-lead-role -->
```

---

#### `.claude/agents/dev-worker.md`

```markdown
<!-- scaffold:begin managed dev-worker-role -->
Implements the assigned change, writes or updates tests, and documents how to verify the change.
<!-- scaffold:end managed dev-worker-role -->
```

---

#### `.claude/agents/qa-verifier.md`

```markdown
<!-- scaffold:begin managed qa-verifier-role -->
Owns independent verification. Use the project verification profile in `.claude/project-artifacts/verification/`.
For browser-based projects: use Playwright MCP (`mcp__playwright__*`) if available.
<!-- scaffold:end managed qa-verifier-role -->
```

---

#### `.claude/agents/devops.md`

```markdown
<!-- scaffold:begin managed devops-role -->
Owns deployment and deployment verification when the task requires it.
<!-- scaffold:end managed devops-role -->
```

---

#### Custom roles and modified responsibilities (from Step 5)

For each **modified default role**: replace the managed section content with the user-provided text instead of the default.

For each **new role** collected in Step 5, create `.claude/agents/<role-name>.md`:

```markdown
<!-- scaffold:begin managed <role-name>-role -->
[User-provided responsibility description]
<!-- scaffold:end managed <role-name>-role -->
```

New role files follow the same re-run rules: the managed section is regenerated, content outside it is never touched.

---

#### `.claude/project-artifacts/testing/test-commands.md`

```markdown
# Test Commands

<!-- scaffold:begin managed detected-test-commands -->
[GENERATED: actual test/build commands for the detected stack]
<!-- scaffold:end managed detected-test-commands -->
```

---

#### `.claude/project-artifacts/verification/strategy.md`

```markdown
# Verification Strategy

<!-- scaffold:begin managed strategy-layer-defaults -->
[GENERATED: QA mode appropriate for detected stack]
<!-- scaffold:end managed strategy-layer-defaults -->

Also specify required evidence and smoke vs deeper verification expectations.
```

---

#### `.claude/project-artifacts/verification/browser-playwright.md` *(browser-based projects only)*

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

#### `.claude/project-artifacts/deployment/deploy.md`

```markdown
# Deploy

<!-- scaffold:begin managed deployment-defaults -->
[GENERATED: deploy steps for detected deploy target]
<!-- scaffold:end managed deployment-defaults -->
```

---

#### `.claude/project-artifacts/deployment/environments.md`

```markdown
# Environments

[GENERATED: detected environment names and URLs, or stub: "Document environments, URLs, gates, and owners here."]
```

---

#### `.claude/project-artifacts/deployment/rollback.md`

```markdown
# Rollback

[GENERATED: rollback steps for detected deploy target, or stub: "Document rollback steps and failure signals here."]
```

---

#### `.claude/project-artifacts/github/pr-checks.md` *(team projects only — skip for solo)*

```markdown
# PR Checks

List required CI and review gates.
```

---

#### `.claude/project-artifacts/review/review-strategy.md`

```markdown
# Review Strategy

<!-- scaffold:begin managed review-gate -->
Track 1 requires mandatory deep review before merge.
Run: `/project-doctor`
<!-- scaffold:end managed review-gate -->
```

---

#### `.claude/project-overrides/project-profile.md` *(skip if already exists with content)*

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

#### `.claude/project-overrides/stack-overrides.md`

```markdown
# Stack Overrides

<!-- scaffold:begin managed stack-override-hints -->
Active layer: [GENERATED: stack-name]
Detected likely layer: [GENERATED: stack-name]
Override only what differs from the layer defaults.
<!-- scaffold:end managed stack-override-hints -->
```

---

#### `.claude/project-overrides/active-layer.txt`

Write the detected stack name, e.g.: `astro-static`

---

#### `docs/process/tracks.md` *(skip if already exists)*

```markdown
# Tracks

## Track 1
PM → Tech Lead → Devs → QA → mandatory `/project-doctor` review → Tech Lead → DevOps if needed → PM

## Track 2
PM → Tech Lead → Devs → QA → Tech Lead → DevOps if needed → PM

## Track 3
PM owns or delegates, no code changes required.
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

### Skipped (already customised — manual content preserved)
[list files that exist without managed sections]

### Needs attention
[anything that couldn't be auto-filled, e.g. environments.md if no deploy target was detected, missing test commands]

---
From now on, Claude will classify every task as Track 1, 2, or 3 before acting.
Use /project-doctor for Track 1 deep review before merge.
Re-run /blueprint anytime to refresh your configuration or add/update roles.
```

---

## Pending layer (new project, unknown stack)

When the user says the stack is unknown or cannot be detected, create only the structural files:

- `CLAUDE.md` — full content with track instructions managed section
- `.claude/agents/pm.md`, `tech-lead.md`, `dev-worker.md`, `qa-verifier.md`, `devops.md` — full content as above (with managed sections), no role customization prompt in pending mode
- `docs/process/tracks.md` — full content as above
- `.claude/project-overrides/project-profile.md` — all sections present but values set to `[TBD — re-run /blueprint once stack is chosen]`
- `.claude/layers/pending/README.md` — content: `Stack not yet determined. Re-run /blueprint once the technology stack is chosen.`
- `.claude/project-overrides/active-layer.txt` — content: `pending`

Do NOT create: test-commands, verification strategy, deployment files, browser-playwright, stack-overrides.

Then tell the user:

> Structural scaffold complete. Re-run `/blueprint` once you've chosen your technology stack to generate the layer, test commands, verification strategy, deployment config, and project profile.
