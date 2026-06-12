---
name: full-checkup
description: >
  Run a full project health audit — security, correctness, complexity, data-flow,
  tech debt, deployment readiness, and live verification (tests + booting the app) —
  producing one consolidated report with a ship / fix-first / no-ship verdict.
  Use this skill whenever the user asks for a checkup, audit, deep review, security
  sweep, pre-deploy or production-readiness check, "is this safe to ship/deploy",
  "review my whole repo", or invokes /full-checkup. Accepts a scope argument
  (diff | full) and an optional --fix flag that applies only safe low-risk cleanups.
argument-hint: "[diff|full] [--fix]"
---

# Full Checkup

A seven-phase project audit that ends in ONE consolidated markdown report with a clear verdict. Run the phases in order — later phases reuse what earlier ones discover (the route table from Security feeds Pipeline & Flow; everything non-blocking feeds the Tech Debt inventory).

## Arguments

Parse from the invocation text:

- **scope** — `diff` (default) or `full`.
  - `diff`: audit pending work only — uncommitted changes plus commits on this branch that aren't on the main branch (`git status`, `git diff`, `git diff <main>...HEAD`). Also pull in files the changed code directly calls, otherwise broken contracts get missed.
  - `full`: the whole repository.
- **--fix** — after the report is delivered, apply safe cleanups (see "--fix mode" below). Without it, the entire run is read-only.

## Ground rules

These keep the checkup safe to run on any repo at any time:

1. **Read-only unless `--fix` was passed.** Never commit, never push — even in fix mode.
2. **Project rules first.** Before auditing, read CLAUDE.md, .claude/rules/, README, and docs. Project-specific decisions override this skill's generic checklist — e.g., a route documented as intentionally public is "public by design", not a missing-auth finding. The user may also supply project context in the invocation; treat that the same way.
3. **Detect the stack from the repo** (manifests, lockfiles, framework configs). Don't assume.
4. **Verify commands exist before running them** (`<tool> --version`) and prefer the project's own documented test/run commands over guessed ones.
5. **Own your processes.** Any server or background job you start, you kill before finishing — even if the checkup errors out halfway. Confirm the port is released.
6. **Write for a non-expert.** Every finding explains why it matters in plain English; jargon only where it names the exact fix.

## Phase 0 — Orient

Map the repo before judging it: entry points, framework, database, how the frontend talks to the backend, where tests live, how the app runs. Build the file list for the chosen scope. Skim the dependency manifest for anything unusual. Note the main branch name.

## Phase 1 — Security (blockers first)

Work top-down — the early items are the ones that kill a deploy:

1. **Secrets**: hardcoded API keys, passwords, tokens, connection strings in code or config. Also spot-check git history (`git log -p` on config files, `git log --all --diff-filter=D -- .env*`) for secrets committed then removed — removal doesn't un-leak them.
2. **Auth & sessions**: how sessions/tokens are issued, validated, expired, revoked.
3. **Endpoint coverage**: build a route table — method, path, auth required?, role/scope enforced?. Every endpoint must enforce login AND the right permission scope unless documented as public by design. Keep the table; Phases 4 and 7 reuse it.
4. **Input validation** at every boundary: request bodies, query/path params, file uploads, webhook payloads, anything user-controlled.
5. **Injection trio**: SQL injection (string-concatenated queries), XSS (user content rendered unescaped into HTML/JS), path traversal (file paths built from user input in upload/download handlers).
6. **Leaky errors**: responses exposing stack traces, SQL, internal paths, library versions.
7. **Rate limiting** on login, signup, password reset, and expensive or public endpoints.

## Phase 2 — Correctness & anomalies

Logic bugs; unhandled errors (bare except / empty catch, missing await, unchecked return values); race conditions (check-then-act on shared state, concurrent writes to the same row/file); edge cases (null/empty/duplicate inputs, off-by-one, timezone); **backend↔frontend contract mismatches** — compare what each API endpoint actually returns against the fields the frontend code reads; dead or unreachable code.

## Phase 3 — Complexity & simplification

Flag: functions over ~50 lines, files over ~800 lines, nesting deeper than 4 levels, duplicated logic that should be one shared function, premature abstraction (interfaces with one implementation, config for things that never vary), unused code/config/dependencies. For each, note whether it's worth fixing now or just recording — this feeds Phase 5.

## Phase 4 — Pipeline & flow

Identify the product's 2–4 core data flows (e.g., "incoming message → processing → reply out", "signup → activation") and trace each end to end. Flag:

- **Dead ends** — data written that nothing ever reads.
- **Missing failure handling** — what actually happens when the external call / send / job fails? If the answer is "nothing, silently", that's a finding.
- **Stuck states** — status fields or queues with an entry transition but no exit: no retry, no timeout, no dead-letter, no way back.
- **Invisible breakage** — caught-and-ignored exceptions, failed sends never surfaced to anyone, scheduler jobs that error without logging or alerting.

## Phase 5 — Tech debt inventory

Aggregate every non-blocking finding into one refactor list ranked by **impact vs effort** — high-impact/low-effort first. Each entry: what, where, why it matters, rough effort (S/M/L).

## Phase 6 — Deployment readiness

- Required env vars validated at startup (fail fast, not at first use).
- DB migrations idempotent and safe against an existing production database (re-runnable, additive, no destructive defaults).
- `.gitignore` covers secrets, local databases, uploaded/user files.
- No debug prints / console.log / verbose dev logging left in production paths.
- Cookies (Secure, HttpOnly, SameSite), CORS, and HTTPS assumptions production-ready.
- Dependencies pinned (lockfile or exact versions).
- Write a short **rollback note**: how to undo this deploy if it goes wrong.
- Write a **"verify after first deploy" checklist**: 5–10 concrete checks a human can run.

## Phase 7 — Verification (live)

1. Run the test suite exactly as the project documents it. Report counts; paste failures verbatim.
2. If the app has a server: boot it the documented way, wait for readiness, then hit the health endpoint plus 2–4 key endpoints. A protected endpoint returning 401/403 without credentials is a PASS — that means auth is working. Record actual status codes.
3. Kill every process you started and confirm the port is released. Do this even if earlier steps failed.

## Report format

Produce exactly ONE markdown report:

```
# Full Checkup — <project> (<scope> scope, <date>)

## Executive summary
**Verdict: SHIP | FIX FIRST | NO-SHIP** — one short paragraph explaining why.
Counts: X CRITICAL · X HIGH · X MEDIUM · X LOW. Tests: <pass/fail>. Server: <boots / doesn't>.

## 1. Security
## 2. Correctness & anomalies
## 3. Complexity & simplification
## 4. Pipeline & flow
## 5. Tech debt inventory
## 6. Deployment readiness
## 7. Verification results

## Prioritized action list
```

Every finding uses this shape:

- **[CRITICAL|HIGH|MEDIUM|LOW] Short title** — `path/to/file.py:123`
  *Why it matters:* one or two plain-English sentences about the real-world consequence.
  *Fix:* the concrete change to make.

Verdict rules: any CRITICAL ⇒ **NO-SHIP** (block deploy). Highest severity HIGH ⇒ **FIX FIRST**. Only MEDIUM/LOW ⇒ **SHIP** with notes. The prioritized action list orders CRITICAL → HIGH → quick wins.

## --fix mode

Only after the full report is delivered:

1. Sort candidate fixes into two groups: **"Applying now"** — LOW/MEDIUM, mechanical, zero behavior change (dead code, unused imports, debug prints, trivially safe dedup) — and **"Needs your approval"** — everything security-related or behavior-changing, which is listed but never applied automatically.
2. Apply the first group, file by file.
3. Re-run the test suite to prove nothing broke; revert any change that breaks a test.
4. Summarize what changed. Leave everything uncommitted.
