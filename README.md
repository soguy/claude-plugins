# ai-workflow-plugins

Guy Soffer's Claude Code plugin marketplace — AI-assisted development workflow tools.

## Skills

### blueprint
Sets up a structured AI-assisted development workflow for any project. Installs agent roles, a three-track process, project-doctor deep review, and configures everything for your tech stack. Works on new and existing projects. Re-run anytime to update.

### project-doctor
Deep codebase health check: dead code removal, bug detection, security audit, sanity checks, and documentation sync. Works with any tech stack.

---

## Install via Claude Code plugin marketplace

Add this marketplace once:
```
/plugin marketplace add https://github.com/soguy/ai-workflow-plugins.git
```

Then install individual skills:
```
/plugin install blueprint
/plugin install project-doctor
```

---

## Manual install (no plugin system needed)

Copy the skill file directly into your project:

**blueprint:**
```bash
mkdir -p .claude/skills/blueprint
curl -o .claude/skills/blueprint/SKILL.md \
  https://raw.githubusercontent.com/soguy/ai-workflow-plugins/main/plugins/blueprint/.claude/skills/blueprint/SKILL.md
```

**project-doctor:**
```bash
mkdir -p .claude/skills/project-doctor
curl -o .claude/skills/project-doctor/SKILL.md \
  https://raw.githubusercontent.com/soguy/ai-workflow-plugins/main/plugins/project-doctor/.claude/skills/project-doctor/SKILL.md
```

Then invoke in Claude Code:
```
/blueprint
/project-doctor
```

---

## Usage

### Set up a new or existing project
```
/blueprint
```
blueprint will detect your stack, ask about dependencies, and write the full workflow configuration.

### Run a deep code review
```
/project-doctor
```
project-doctor runs a structured pipeline: dead code removal, bug detection, security audit, sanity checks, and documentation sync.

---

## Skills in detail

### blueprint

blueprint sets up a complete AI-assisted development workflow for any project — new or existing, any tech stack.

**What it does:**

1. **Detects your stack** — reads `package.json`, `pyproject.toml`, `go.mod`, `Cargo.toml`, CI configs, and the source tree to identify your framework, test commands, and deploy target automatically.

2. **Installs agent roles** — creates `.claude/agents/` files for five roles: PM (specs and acceptance criteria), Tech Lead (architecture and PR readiness), Dev Worker (implementation), QA Verifier (independent testing), and DevOps (deployment). You can customise any role's responsibilities or add new roles during setup.

3. **Sets up a three-track process** — every task gets classified before Claude acts on it:
   - **Track 1** (risky or architectural) — mandatory `/project-doctor` deep review before merge
   - **Track 2** (normal feature or fix) — standard QA, no deep review required
   - **Track 3** (no code changes) — documentation, planning, research

4. **Generates a project layer** — writes stack-specific config files for test commands, verification strategy (including Playwright MCP for browser projects), and deployment steps.

5. **Installs project-doctor** — wires the Track 1 review gate automatically.

**Re-run anytime** with `/blueprint` after a stack change, to refresh stale config, or to add/update roles. Re-runs are safe: managed sections are regenerated, any content you've added manually is never touched.

**Why it matters:**  
Without a process, Claude will attempt tasks without classifying risk, skip verification steps, and make architectural decisions ad-hoc. blueprint forces a consistent discipline: classify first, spec if needed, implement, verify, and review before merge. The three-track system means small tasks stay lightweight while high-risk changes get appropriate gates.

---

### project-doctor

project-doctor runs a structured six-phase pipeline to diagnose and heal a codebase. It works with any tech stack.

**What it does:**

**Phase 0 — Orient:** Reads the project structure, README, and CI config to understand the stack, entry points, and canonical build commands before touching anything.

**Phase 0.5 — Simplify:** Runs the `simplify` skill on recently changed code to catch reuse and efficiency issues before they get baked in further.

**Phase 1 — Dead code removal:** Uses static analysis tools (`ts-prune`, `vulture`, `cargo udeps`, `go vet`) where available, then manually checks for unreachable exports, commented-out blocks, stale feature flags, and duplicate utilities. Re-runs the build after removal to confirm nothing broke.

**Phase 2 — Bug detection and fixes:** Scans for off-by-one errors, null dereferences, uncaught async errors, resource leaks, race conditions, swallowed errors, and type coercion surprises. Fixes unambiguous bugs directly; marks ambiguous cases with `// FIXME(review):` for human review.

**Phase 3 — Security audit:** Covers the OWASP Top 10 and language-specific pitfalls — hardcoded secrets, SQL injection, command injection, path traversal, XSS, broken auth, insecure session handling, JWT issues, missing CORS restrictions, vulnerable dependencies, and debug mode left on in production. Auto-fixes clear-cut issues; flags architectural decisions with `// SECURITY(review):`.

**Phase 4 — Sanity checks:** Runs your existing test suite, linter, and type checker (`npm test`, `pytest`, `cargo test`, `go test`, etc.) and distinguishes pre-existing failures from anything introduced by its own changes.

**Phase 5 — Documentation sync:** Checks every claim in `README.md` and `docs/` against the actual codebase and fixes stale content in-place. Updates inline docstrings for changed functions. Adds a CHANGELOG entry under `## Unreleased`.

**Phase 6 — Summary report:** Produces a structured report with severity-ranked security findings (🔴 CRITICAL → ℹ️ INFO), a full list of unfixed findings with reasons, and recommended next steps. Offers to fix remaining findings interactively.

**Why it matters:**  
Code review typically catches logic bugs but misses dead code accumulation, security misconfigurations, and documentation drift. project-doctor addresses all of these in one pass, with a consistent severity taxonomy and a report you can act on immediately. Use it as the mandatory Track 1 gate in a blueprint workflow, or run it standalone on any codebase that needs a health check.

---

## Adding skills to this marketplace

1. Create `plugins/<skill-name>/.claude/skills/<skill-name>/SKILL.md`
2. Add an entry to `.claude-plugin/marketplace.json`
3. Push to GitHub — existing users get the update automatically
