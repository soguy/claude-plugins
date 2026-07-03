---
name: project-doctor
description: "Diagnoses and heals software projects end-to-end: dead code removal, bug detection, security audit, build/test sanity checks, and documentation sync. Use for code review, cleanup, audit, refactoring, or any codebase health task."
---

# Project Doctor Skill

A structured pipeline for reviewing a change (or auditing a whole project): finding bugs, catching security issues, cleaning up dead code, verifying tests still pass, and keeping documentation in sync with reality.

**Scope-aware and effort-tiered.** The default run reviews the *current diff*, not the whole codebase — bugs added by this branch matter more than pre-existing lint. Full-codebase mode is opt-in.

## Invocation modes

Detect from the user's request:

| Signal | Mode | Behavior |
|---|---|---|
| `--quick` or "quick check" or "before commit" | **quick** | Phases 0, 0.5, 2, 4, 6. Simplify + diff-scoped bug detection + sanity checks + report. ~30-60s. |
| No flag, active diff exists | **diff review** *(default)* | All phases, scoped to touched files. Codex cross-check runs. ~2-4 min. |
| `--deep` or "audit" or "codebase health" or empty diff | **audit** | All phases, whole codebase. Adversarial Codex review. Loop-until-dry for finder phases. ~5-15 min. |

Announce the mode at the top of your first response so the user can override: *"Running in **diff review** mode — 5 files touched. Say `--deep` if you want a full-codebase audit."*

---

## Phase 0 — Orient

1. Read `view /path/to/project` to see the directory tree.
2. Look for `README.md`, `CHANGELOG.md`, `package.json` / `pyproject.toml` / `Cargo.toml` / equivalent to understand purpose, stack, and entry points.
3. Identify the primary language(s), framework(s), and test runner.
4. Note CI config (`.github/workflows`, `Makefile`, `Justfile`) — the canonical build/test/lint commands.
5. **Read the project's convention docs.** Look for `CLAUDE.md`, `AGENTS.md`, `.cursorrules`, `CONTRIBUTING.md`, `.github/copilot-instructions.md` — anywhere the project encodes its own review rules. Extract project-specific patterns (naming conventions, "always call X after Y" rules, "these N docs must update together", API/DB idioms). Feed these into Phase 2 and Phase 5.

Summarize your findings to the user in 3-5 bullet points, including any project-specific rules you'll be applying. Then proceed immediately to Phase 0.25.

---

## Phase 0.25 — Scope detection

Run once, upfront:

```bash
git diff @{upstream}...HEAD --stat 2>/dev/null || git diff main...HEAD --stat 2>/dev/null || git diff HEAD~1 --stat 2>/dev/null
git diff HEAD --stat  # working tree
git ls-files --others --exclude-standard  # untracked files
```

**Decision:**

- **Diff has content** (committed or working tree): `SCOPE=diff`. Store the touched file list; downstream phases only analyze/report on issues *inside these files* or *directly caused by them*. Findings outside the diff are suppressed unless they explain a diff-caused failure.
- **Both diffs empty AND `--deep` or audit mode requested**: `SCOPE=audit`. All phases analyze the whole codebase.
- **Both diffs empty AND no explicit audit request**: ask the user *"No active diff. Did you want a full-codebase audit, or is there a specific area you want reviewed?"* — don't guess.

Print one line: *"Scope: {diff review, N files: ...} | {audit, whole codebase}"* and continue.

---

## Phase 0.5 — Simplify

Run the `simplify` skill on the changed code. Catches reuse, quality, and efficiency issues before they get baked in. This is the leanest, fastest phase — never skip it.

Invoke: use the `Skill` tool with `skill: "simplify"`.

When simplify completes, **do not pause or wait for user input** — proceed immediately to Phase 2.

**Skip in audit mode** — simplify is diff-scoped by design.

---

## Phase 2 — Bug Detection & Fixes

Run before dead-code removal (Phase 3) — bugs are urgent, dead code isn't.

### 2a. Generic patterns

For each file in scope, scan for:

1. **Off-by-one errors** — loop bounds, slice indices, pagination offsets
2. **Null / undefined dereference** — property access without guards
3. **Async errors not caught** — `await` inside `try` without `catch`, unhandled Promise rejections, `finally` missing state resets
4. **Resource leaks** — file handles, DB connections, event listeners never cleaned up (esp. React `useEffect` without cleanup)
5. **Race conditions** — shared mutable state modified in concurrent paths; boolean flags that should be counters
6. **Incorrect error propagation** — swallowed errors (`catch (e) {}`), wrong HTTP status codes, silent state resets after failed writes
7. **Type coercion surprises** — `==` vs `===`, implicit string/number conversion, `null` vs `undefined` truthiness
8. **Hardcoded secrets or localhost URLs** — flag, don't auto-fix
9. **Missing input validation** — user-supplied data reaching DB queries, shell commands, or file paths
10. **Logic inversion** — `if (!error) { throw }` style mistakes

### 2b. Project-specific patterns

Apply the convention rules you extracted from `CLAUDE.md` / `AGENTS.md` / `.cursorrules` in Phase 0. These are usually where the real bugs live — projects encode rules for the mistakes that hurt them.

Examples of the *shape* to look for:

- "*Always call X after Y*" rules (e.g. "invalidate cache after junction table writes") → grep for Y, verify each hit calls X.
- Data-access idioms (e.g. "use RealDictCursor, so `row[0]` raises") → grep for anti-patterns.
- Cross-doc sync rules ("these four files must update together") → check whether the current diff violates the invariant.
- Chokepoint patterns ("all writes flow through `crud.ENTITY_DEFS.updatable`") → verify new code respects the chokepoint.

If the project has no such doc, note that and stick to 2a.

### 2c. Fix approach

- Fix unambiguous bugs (clear wrong → clear right).
- For ambiguous cases, add a `// FIXME(review): <description>` comment and list them in the summary.
- Do not refactor working code just because you disagree with the style — that's `/simplify`, not this phase.

---

## Phase 3 — Security Audit

**Filter checks by what the diff actually touches.** In diff mode, do not run every security pattern below — grep the changed files first, run only the categories that match.

### 3a. Diff → category mapping

| Diff touches | Run |
|---|---|
| SQL / ORM raw queries (`execute(`, `f"SELECT`, `.raw()`) | 3b (injection: SQL) |
| `subprocess`, `os.system`, `child_process`, backticks | 3b (injection: command) |
| File paths from user input, `open()`, `fs.readFile()` | 3b (injection: path traversal) |
| `dangerouslySetInnerHTML`, `innerHTML`, `v-html`, Jinja `\|safe` | 3b (injection: template/XSS) |
| Auth middleware, JWT, session code | 3c (authn/authz) |
| API response payloads, logging | 3d (data exposure) |
| `package.json` / `pyproject.toml` / `Cargo.toml` deps | 3e (dependency audit) |
| CORS config, TLS config, cookie flags | 3e (config) |
| Secrets, `.env`, config files | 3a (secrets) |
| `eval`, `exec`, `pickle.loads`, `yaml.load` on untrusted data | 3f (language-specific) |

If none of the diff files match any category, print *"Security: no relevant surface touched, skipping"* and move on.

In audit mode, run all categories.

### 3b. Category details (only for categories the diff activated)

Full details expanded below. Skip past subsections you didn't activate.

**Secrets & credentials (3a):** hardcoded API keys / passwords / tokens; check `.gitignore` covers `.env`, `*.pem`, `*.key`; run `git log --all -p -- '*.env' '*.key' '*.pem'` to check history exposure; flag frontend bundles that ship secrets; default admin/admin credentials.

**Injection (3b):** raw string concatenation in SQL/queries; `subprocess.run(shell=True)` or `os.system` with user input; user-supplied filenames in `open()` without a stay-within-parent check; user input in templates without escaping; `dangerouslySetInnerHTML`/`innerHTML`/`v-html`; user input into logs without sanitization (CRLF forging).

**Authn/Authz (3c):** endpoints missing auth; endpoints checking auth but not authorization (any-user-can-hit-admin-route); session tokens in URLs; missing `HttpOnly`/`Secure`/`SameSite`; plaintext / MD5 / SHA1 password storage; missing login rate limiting; JWT `alg: none` accepted; JWT missing expiry.

**Data exposure (3d):** passwords, tokens, internal IDs, PII in API responses; stack traces / SQL errors exposed to users; wildcard CORS in production; unencrypted PII at rest.

**Dependencies & config (3e):** `npm audit` / `pip audit` / `cargo audit` — flag critical/high; outdated majors with known CVEs; `DEBUG=true` or `FLASK_ENV=development` in production; missing CSP / HSTS / X-Frame-Options / X-Content-Type-Options; HTTP URLs in production config; `verify=False` on TLS calls.

**Language-specific (3f):**

| Language | Watch for |
|---|---|
| Python | `eval`, `exec`, `pickle.loads` untrusted, `yaml.load` without SafeLoader, `subprocess(shell=True)`, `__import__` with user input |
| JS/TS | `eval`, `Function()`, prototype pollution via `Object.assign`/spread on user input, dynamic `require()`, RegExp DoS |
| Go | `fmt.Sprintf` in SQL, unchecked `err`, `unsafe` usage |
| Rust | `unsafe` blocks, `.unwrap()` on user input, `Command` with unsanitized args |
| Ruby | `send`/`public_send` with user input, `ERB` without escaping, `YAML.load` untrusted |

### 3c. Fix approach

- **Auto-fix** clear-cut issues: replace `shell=True` with argument lists, add parameterized queries, tighten `.gitignore`, add cookie flags.
- **Flag but don't auto-fix** anything architectural (add auth middleware, choose encryption). Mark `// SECURITY(review): <description>`.
- **Never remove security controls** even if they look unused — they may be defense-in-depth. Ask first.
- **Rotate exposed secrets** — if a secret is in git history, flag **HIGH** severity. Deleting the current reference is not enough; the value is already in history and must be rotated externally.

---

## Phase 4 — Sanity Checks

Verify tests pass. Prefer CI signal over local re-run when both are available and green.

### 4a. Check CI first

If the branch has been pushed and there's a CI provider configured (`.github/workflows`, `.gitlab-ci.yml`, etc.), check the last run:

```bash
gh pr checks 2>/dev/null || gh run list --branch "$(git branch --show-current)" --limit 1 --json status,conclusion,createdAt
```

- **CI green within the last hour, on this SHA** → note *"CI passed on {SHA} at {time}"* and skip 4b. This is the common case in a mature project.
- **CI failed or is running or is stale** → run 4b locally.
- **No CI configured** → run 4b locally; also note *"no CI detected"* in the report so the user knows to consider adding one.

### 4b. Local verification (fallback)

Try each in order, stop at the first that works:

```bash
npm test               # Node / JS
npm run lint
npx tsc --noEmit       # TypeScript type check

python -m pytest       # Python
python -m mypy .

cargo test             # Rust
cargo clippy -- -D warnings

go test ./...          # Go
go vet ./...

make test              # Generic Makefile
just test              # Justfile
```

Record output. If tests fail:

1. Determine if the failure is *caused by this diff* or *pre-existing*.
2. Fix diff-caused failures first.
3. List pre-existing failures separately in the report — the user needs to know but this run is not the place to fix them unless they're one-line obvious.

---

## Phase 5 — Documentation Sync

**Scope docs to what changed.** Do not scan every markdown file in the repo — only docs that reference the code / API / files this diff touched.

### 5a. Discover relevant docs

For each file in scope, find docs that reference it:

```bash
# For each changed file / function / endpoint, find where it's mentioned:
grep -rn "function_name\|/api/endpoint\|filename.py" docs/ README.md *.md 2>/dev/null
```

Also honor project-specific sync rules found in Phase 0. Example from Kiwi's `CLAUDE.md`:

> Whenever new MCP tools, MCP-tool signature changes, or user-visible Kiwi behaviors land, four docs must update in lockstep: [list].

When the current diff triggers such a rule, verify all four (or however many) actually updated. Missing sync-required docs is a first-class finding.

### 5b. Update stale claims in-place

For each doc that references touched surface:

- Verify factual claims (function signatures, endpoint paths, field names, env var names).
- **Fix stale content directly** — do not merely flag. Change the number, change the field name, mark completed TODO items.
- Sanity-check code snippets against current signatures.
- Flag broken relative links introduced when files moved.

### 5c. Inline docs

- Update docstrings / JSDoc for functions changed in Phase 2-3.
- Remove doc comments for code deleted in Phase 3 (dead code removal, next phase).
- Fix `@param` / `:type` annotations that no longer match signatures.

### 5d. CHANGELOG / release notes

If `CHANGELOG.md` exists, add an entry under `## Unreleased`:

```markdown
## [Unreleased]
### Removed
- Deleted unused `fooHelper` utility (dead code)
### Fixed
- Fixed off-by-one in pagination offset calculation
### Security
- Fixed SQL injection in search endpoint
```

---

## Phase 5.5 — Codex Cross-Check (conditional)

Goal: second opinion from Codex on the cleaned, fixed, doc-synced state. **Codex finds, Claude fixes** — never round-trip Codex for the fix, that defeats the token-saving point.

### 5.5a. Detection gates

Both must pass:

1. **Host supports subagents.** This skill dispatches the review via the `codex:codex-rescue` subagent (Agent tool). If the Agent tool with subagent types is unavailable — Codex CLI, Cursor, stripped-down harness — skip.
2. **Codex plugin is installed.** Test:
   ```bash
   ls ~/.claude/plugins/cache/openai-codex/codex/*/commands/review.md 2>/dev/null \
     || ls ~/.claude/plugins/marketplaces/openai-codex/plugins/codex/commands/review.md 2>/dev/null
   ```

If either gate fails, log one line — *"Phase 5.5 skipped: {reason}"* — and continue.

### 5.5b. Run the review

Dispatch through the `codex:codex-rescue` subagent (Agent tool, `subagent_type: "codex:codex-rescue"`). Its purpose is to forward any prompt to the Codex runtime and return Codex's response verbatim.

**Review depth by mode:**

- **quick / diff-review** → *standard review* prompt. Ask Codex to find implementation defects, missed edge cases, race conditions, cleanup gaps.
- **audit / --deep** → *adversarial review* prompt. Ask Codex to challenge the design itself — assumptions, alternatives, tradeoffs, "would this fail in production".

Also escalate to adversarial (even in diff-review mode) when the diff crosses any of these lines:

- New subsystem or service module (not a modification)
- New DB migration
- Auth / permission surface changes
- Cross-service integration (touches two or more external systems at once)
- Feature flag or dark rollout added
- Deleted data flows / anything irreversible

Prompt template — keep it short. Codex reads the diff directly, do not paraphrase it back:

```
Review the working-tree diff on this branch. Please focus specifically on:
- <specific concern 1 — e.g. concurrency, state ordering, cleanup>
- <specific concern 2 — e.g. auth surface, data exposure>
- <specific concern 3 — e.g. anything the mode escalation above flagged>

Review only, do not fix.
```

Capture Codex's findings verbatim. If the subagent errors out, treat as a skip with the error logged.

### 5.5c. Triage

- **Auto-fixable** — clear wrong → clear right. Fix now, in the working tree, with your (Claude) tokens.
- **Needs decision** — architectural, ambiguous. Add to "Unfixed Findings" with severity, file:line, Codex's reasoning as the "Why not fixed" cell. Tag source as `codex`.
- **False positive** — count them, report `Codex false positives: N`. Don't list each.
- **Conflicts with earlier phase** — defer with `Why not fixed: conflicts with earlier phase decision`.

### 5.5d. Re-verify

If Phase 5.5 made code changes, re-run Phase 4's test command. If a fix regresses tests, revert it and move the finding to "Unfixed" with `Why not fixed: caused regression`.

---

## Phase 6 — Dead Code Removal (diff-scoped)

Ordered *after* bugs/security/tests/docs because it's the least-urgent phase. Run only on the diff's surface, not the whole codebase.

**In diff mode:** flag only dead code *introduced or made-dead by this diff*:

- New exported symbols never imported outside their own file
- Callers removed by this diff whose callee is now unreferenced
- Feature flags / env vars this diff no longer uses
- Commented-out blocks added by this diff (any age)

**In audit mode:** run full-codebase detectors:

| Language | Tool |
|---|---|
| JS/TS | `npx ts-prune` or `npx unimported` |
| Python | `vulture` |
| Rust | `cargo udeps` |
| Go | `go build ./... -gcflags="-e"` |
| Generic | grep-based import tracing |

### Removal rules

- **Never remove** code marked `// TODO: re-enable`, `# noqa: dead`, or comparable "kept intentionally" markers without explicit user approval.
- **Always** remove the import/require statement when you remove the last usage.
- After removing, re-run Phase 4 to confirm nothing broke.

---

## Phase 7 — Summary Report

Render only sections that have content. Empty sections are noise, not signal.

Skeleton:

```
## Code Review Summary

### Project
<name, language, framework — one line>

### Mode
<quick | diff review, N files: … | audit, whole codebase>

### Changes Made
[Only include rows with content. Omit the row entirely if 0.]
- Simplify fixes: <count> — <brief list>
- Bugs fixed: <count> — <brief list>
- Security issues fixed: <count> — <brief list>
- Dead code removed: <count> items — <brief list>
- Codex cross-check fixes: <count> — <brief list>
- Docs updated: <which files, what changed>

### Verification
- Tests: <M/N passing | CI green on {SHA} at {time} | pre-existing failures: …>
- Build: <clean | warnings | failed>

### Security Findings
[Only render the table if there are findings.]
Severity levels: 🔴 CRITICAL | 🟠 HIGH | 🟡 MEDIUM | 🔵 LOW | ℹ️ INFO

| # | Severity | Category | Description | File:Line | Status |
|---|----------|----------|-------------|-----------|--------|

### Unfixed Findings
[Only render if there are unfixed items. Include findings from Claude and Codex, ranked by severity.]

| # | Severity | Source | Category | Description | File:Line | Why not fixed |
|---|----------|--------|----------|-------------|-----------|---------------|

### Recommended Next Steps
<3-5 actionable items, security fixes first — omit section if none>
```

After presenting the report, if there are unfixed findings:

> "There are <N> unfixed findings above. Want me to fix any / all of them?"

If yes, fix and update the report.

---

## Principles

- **Minimal blast radius** — small targeted changes over sweeping rewrites.
- **Verify after each phase** — re-run tests after 2 (bugs) and 3 (security).
- **Explain, don't just change** — one-line comment or a mention in the summary for non-obvious changes.
- **Ask before deleting anything ambiguous** — if unsure whether code is truly dead, ask.
- **Never auto-commit** — present the summary, let the user decide what to stage.
- **Security findings get priority** — list first in the summary.
- **Never suppress security controls** without explicit user approval — even unused, they may be defense-in-depth.
- **Right-size effort to the diff** — a 5-file UI polish PR does not need adversarial Codex review or full-codebase dead-code scans. Match the pipeline to the work.
