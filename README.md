# Claude Code Audit

Three-tier adversarial code review plugin for [Claude Code](https://docs.anthropic.com/en/docs/claude-code). Catches AI slop, enforces code quality, prevents people-pleasing, and verifies CLAUDE.md compliance.

## What it does

When triggered, the plugin runs three sequential tiers:

1. **Tier 1 — Quick Lint Pass:** Detects and runs available project tooling (ESLint, PHPStan, Ruff, gitleaks, etc.) against changed files. Findings feed into Tier 2 as hard evidence.
2. **Tier 2 — Adversarial Audit:** Dispatches a subagent with a "guilty until proven innocent" stance that reviews every changed line against a 13-point checklist.
3. **Tier 3 — Fix Regression Check:** After fixes are applied, verifies the fixes didn't introduce new slop. Loops back to Tier 2 if needed (max 3 iterations).

## The 13-Point Checklist

**Anti-slop detection:**
1. Unjustified abstractions
2. Over-commenting
3. Defensive over-engineering
4. People-pleasing patterns
5. Scope creep
6. Verbose where concise works
7. Generic implementations
8. Phantom requirements
9. Style inconsistency

**Code quality assurance:**
10. Security (always CRITICAL)
11. Performance
12. Readability

**Project rules compliance:**
13. CLAUDE.md / AGENTS.md violations

## Installation

Requires [Claude Code](https://docs.anthropic.com/en/docs/claude-code) and the [superpowers](https://github.com/obra/superpowers) plugin.

```bash
# Add the marketplace
/plugin marketplace add https://github.com/Glubiz/claude-code-audit.git

# Install the plugin
/plugin install claude-code-audit@claude-code-audit

# Activate
/reload-plugins
```

## Usage

### On-demand
```
/audit
```
Runs the full three-tier audit against your current uncommitted changes. If no changes exist, it offers to audit the last commit.

### Automatic triggers

The plugin also triggers automatically:
- **After code review** — when the `superpowers:code-reviewer` subagent completes
- **Before finishing a branch** — as a gate before merge/PR via `superpowers:finishing-a-development-branch`

### Options

- `/audit --skip` — Bypass a FAIL gate (recorded in state)
- `/audit --last-commit` — Audit the last commit when no uncommitted changes exist
- `/audit --with-ci` — Also run matching CI quality jobs via `gitlab-ci-local` (opt-in)

## Output

The audit produces structured, machine-parseable output:

```
## Audit Result

**verdict:** FAIL
**blocking_issues:** 1
**total_issues:** 4

### CRITICAL
- file: src/auth.ts:42
  issue: SQL injection via unsanitized user input
  action: Use parameterized query instead of string concatenation
  category: security

### IMPORTANT
- file: src/utils.ts:10-15
  issue: formatDate helper has single caller
  action: Inline into src/components/Header.ts:23
  category: unjustified-abstraction

### NEXT_STEPS
1. Fix SQL injection in auth.ts
2. Inline formatDate helper
```

**Verdicts:**
- `PASS` — No issues found
- `PASS_WITH_WARNINGS` — IMPORTANT/MINOR issues only (advisory, does not block)
- `FAIL` — CRITICAL issues found (blocks progress until fixed)

## How it integrates with superpowers

This plugin complements, not replaces, the superpowers `code-reviewer`. The code-reviewer focuses on correctness and architecture. This audit focuses on slop detection and quality enforcement. They run sequentially — code-reviewer first, then this audit.

## License

MIT
