---
name: weekly-mrr-digest
description: "Pull Stripe + Mixpanel + GA4 once a week, compute MRR / churn / ARPU / activation, send a digest to Slack with anomalies flagged."
version: 1.0.0
metadata:
  hermes:
    tags: [analytics, saas, mrr, stripe, mixpanel, slack, founder]
---

# Weekly MRR Digest

A weekly metrics report for solo founders and small SaaS teams who don't have a data analyst yet. Pulls Stripe, Mixpanel (or Amplitude / PostHog), and GA4, computes the core SaaS numbers, runs anomaly detection on each one, and sends a plain-English digest to Slack every Monday at 9am.

## What this skill does

Every Monday at 9am (or whenever you schedule it):

1. **Pull from data sources** — Stripe (subscriptions, invoices, refunds), Mixpanel/Amplitude/PostHog (events, cohorts), GA4 (traffic, attribution).
2. **Compute the core SaaS metrics:**
   - MRR (current week, +/- vs last week, MoM)
   - Net new MRR (new subs minus churned)
   - Churn rate (last 30d, last 7d)
   - ARPU
   - Signup → paid conversion (last 14d, last 90d trailing)
   - Activation rate (you define what activation means in the config)
   - Top traffic sources by signups, by paid conversion
3. **Run anomaly detection** — z-score against trailing 30-day baseline. Flag any metric 2σ+ off in either direction.
4. **Add narrative context** — "MRR dipped because 3 annual subs churned this week. Two were on the deprecated $19 tier. The third left a feedback note (link)."
5. **Surface 1-3 action items** — based on what's anomalous, suggest the most leveraged thing to investigate this week.
6. **Send to Slack** — to the channel you specify, formatted as Slack mrkdwn with metric blocks and a TL;DR at the top.

## What this skill does NOT do

- It does NOT replace a real data analyst — for ad-hoc questions ("why did churn spike in Brazil?"), you still need someone who can write SQL.
- It does NOT build dashboards. For real-time visibility, use Mixpanel/Amplitude's native dashboards or pair with a separate dashboard skill.
- It does NOT predict future MRR. Forecasting is a separate problem; this skill is descriptive analytics, not predictive.
- It does NOT replace Stripe Sigma for deep financial analysis.

## How to invoke

Scheduled (recommended): use Hermes' built-in cron.

```
/cron weekly "every Monday at 9am, run /weekly-mrr-digest and post to #standups"
```

Manual run:

```
/weekly-mrr-digest
```

Or with custom date range:

```
/weekly-mrr-digest --since 2026-04-01 --until 2026-04-30
```

## Configuration

```yaml
data_sources:
  stripe:
    api_key_env: STRIPE_SECRET_KEY
    include_test_mode: false
  mixpanel:
    project_id: "12345"
    api_secret_env: MIXPANEL_API_SECRET
  ga4:
    property_id: "987654321"
    service_account_json_env: GA4_SA_JSON
slack:
  channel: "#metrics"
  bot_token_env: SLACK_BOT_TOKEN
metrics:
  activation_event: "first_query_run"   # what counts as "activated"
  trial_period_days: 7
  baseline_window_days: 30
  anomaly_z_score: 2.0
schedule:
  cron: "0 9 * * 1"   # Monday 9am
  timezone: "America/New_York"
```

## Models

This is mostly structured queries with narrative summarization on top:

- **Recommended:** Claude Sonnet 4.6 for the narrative + anomaly explanations. The numbers come from APIs; the model writes the digest text.
- **For raw query work:** Haiku 4.5 is fine and ~10x cheaper.
- **Total weekly cost:** ~$0.40-1.20 in API spend for a typical solo SaaS at $1K-50K MRR (mostly the narrative pass).

## Sample digest output

```
📊 Weekly MRR Digest — Week of 2026-04-29

TL;DR: MRR up 1.8% WoW. Anomaly: signup spike Tuesday (+47% vs baseline)
likely from a HN front-page mention. Investigate: activation dropped 14
percentage points after the Apr 24 deploy.

MRR: $3,847 (+1.8% WoW, +12% MoM)
  • Net new: +$46 (3 new $216, 2 churned $170)
  • Active subs: 247

Conversion (signup → paid, trailing 14d): 4.2%
  • Industry SaaS median at your ARR: 2-5%, you're median+

Churn (last 30d monthly): 5.4%
  • Industry SaaS 60-75th percentile, top quartile <4%

🚨 Anomaly: Activation rate dropped from 38% → 24% over the past 5 days.
  • Likely cause: deploy on Apr 24 changed the onboarding flow.
  • Recommend: rollback or fix before more cohort flows through.

Top action item: investigate activation drop before more users hit it.
```

## Pairing

This skill is the analyst piece of a 3-agent data crew. Pair with a dashboard-builder skill (coming soon) and a market-context skill (coming soon) for full coverage.

For the full crew: see [crewclaw.com/use-cases/data-analytics-team](https://crewclaw.com/use-cases/data-analytics-team).

## Failure modes to watch

- **API rate limits:** Mixpanel raw export rate-limits at 60 req/hour. The skill batches sensibly, but for very high event volumes, set `mixpanel.use_jql: true` (faster) or run from a VPS in the same region.
- **Time-zone confusion:** all metrics computed in the timezone you configure. If your Stripe data is UTC and your Slack channel expects local time, the digest will note the mismatch but won't auto-convert.
- **Cohort definition drift:** if you change `activation_event` after running for 2+ weeks, historical baselines will look wrong. Reset the baseline window after any definition change.
