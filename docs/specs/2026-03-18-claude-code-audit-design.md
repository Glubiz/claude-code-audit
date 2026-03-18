# Claude Code Audit — Design Spec

## Overview

A plugin that sits on top of superpowers, providing a three-tier adversarial code review system designed to catch AI slop, enforce code quality, and prevent people-pleasing. Works across any language or stack, with a focus on detecting failure modes specific to AI-generated code.

## Architecture

Three sequential tiers:

```
Tier 1: Quick Lint Pass (tooling)
  ↓ findings feed into
Tier 2: Full Adversarial Audit (subagent)
  ↓ if FAIL → fixes applied → re-audit from Tier 2
  ↓ if PASS
Tier 3: Fix Regression Check (subagent)
  ↓ only runs after fixes were applied
  ↓ reviews ONLY the fix diff
  ↓ if new issues → back to Tier 2 (max 3 loops, then escalate to human)
  ↓ if clean → PASS
```

Escalation rule: max 3 total Tier 2 runs (1 initial + 2 re-runs). After the 3rd Tier 2 run still produces FAIL, stop and surface to human with an escalation report:

```
## Escalation — Audit loop exceeded 3 iterations

**Iteration history:**
- Round 1: [what was found] → [what was attempted]
- Round 2: [what was found] → [what was attempted]
- Round 3: [what was found]

**Recurring issues:**
- [category]: [what keeps failing and why fixes aren't resolving it]

**Recommendation:**
[Suggested approach for the human to resolve the deadlock]
```

## Tier 1 — Quick Lint Pass

Gathers ground truth from real tooling before the AI review.

**Behavior:**
- Detect available tooling by scanning project root for config files
- Run tooling against changed files only
- If no tooling found, skip to Tier 2 with a note
- Collect output as structured findings for Tier 2

**Changed files resolution (varies by trigger):**
- After code-reviewer (integration point 1): `git diff --name-only {base_sha}..HEAD` — same SHAs used by the code-reviewer
- Finish gate (integration point 2): `git diff --name-only {merge_base}..HEAD` where merge_base is the common ancestor with the target branch
- On-demand `/audit` (integration point 3): `git diff --name-only HEAD` to get all uncommitted changes (both staged and unstaged) vs HEAD. If empty, inform user "No uncommitted changes to audit. Run against last commit with `/audit --last-commit`?" and wait for confirmation before falling back to `git diff --name-only HEAD~1..HEAD`

**Never blocks on its own.** Output is input for Tier 2.

**Tooling detection map (initial set — structured so adding new tools is a config-level change):**

| Config found | Detection logic | Command |
|---|---|---|
| `.eslintrc*` / `eslint.config.*` | File exists | `npx eslint --no-warn-ignored {changed_files}` |
| `phpstan.neon*` | File exists | `vendor/bin/phpstan analyse {changed_files}` |
| `pyproject.toml` | Contains `[tool.ruff]` section | `ruff check {changed_files}` |
| `pyproject.toml` | Contains `[tool.flake8]` section | `flake8 {changed_files}` |
| `.gitlab-ci.yml` | File exists, user has opted in via `/audit --with-ci` | `gitlab-ci-local --list`, then run jobs whose name contains `lint`, `phpstan`, `eslint`, `analyse`, or `quality`. 60-second timeout per job. This is opt-in only because CI jobs may have side effects — the user takes responsibility for which jobs are safe to run locally. |
| `gitleaks.toml` or `.gitleaks.toml` | File exists (or fallback: check if `gitleaks` is installed) | `gitleaks detect --log-opts="{base_sha}..HEAD"` (uses git log range to scan only changed content) |
| Generic fallback | No config found | Skip, note absence in Tier 2 context |

## Tier 2 — Full Adversarial Audit

The core of the plugin. A single subagent with an explicitly adversarial stance.

**Default position:** "This code is guilty until proven innocent." The subagent must justify why something is acceptable, not why it's problematic.

**Context provided:**
- Git diff of all changes (base SHA to HEAD)
- Tier 1 tooling findings (if any)
- Commit messages (as proxy for task intent)
- Plan file path (if one exists in `docs/superpowers/plans/`)
- Surrounding codebase conventions: for each unique file type in the changed files, sample up to 3 non-test files from the same directory (most recently modified first), max 200 lines each, capped at 5 files total across all types/directories. If changes span many directories, prioritize directories with the most changed files. A file is considered a test file if its path matches: `tests/`, `__tests__/`, `test/`, `spec/` directories, or filename patterns `*_test.*`, `*.test.*`, `*.spec.*`, `test_*.*`.

**The audit checklist:**

### Anti-slop detection
1. **Unjustified abstractions** — helpers with one caller, interfaces with one implementation, unnecessary wrappers
2. **Over-commenting** — comments restating code, docstrings on obvious methods
3. **Defensive over-engineering** — error handling for impossible cases, unnecessary fallbacks, validation of trusted internal data
4. **People-pleasing patterns** — cosmetic changes bundled with real work, "looks productive" edits
5. **Scope creep** — changes to unrelated files, "while I'm here" improvements
6. **Verbose where concise works** — 10 lines doing what 3 could, unnecessary intermediate variables
7. **Generic implementations** — tutorial code pasted into a production codebase, not matching existing patterns
8. **Phantom requirements** — feature flags, config options, extensibility points nobody asked for
9. **Style inconsistency** — new patterns introduced without justification

### Code quality assurance
10. **Security** — injection vectors, auth gaps, unsanitized input, secrets in code
11. **Performance** — unnecessary loops, N+1 queries, missing indexes, unneeded allocations
12. **Readability** — unclear naming, convoluted control flow, deep nesting, missing early returns

**Output format:**
```
## Audit Result

**verdict:** PASS | PASS_WITH_WARNINGS | FAIL
**blocking_issues:** N
**total_issues:** N

### CRITICAL
- file: path:line
  issue: ...
  action: ...
  category: ...

### IMPORTANT
- file: path:line
  issue: ...
  action: ...
  category: ...

### MINOR
- file: path:line
  issue: ...
  action: ...
  category: ...

### NEXT_STEPS
1. ...
```

**Verdict rules:**
- Any CRITICAL → `FAIL` — parent agent fixes CRITICALs and re-triggers audit loop
- Only IMPORTANT/MINOR → `PASS_WITH_WARNINGS` — no automatic fix loop; findings are presented to the parent agent/user as advisory. The parent agent may choose to fix IMPORTANTs but is not required to.
- Nothing found → `PASS`

## Tier 3 — Fix Regression Check

Verifies that fixes didn't introduce new slop.

**Only runs after fixes are applied.** Reviews the fix diff only, not the full changeset.

**Context provided:**
- Original Tier 2 audit report
- Fix diff only
- Same checklist as Tier 2

**Watches for:**
- Fix introduced a new abstraction to "solve" an unnecessary abstraction finding
- Fix significantly increased complexity relative to the issue it addressed (a net-negative trade-off, not simply "more lines" — adding a missing early return is fine even though it adds lines)
- Fix moved the problem instead of eliminating it
- Fix addressed the letter of the finding but not the spirit
- Fix broke something that was working

**Who applies fixes:** The parent orchestrating agent (not the audit subagents). The audit subagent returns findings, the parent agent attempts to fix ALL CRITICAL issues based on the `action` fields before re-triggering. If the parent agent cannot fix all CRITICALs (e.g. doesn't know how, or the action is ambiguous), it must still re-trigger Tier 2 with whatever fixes were applied — Tier 2 will re-evaluate and may downgrade, confirm, or clarify remaining issues. Partial fixes are acceptable; the loop handles convergence.

**Loop logic:**
```
Tier 2 returns FAIL
  → parent agent applies fixes based on action fields
  → Tier 3 reviews fix diff
    → PASS → done
    → FAIL → back to Tier 2 (full diff including fixes)
      → tier2_run_count++
      → if tier2_run_count >= 3: STOP, escalate to human
```

Back to Tier 2 (not Tier 3 again) because after fixes the full changeset has changed shape and needs re-evaluation.

## Integration Points

### 1. After code-reviewer pass (automatic)
After `requesting-code-review` completes, the audit triggers automatically. Code-reviewer focuses on correctness and architecture. Audit focuses on slop and quality. This is the primary automatic trigger — the audit result is recorded against the current HEAD SHA.

### 2. Gate in finishing-a-development-branch (automatic)
Before presenting merge/PR options, checks if an audit has passed for the current HEAD SHA. If yes, uses the existing result (no re-run). If no audit exists for current HEAD (e.g. new commits since last audit, or code-reviewer was skipped), runs a fresh audit. FAIL blocks the finish workflow. PASS_WITH_WARNINGS shows warnings but allows proceeding.

**Override:** If the audit produces a false-positive CRITICAL that cannot be resolved, the user can run `/audit --skip` to bypass the gate with an explicit acknowledgment. The skip is recorded in `.claude-audit-state` with `skipped: true`.

### 3. On-demand via `/audit` (manual)
Runs against current uncommitted + staged changes. If no changes, runs against last commit diff. No automatic re-trigger — user controls the loop.

### Avoiding double-runs
Tracks last audited commit SHA. If HEAD hasn't changed since last audit pass, skips with "Already audited at {sha}."

## Blocking Rules

- CRITICAL → blocks progress, must fix
- IMPORTANT → should fix before merge, does not block
- MINOR → advisory only

## Plugin Structure

```
claude-code-audit/
  .claude-plugin/
    plugin.json
  skills/
    code-audit/
      SKILL.md              # Orchestrates all 3 tiers, contains trigger logic and flow control
      tier2-auditor.md       # Subagent prompt: adversarial audit checklist and output format
      tier3-regression.md    # Subagent prompt: fix regression detection and output format
  agents/
    code-auditor.md          # Agent definition — dispatched by SKILL.md with tier2-auditor.md as prompt
    regression-checker.md    # Agent definition — dispatched by SKILL.md with tier3-regression.md as prompt
```

## Agent Definitions

Agent definition files follow the superpowers convention — YAML frontmatter with name, description, and model:

```yaml
---
name: code-auditor
description: Adversarial code audit subagent — reviews code for AI slop and quality issues
model: inherit
---

[Full prompt content from tier2-auditor.md is pasted here at dispatch time by SKILL.md]
```

The agent `.md` files define the identity and model selection. The prompt content (from `tier2-auditor.md` / `tier3-regression.md`) is injected by SKILL.md when dispatching, following the superpowers pattern of providing full text rather than file references.

## State Persistence

Audit state is persisted in `.claude-audit-state` at the project root (gitignored):

```json
{
  "last_audit": {
    "sha": "abc123",
    "verdict": "PASS_WITH_WARNINGS",
    "timestamp": "2026-03-18T14:30:00Z",
    "skipped": false,
    "trigger": "code-review"
  }
}
```

The `trigger` field records which integration point produced the result (`code-review`, `finish-gate`, `on-demand`). The finish gate (IP2) only accepts results where `sha` matches current HEAD — the trigger type is informational, not a filter.

This file is used for:
- **Double-run prevention:** If `last_audit.sha` matches current HEAD, skip with "Already audited at {sha}"
- **Finish gate check:** Integration point 2 reads this to decide whether to run a fresh audit
- **Skip recording:** When `/audit --skip` is used, `skipped: true` is recorded so the skip is visible

The `.claude-audit-state` file should be added to `.gitignore` — it is local session state, not committed.

## Integration Mechanism

Claude Code plugins cannot directly hook into other plugins' skill execution. Instead, the audit integrates via **skill description triggers** — the same mechanism superpowers uses internally.

**Integration point 1 (after code-review):** The SKILL.md description includes a trigger condition for when code review has just completed. The skill's instructions tell the orchestrating agent to invoke the audit after dispatching the code-reviewer subagent. This works because skills are loaded based on context — when code review output is present in the conversation, the audit skill's trigger condition matches.

**Integration point 2 (finish gate):** The SKILL.md description includes a trigger condition for when the user is about to finish a development branch. The skill instructs the agent to check `.claude-audit-state` for a passing result at current HEAD before proceeding with merge/PR. If no result exists, run the audit first.

**Integration point 3 (on-demand):** The `/audit` command directly invokes the skill.

**Limitation and fallback:** Integration points 1 and 2 depend on Claude recognizing the trigger conditions in conversation context. This is the same mechanism all superpowers skills use — it works reliably when descriptions are well-calibrated, but is not a guaranteed hook. If IP1 fails to trigger (audit doesn't run after code-review), IP2 catches it at finish time — the audit still runs, just later in the workflow. This is acceptable because the audit happens before any code ships. For maximum reliability, users can invoke `/audit` explicitly.
