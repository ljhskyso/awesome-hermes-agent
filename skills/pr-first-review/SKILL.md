---
name: pr-first-review
description: "First-pass GitHub PR review: OWASP Top 10, style violations, scope creep, breaking changes. Comments before the maintainer looks."
version: 1.0.0
metadata:
  hermes:
    tags: [github, oss, code-review, security, pr]
---

# PR First Review

A first-pass code review for incoming pull requests. Catches the obvious things — failing tests, security anti-patterns, scope creep, breaking changes — and leaves comments before the maintainer opens the PR. The maintainer's review starts from "is this AI right?" instead of "what does this PR do?".

## What this skill does

For every new PR (or new commit on an existing PR):

1. **Read the diff and the description** — what files changed, what the author claims they did, whether tests were added.
2. **Run the OWASP Top 10 pattern check** on the diff: SQL injection (string concatenation into queries, raw ORM calls), XSS (dangerouslySetInnerHTML, unescaped output), IDOR (resource access without auth checks), SSRF (server-side fetch from user-controlled URLs), broken auth (missing session validation), XXE, mass assignment.
3. **Style and convention check** — does the change match the repo's established style (config file naming, error handling patterns, log format)? Is there a `.eslintrc` / `.prettierrc` / framework convention being violated?
4. **Scope check** — does the PR do what its title/description claims, or did it sneak in unrelated changes? Flag any out-of-scope file in a separate comment.
5. **Breaking-change detection** — function signatures changed? Public API field renamed? Migration script added? If yes and there's no CHANGELOG / version bump, flag it.
6. **Test coverage** — did the PR change behavior without adding/updating a test? Comment with the specific behavior that should have a test.
7. **Block or flag** — if a high-severity issue is found (RCE, exposed secrets, unauthenticated PII endpoint), request changes via the GitHub API. Otherwise, leave comments inline and let the human decide.

## What this skill does NOT do

- It does NOT replace a real static analyzer like Semgrep or CodeQL. Use those alongside for taint tracking and deep dataflow analysis.
- It does NOT auto-merge or auto-approve PRs. Default behavior is comment-only; "request changes" requires explicit configuration.
- It does NOT review architectural decisions or product fit — that's the human's call. It catches mechanical problems.
- It does NOT review every line. The default config skips lockfiles, generated files (`*.min.js`, `dist/`), and snapshot tests.

## How to invoke

Default trigger is the GitHub `pull_request.opened` and `pull_request.synchronize` webhooks. Configure via `hermes webhook list`. For manual run:

```
/pr-first-review owner/repo#312
```

## Configuration

```yaml
github:
  default_repo: "owner/repo"
  block_on_high_severity: false   # set true to actually request changes
  skip_paths:
    - "package-lock.json"
    - "yarn.lock"
    - "pnpm-lock.yaml"
    - "**/*.min.js"
    - "dist/**"
    - "__snapshots__/**"
  severity_thresholds:
    high: ["sql_injection", "xss", "ssrf", "exposed_secret", "broken_auth"]
    medium: ["scope_creep", "missing_test", "style_violation"]
    low: ["typo", "unused_import"]
  comment_style: "constructive"   # alternative: "terse"
```

## Models

PR review needs nuance — a Haiku-class model will miss security-relevant context.

- **Recommended:** Claude Sonnet 4.6 or GPT-5.5 standard. The accuracy gap between Haiku and Sonnet on real PRs is large (~30 percentage points on OWASP catch rate in our testing).
- **For monorepos:** If you review 50+ PRs/month, set up a tiered config — Haiku for the diff summary, Sonnet for the security pass.
- **Avoid:** Opus on every PR is overkill unless the codebase is highly security-sensitive.

## Real-world calibration

Out of the box this skill produces ~10-15% false positives. Calibrate over the first 30 days:

- If it flags too many style issues, narrow the rules in your `.eslintrc` or remove style from `severity_thresholds.medium`.
- If it misses real security issues, run Semgrep alongside and tune the SKILL.md prompts based on what Semgrep caught.
- For repos with unique patterns (Rails mass-assignment, Phoenix contexts, Go interfaces), add repo-specific rules to the SKILL.md `metadata` block.

## Pairing

Pairs naturally with [`github-issue-triage`](../github-issue-triage/) — issues come in, get triaged; PRs go out, get reviewed. Both should run as the same agent under the [`triager`](../../personalities/triager.md) personality.

## Want the full GitHub maintainer crew?

The 4-agent GitHub Maintainer Team ships with this skill plus changelog automation, docs sync, and shared coordination via AGENTS.md. See [crewclaw.com/use-cases/github-maintainer-team](https://crewclaw.com/use-cases/github-maintainer-team).
