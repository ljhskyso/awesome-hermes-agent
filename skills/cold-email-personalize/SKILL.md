---
name: cold-email-personalize
description: "Write signal-driven first lines for cold outreach. Leads with a recent hire, product launch, podcast quote, or funding round. Avoids generic 'I noticed your company...' openers."
version: 1.0.0
metadata:
  hermes:
    tags: [sales, outreach, cold-email, personalization, founder]
---

# Cold Email Personalize

Write the opening line of a cold email that doesn't sound like a cold email. The skill takes a prospect (LinkedIn URL, company domain, or just name + role) and writes a 4-line first message that hooks on a specific signal — a recent hire, a product launch, a podcast quote, a funding round, a public tweet — then asks one clear question.

## What this skill does

For each prospect:

1. **Enrich the prospect** — pull from the input (LinkedIn URL, company domain, name+role) and supplement with public signals: recent hires (LinkedIn jobs page), product launches (Product Hunt / changelogs / blog), podcast appearances, funding announcements (Crunchbase / Twitter), recent tweets, GitHub activity if technical role.
2. **Pick the strongest signal** — rank the signals by recency and relevance. The strongest is usually a public commitment (just hired, just launched, just funded) within the last 14 days.
3. **Write a 4-line message:**
   - Line 1: signal-driven hook (what they did, when, why it caught your eye)
   - Line 2: one-sentence relevance bridge (why your product matters given that signal)
   - Line 3: concrete result you've seen for similar companies (with a number, not a vibe)
   - Line 4: one specific question, not "Are you free for a 15-min call?"
4. **Refuse generic openers** — the SKILL.md is configured to reject "I noticed your company...", "I came across [name]...", "I see you're working on...", "Hope you're doing well." If the available signal is too weak, the skill says so explicitly instead of writing weak copy.
5. **Output** — markdown with the email body + a one-line "why this signal" note for your records.

## What this skill does NOT do

- It does NOT send the email. Delivery is your call. Pair with a sending tool (Lemlist, Smartlead, Instantly, your own SMTP) and human approval before send.
- It does NOT scrape Sales Navigator at scale (that's a TOS issue). Bring data from Apollo, Clay, or your existing list.
- It does NOT write the follow-up sequence. That's a separate skill (coming soon).
- It does NOT guarantee replies. Reply rate depends on signal quality + offer fit. The skill optimizes the floor, not the ceiling.

## How to invoke

Single prospect:

```
/cold-email-personalize https://linkedin.com/in/jane-prospect
```

Or with structured input:

```
/cold-email-personalize --name "Jane Prospect" --role "Head of RevOps" --company "AcmeCorp" --domain "acme.com"
```

Batch (CSV input):

```
/cold-email-personalize --csv prospects.csv --output drafts.md
```

## Configuration

```yaml
your_pitch:
  product: "CrewClaw"
  one_liner: "pre-built multi-agent crews for solo founders and 2-person sales teams"
  result_proof:
    - "team of 4 closed $42K pipeline in 60 days from cold outbound, no SDR"
    - "reply rate jumped from 1.2% to 6.8% after switching from templated to signal-driven"
signal_priority:
  - "funding_round"           # last 30 days
  - "product_launch"          # last 14 days
  - "leadership_hire"         # last 14 days
  - "podcast_appearance"      # last 30 days, them speaking
  - "public_commitment"       # tweets / blog posts about the problem
  - "github_activity"         # for technical roles
banned_phrases:
  - "I noticed your company"
  - "I came across"
  - "Hope you're doing well"
  - "I'd love to"
  - "quick question"
  - "circle back"
  - "touch base"
weak_signal_handling: "refuse"   # alternative: "warn"
tone: "direct"                   # alternative: "warm"
```

## Models

Cold email quality directly affects reply rate. Junior models produce flat copy.

- **Recommended:** Claude Sonnet 4.6 or Opus 4.7 for the message itself. The accuracy gap on "is this opener actually compelling" is wide.
- **Cost-optimized:** Haiku 4.5 for prospect enrichment (structured fact extraction), Sonnet only for the writing pass.
- **Avoid:** any model under Sonnet-class for the writing step. Reply rates measurably drop.
- **Cost per email:** ~$0.04-0.10 with Sonnet for enrichment + writing combined.

## Sample output

```
PROSPECT: Jane Prospect, Head of RevOps at AcmeCorp
SIGNAL (strongest): Promoted to Head of RevOps 11 days ago (LinkedIn);
  AcmeCorp closed Series A $12M last month (Crunchbase).

DRAFT (Subject: "First 90 days at Acme — quick thought"):

Saw your move into the Head of RevOps seat last week — congrats. With
fresh Series A capital and a sales team scaling, the bottleneck most
new RevOps leads tell me about in their first 90 days is outbound
infrastructure that doesn't exist yet.

We package the four agents that handle the boring half of outbound
(lead scoring, sequence orchestration, signal-driven first lines, CRM
logging) so you can spin it up before the first hire instead of after
the third one.

Two CrewClaw users your size went from 1.2% reply rates on templated
outbound to 6.8% in 60 days, same lead volume. No new SDRs needed.

Worth 12 minutes of your week to walk through the stack on Friday?

— Mustafa
CrewClaw
```

## Pairing

Pairs naturally with the [`closer`](../../personalities/closer.md) personality. For full pipeline (lead scoring + sequence orchestration + first-line writing + CRM), see the Sales Outreach Team bundle: [crewclaw.com/use-cases/sales-outreach-team](https://crewclaw.com/use-cases/sales-outreach-team).

Also pairs with [`competitor-landing-watch`](../competitor-landing-watch/) — when a competitor changes pricing or positioning, that's a fresh signal for your outbound the same day.

## Failure modes to watch

- **Weak signal mode:** if no signal in the last 30 days is strong enough, the skill defaults to `refuse` (no draft). Set `weak_signal_handling: warn` to get a draft anyway with a warning, but reply rates on weak-signal drafts are typically 2-3x lower.
- **Stale enrichment data:** Apollo / Clay / Crunchbase data lags real-world events by 1-7 days. For high-stakes prospects, manually verify the signal (LinkedIn post, public tweet) before sending.
- **Voice drift:** by default the skill mirrors the example in `your_pitch.result_proof` for tone. If your voice is different (more casual, more technical), provide 5-10 examples of your own past winning emails in the SKILL.md `metadata` block and the skill will calibrate.

## Compliance reminder

This skill is configured to comply with CAN-SPAM (US) and GDPR (EU/B2B legitimate interest) by default: it reminds you in the output to include a physical address, unsubscribe path, accurate from-name, and to honor opt-outs. CASL (Canada) requires more care — flag Canadian domains separately if you target them.
