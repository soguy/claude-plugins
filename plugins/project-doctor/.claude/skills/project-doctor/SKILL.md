---
name: project-doctor
description: "Diagnoses and heals software projects end-to-end: dead code removal, bug detection, security audit, build/test sanity checks, and documentation sync. Use for code review, cleanup, audit, refactoring, or any codebase health task."
---

# Project Doctor Skill

A structured pipeline for analyzing a project, removing dead code, fixing bugs, auditing security, running sanity checks, and keeping documentation in sync with reality.

---

## Phase 0 — Orient

Before doing anything else, understand the project:

1. **Read `view /path/to/project`** to see the directory tree.
2. Look for `README.md`, `CHANGELOG.md`, `package.json` / `pyproject.toml` / `Cargo.toml` / equivalent to understand the project's purpose, stack, and entry points.
3. Identify the primary language(s), framework(s), and test runner.
4. Note any CI config (`.github/workflows`, `Makefile`, `Justfile`) — these reveal the canonical build/test/lint commands.

Summarize your findings to the user in 3–5 bullet points, then proceed immediately to Phase 0.5.

---

## Phase 0.5 — Simplify

Before deeper analysis, run the `simplify` skill on any recently changed code. This catches reuse, quality, and efficiency issues introduced by recent work before they get baked in further.

Invoke: use the `Skill` tool with `skill: "simplify"`.

When simplify completes, **do not pause or wait for user input** — proceed immediately to Phase 1.

---

## Phase 1 — Dead Code Removal

Goal: delete code that is never reached, never imported, or never called.

### 1a. Automated detection
Run language-appropriate static analysis tools if available:

| Language | Tool | Install check |
|---|---|---|
| JavaScript / TypeScript | `npx ts-prune` or `npx unimported` | `which npx` |
| Python | `vulture` | `pip show vulture` |
| Rust | `cargo udeps` | `cargo udeps --help` |
| Go | built-in: `go build ./...` with `-gcflags="-e"` | — |
| Generic | `grep`-based import tracing | always available |

If a tool isn't installed, note it but don't block — proceed manually.

### 1b. Manual patterns to check
- Exported symbols never imported outside their own file
- Commented-out blocks > 5 lines old (check git blame if `.git` exists)
- Feature flags / env vars that no longer exist in any config or CI
- Test helpers imported only in files that no longer exist
- Duplicate utility functions (same logic, different names)

### 1c. Removal rules
- **Never remove** code marked `// TODO: re-enable` or `# noqa: dead` unless the user explicitly says so.
- **Always** remove the import/require statement when you remove the last usage of a symbol.
- After removing, re-run the build/test command (see Phase 3) to confirm nothing broke.

---

## Phase 2 — Bug Detection & Fixes

### High-priority patterns to look for
1. **Off-by-one errors** — loop bounds, slice indices, pagination offsets
2. **Null / undefined dereference** — accessing properties without guards
3. **Async errors not caught** — `await` inside `try` without `catch`, unhandled Promise rejections
4. **Resource leaks** — file handles, DB connections, event listeners never cleaned up
5. **Race conditions** — shared mutable state modified in concurrent paths
6. **Incorrect error propagation** — swallowed errors (`catch (e) {}`), wrong HTTP status codes returned
7. **Type coercion surprises** — `==` vs `===`, implicit string/number conversion
8. **Hardcoded secrets or localhost URLs** — flag these; don't auto-fix, just report
9. **Missing input validation** — user-supplied data passed directly to DB queries, shell commands, or file paths
10. **Logic inversion** — `if (!error) { throw }` style mistakes

### Fix approach
- Fix bugs that are unambiguous (clear wrong behavior, clear correct behavior).
- For ambiguous cases, add a `// FIXME(review): <description>` comment and include it in the summary report.
- Do not refactor working code just because you disagree with the style — that's a separate task.

---

## Phase 3 — Security Audit

Goal: identify vulnerabilities, insecure patterns, and security misconfigurations. Focus on issues from the OWASP Top 10 and common language-specific pitfalls.

### 3a. Secrets & credentials
- **Hardcoded secrets** — API keys, passwords, tokens, connection strings in source code (not just `.env`)
- **Secrets in version control** — check `.gitignore` covers `.env`, `*.pem`, `*.key`, credential files. Run `git log --all -p -- '*.env' '*.key' '*.pem'` to check if secrets were ever committed.
- **Overly broad `.env` exposure** — environment variables containing secrets loaded into frontend bundles or logged to stdout
- **Default credentials** — admin/admin, test tokens, placeholder API keys shipped in config

### 3b. Injection vulnerabilities
- **SQL injection** — raw string concatenation in queries instead of parameterized/prepared statements
- **Command injection** — user input passed to `os.system()`, `subprocess.run(shell=True)`, `child_process.exec()`, or backtick execution without sanitization
- **Path traversal** — user-supplied filenames used with `open()`, `fs.readFile()`, or `os.path.join()` without validating they stay within expected directories
- **Template injection** — user input rendered in server-side templates without escaping
- **XSS** — user-supplied data rendered in HTML/JSX without proper escaping
- **Log injection** — user input written to log files without sanitization

### 3c. Authentication & authorization
- **Missing auth on endpoints** — API routes that should require authentication but don't
- **Broken access control** — endpoints that check auth but not authorization
- **Insecure session handling** — session tokens in URLs, missing `HttpOnly`/`Secure`/`SameSite` cookie flags, no session expiry
- **Weak password handling** — plaintext storage, weak hashing (MD5, SHA1), missing salt, no rate limiting on login
- **JWT issues** — `alg: none` accepted, secrets in code, no expiry, overly broad claims

### 3d. Data exposure & privacy
- **Sensitive data in API responses** — passwords, tokens, internal IDs, PII leaked in output
- **Verbose error messages** — stack traces, SQL errors, or internal paths exposed to users in production
- **Missing CORS restrictions** — wildcard `*` origin in production
- **Unencrypted sensitive data** — PII or credentials stored without encryption at rest

### 3e. Dependency & configuration security
- **Known vulnerable dependencies** — run `npm audit` / `pip audit` / `cargo audit` if available
- **Debug mode in production** — `DEBUG=true`, `FLASK_ENV=development`, verbose logging enabled by default
- **Missing security headers** — no CSP, HSTS, X-Frame-Options, X-Content-Type-Options
- **Insecure TLS** — HTTP URLs for APIs, disabled cert verification

### 3f. Language-specific checks

| Language | What to check |
|---|---|
| Python | `eval()`, `exec()`, `pickle.loads()` on untrusted data, `yaml.load()` without `Loader=SafeLoader`, `subprocess` with `shell=True` |
| JavaScript / TypeScript | `eval()`, `Function()` constructor, `innerHTML`, prototype pollution, `require()` with dynamic paths |
| Go | `fmt.Sprintf` in SQL queries, unchecked `err` returns, `unsafe` package usage |
| Rust | `unsafe` blocks, `.unwrap()` on user input, `std::process::Command` with unsanitized args |
| Ruby | `send()`/`public_send()` with user input, `ERB` without escaping, `YAML.load` on untrusted data |

### 3g. Fix approach
- **Auto-fix** clear-cut issues: replace `shell=True` with argument lists, add parameterized queries, set `Secure`/`HttpOnly` cookie flags.
- **Flag but don't auto-fix** issues requiring architectural decisions. Mark with `// SECURITY(review): <description>`.
- **Never remove security controls** without asking the user first.
- **Rotate exposed secrets** — if you find a secret committed to git history, flag it as **HIGH** severity and recommend rotation.

---

## Phase 4 — Sanity Checks

Run the project's own verification suite. Adapt commands to what exists:

```bash
# Try each in order, stop at first that works:
npm test          # Node / JS
npm run lint
npx tsc --noEmit  # TypeScript type check

python -m pytest  # Python
python -m mypy .

cargo test        # Rust
cargo clippy -- -D warnings

go test ./...     # Go
go vet ./...

make test         # Generic Makefile
just test         # Justfile
```

Record output. If tests fail:
1. Determine if the failure is pre-existing or caused by Phase 1–3 changes.
2. Fix failures caused by your own changes first.
3. For pre-existing failures, list them in the summary report as "Pre-existing failures."

If no test runner exists, note this as a finding and suggest adding one.

---

## Phase 5 — Documentation Updates

Goal: make docs reflect the current state of the code. **This phase is mandatory — update stale documentation directly, do not merely flag issues.**

### 5a. README.md
Check every claim against reality: listed features, installation steps, API/CLI usage examples, environment variables, badges. **Fix all stale content in-place.**

### 5b. All project documentation files
Scan for `.md` files and `docs/` directories. For each:
- Verify every factual claim against the current codebase
- **Update any stale content directly**
- Remove documentation for features that no longer exist
- Add documentation for features that exist in code but aren't documented

### 5c. Inline documentation
- Update or add docstrings/JSDoc for functions changed in Phase 2–3
- Remove doc comments for code deleted in Phase 1
- Fix `@param` / `:type` annotations that no longer match actual signatures

### 5d. CHANGELOG / release notes
If `CHANGELOG.md` exists, add an entry under `## Unreleased` using Keep a Changelog format:
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

## Phase 6 — Summary Report

Produce a structured report in the conversation. Do **not** write it to a file unless the user asks.

```
## Code Review Summary

### Project
<name, language, framework — one line>

### Changes Made
- **Dead code removed**: <count> items — <brief list>
- **Bugs fixed**: <count> — <brief list>
- **Security issues fixed**: <count> — <brief list>
- **Docs updated**: <which files, what changed>

### Security Findings
Severity levels: 🔴 CRITICAL | 🟠 HIGH | 🟡 MEDIUM | 🔵 LOW | ℹ️ INFO

| # | Severity | Category | Description | File:Line | Status |
|---|----------|----------|-------------|-----------|--------|

### Other Findings (not auto-fixed)
- FIXME items: <list with file:line>
- SECURITY(review) items: <list with file:line>
- Pre-existing test failures: <list>
- Missing tooling: <e.g., "no test suite found">

### Unfixed Findings
List every finding that was **not** fixed — regardless of reason (low severity, needs architectural decision, ambiguous, out of scope). For each:

| # | Severity | Category | Description | File:Line | Why not fixed |
|---|----------|----------|-------------|-----------|---------------|
| 1 | 🔵 | Config | Debug mode enabled by default | settings.py:5 | Low priority |
| 2 | 🟡 | Auth | No rate limiting on login | auth.py:80 | Architectural decision needed |

After presenting the full report, ask the user:
> "There are <N> unfixed findings above. Would you like me to fix any or all of them?"

If the user says yes (to all or some), fix them now and update the report.

### Recommended Next Steps
<3–5 actionable items, security fixes first>
```

---

## Principles

- **Minimal blast radius** — prefer small, targeted changes over sweeping rewrites.
- **Verify after each phase** — run the build/test after Phase 1 and again after Phase 2.
- **Ask before deleting anything ambiguous** — if you're unsure whether code is truly dead, ask.
- **Never auto-commit** — present the summary, let the user decide what to stage.
- **Security findings get priority** — list security issues first in the summary report.
- **Never suppress security controls** — don't weaken auth, validation, or sanitization without explicit user approval.
