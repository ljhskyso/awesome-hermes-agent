---
name: competitor-landing-watch
description: "Daily diff of competitor landing pages — pricing changes, hero copy swaps, new features mentioned. Flags meaningful changes only."
version: 1.0.0
metadata:
  hermes:
    tags: [competitive-intelligence, monitoring, sales, marketing]
---

# Competitor Landing Watch

Daily snapshot + diff of competitor landing pages. Catches pricing changes the same day they happen, new feature claims, hero copy swaps, and quiet pages that go silent. Filters out the noise (deploy hashes, JS asset URLs, copyright year bumps) so you only see signal.

## What this skill does

Once a day (default 6am):

1. **Fetch each competitor URL** in your watchlist.
2. **Extract the meaningful content** — strip JS bundle hashes, ignore Cloudflare/CDN footer text, normalize whitespace, keep `<h1>`-`<h3>` headings, hero/sub-hero copy, pricing blocks, feature lists, CTA button text.
3. **Diff against yesterday's snapshot** — semantic diff, not character-level. "Pro plan went from $49 to $59" beats "12 chars changed."
4. **Classify each change** by importance:
   - **High** — pricing, plan structure, new feature claim, removed feature, CTA change
   - **Medium** — hero copy, social proof additions, new logos
   - **Low** — minor wording, marketing copy nudges
5. **Surface only High + Medium** in the daily digest. Low changes are logged but suppressed.
6. **Post to Slack/Discord/Telegram** — formatted as a per-competitor section with the change quoted and a link to the page.
7. **Weekly rollup** — every Friday, a "what shifted this week across all competitors" summary.

## What this skill does NOT do

- It does NOT scrape login-gated pages. If a competitor's pricing is behind a "Talk to sales" wall, this skill can't see it.
- It does NOT do trademark monitoring or aggressive content scraping at scale. Default config respects robots.txt and waits 2 seconds between fetches.
- It does NOT replace a real CI tool like Crayon or Klue for enterprise teams. This is the lightweight version for solo founders and 2-person sales teams.
- It does NOT analyze your own product positioning — it watches competitors. Use a separate skill for self-audit.

## How to invoke

Scheduled (recommended):

```
/cron daily "every day at 6am, run /competitor-landing-watch"
```

Manual run on a specific URL:

```
/competitor-landing-watch https://competitor.com/pricing
```

Manual run on the full watchlist:

```
/competitor-landing-watch --all
```

## Configuration

```yaml
watchlist:
  - name: "Competitor A"
    urls:
      - "https://competitora.com"
      - "https://competitora.com/pricing"
      - "https://competitora.com/features"
  - name: "Competitor B"
    urls:
      - "https://competitorb.com"
      - "https://competitorb.com/pricing"
delivery:
  channel: "#competitive-intel"
  platform: "slack"   # or "discord", "telegram", "email"
fetch:
  delay_seconds: 2
  user_agent: "CompetitorWatch/1.0 (+contact@yourdomain.com)"
  respect_robots_txt: true
  timeout_seconds: 30
diff:
  importance_threshold: "medium"   # send only medium+ changes
  ignored_patterns:
    - "Copyright (c) \\d{4}"
    - "v[0-9]+\\.[0-9]+\\.[0-9]+"
    - "build-[a-f0-9]{7,}"
storage:
  snapshot_dir: "~/.hermes/skills/competitor-landing-watch/snapshots"
  retain_days: 90
```

## Models

The classification is pattern-aware but the importance ranking benefits from real reasoning:

- **Recommended:** Claude Sonnet 4.6 or GPT-5.5 standard for the diff classifier. It has to decide "is this change strategically meaningful?" — that's nuance.
- **Cost-optimized:** Haiku 4.5 for the per-page extraction (just HTML → structured content), Sonnet only for the diff scoring.
- **Total cost:** typically $0.20-0.50/day for a 5-competitor watchlist.

## Sample output

```
🔍 Competitor Landing Watch — 2026-04-30

🚨 Competitor A
  Page: /pricing
  Change (HIGH): Pro plan price $49 → $59 (+$10, +20%)
  Change (HIGH): "Free trial" copy removed; CTA changed to "Book demo"
  Likely signal: moving up-market, exiting self-serve. Worth a sales call
  to any of your prospects who were comparing against them.

📍 Competitor B
  Page: /
  Change (MEDIUM): Hero swapped from "AI for everything" to "AI sales
  agents that book meetings". Tightened positioning.
  Page: /features
  Change (LOW, suppressed): minor copy edits across 3 feature blurbs.

✅ Competitor C — no changes today.
✅ Competitor D — no changes today.
```

## Failure modes to watch

- **JavaScript-rendered pages:** if a competitor's page is fully client-rendered (React SPA with no SSR), the default `fetch` won't see content. Set `fetch.use_browser: true` to use a headless Chrome via Playwright — slower and uses more compute.
- **Aggressive bot blocks:** Cloudflare-protected pages may serve a challenge. The skill respects this and falls back to "fetch failed, retry tomorrow." Don't try to bypass it — it'll get your IP banned.
- **Too many false positives:** if the skill reports daily changes that are actually just your own perception drift, raise `importance_threshold` to `high` or add patterns to `ignored_patterns`.

## Pairing

Pairs well with the [`closer`](../../personalities/closer.md) personality for sales teams who want competitive context attached to outreach. Also pairs with [`cold-email-personalize`](../cold-email-personalize/) — when a competitor changes pricing, that's a hook for outbound to their prospects.

For a full competitive intelligence crew (watch + research + market brief), see [crewclaw.com/use-cases/competitor-intelligence](https://crewclaw.com/use-cases/competitor-intelligence).
