---
name: github-issue-triage
description: "Triage GitHub issues: label by type/priority, ask for repro on bugs, close obvious duplicates with a link to the canonical thread."
version: 1.0.0
metadata:
  hermes:
    tags: [github, oss, maintainer, triage, issues]
---

# GitHub Issue Triage

Triage incoming GitHub issues so a solo maintainer's inbox doesn't explode. This skill labels new issues, asks for reproduction steps when they're missing, and closes duplicates with a link to the canonical thread.

## What this skill does

For every new issue in a repository:

1. **Read the issue body** — title, description, any code blocks, version mentioned, OS/runtime mentioned.
2. **Classify by type** — apply one of: `bug`, `feature-request`, `docs`, `question`, `support`, `spam`. Use the existing labels on the repo if they exist; don't invent new label names.
3. **Estimate priority** — if the issue mentions data loss, security, crashes, or production blockage → `priority-high`. If it mentions a minor visual glitch or a "nice to have" → `priority-low`. Default → `priority-medium`.
4. **Check for duplicates** — search the last 90 days of closed and open issues for similar titles or error messages. If a duplicate exists, post a comment linking the canonical thread and close the new issue (only if confidence is high).
5. **Ask for repro** — for any `bug` label without clear reproduction steps (steps, expected, actual, version), post a comment asking for them in plain language.
6. **Spam detection** — if the issue is single-line marketing, has no relevance to the repo, or is from a brand-new account with a generic title → `spam` label and close.

## What this skill does NOT do

- It does not auto-merge or auto-fix issues.
- It does not assign issues to specific maintainers.
- It does not reply with code suggestions — only labels, classification, repro requests, and dup-closes.
- It does not auto-close stale issues. Use a separate stale-bot for that.

## How to invoke

Default trigger is the GitHub `issues.opened` webhook. Configure via `hermes webhook list`. For manual run on a single issue:

```
/github-issue-triage owner/repo#42
```

## Configuration

In your agent config or skill metadata:

```yaml
github:
  default_repo: "owner/repo"
  labels:
    bug: "bug"
    feature: "enhancement"
    docs: "documentation"
    spam: "spam"
  dup_search_days: 90
  spam_threshold: 0.85
  ask_repro_template: |
    Thanks for opening this. To reproduce on our end, can you share:
    - The version of {project} you're on
    - Steps to reproduce (1, 2, 3...)
    - What you expected vs what actually happened
    - Your OS / runtime if relevant
```

## Models

This skill is mostly classification + pattern matching:

- **Recommended:** Claude Haiku 4.5 or GPT-5.5 mini class. Cheap, fast, accurate enough for triage.
- **Avoid:** Opus-class models — overkill for the workload, you'll burn budget.
- **Edge cases:** for ambiguous issues (the LLM's confidence is below threshold), the skill is configured to NOT auto-act and instead flag for human review.

## Pairing

This skill works best with the [`triager`](../../personalities/triager.md) personality. Together they handle ~80% of incoming issues without the maintainer touching a keyboard.

For full PR review (not just issues), see the [`pr-first-review`](../pr-first-review/) skill.

## Failure modes to watch

- **Wrongly labeled issues:** the skill defers to existing human labels and won't re-label. If you see consistent miscategorization, tune the SKILL.md classification keys for your repo's terminology.
- **Spam false positives:** the spam detector is conservative (default threshold 0.85). If legitimate issues get spam-closed, raise the threshold to 0.92 and revisit weekly.
- **Repro request loops:** the skill only asks for repro once. If the user doesn't reply within 14 days, a stale-bot should handle it — this skill won't pester.

## Want a coordinated crew (issue triage + PR review + changelog + docs)?

The full GitHub Maintainer team ships as a 4-agent bundle. See [crewclaw.com/use-cases/github-maintainer-team](https://crewclaw.com/use-cases/github-maintainer-team).
