---
name: pm-weekly-product-pulse
description: >
  Weekly Slack intelligence report for Product Managers. Scans company-wide public channels and
  DMs over the past 30 days, anchored to a specific product, and delivers a structured Canvas
  covering: top questions, feedback/suggestions, and overall product sentiment/consensus.
  Also surfaces cross-product interaction signals and domain-level discussions relevant to
  the target product. Triggers on phrases like "weekly product pulse", "weekly report",
  "what's the sentiment this week on [product]", "give me my Slack digest", "product feedback
  report for [product]", "what are people asking about [product]", or when a PM asks for a weekly roundup.
---

# PM Weekly Product Pulse

Produces a structured weekly Slack intelligence report anchored to a specific Mollie product.
Covers public channels, DMs, cross-product interactions, and domain-level discussions.

---

## How it works

This skill scans Slack — public channels and your DMs — for the past 30 days, anchored to a specific product you own. It clusters what it finds into three things every PM cares about: what people are asking, what's broken or requested, and what the overall feeling is. The result lands as a Slack Canvas in a channel of your choice.

**First time? You need two decisions:**

1. **Which product** — be specific. "Resell Pricing" works. "Monetisation" is too broad and will produce noise.
2. **Where to post** — pick a channel your team already uses. The skill posts a Canvas link there with a one-line summary.

Once those are set, you can run it on a schedule with `/loop` and never think about it again.

**The feedback loop**

After every report, the skill asks if anything was wrong — misclassified, irrelevant, or too noisy. You answer in plain language:

> *"The API status code discussion isn't relevant to me"*
> *"Don't include engineering-only threads next time"*
> *"That was a suggestion, not a pain point"*

Claude saves your feedback to a `PREFERENCES.md` file inside the skill. Every future run reads that file first and applies your preferences silently — excluded signals are dropped, misclassified ones move to the right section. The skill gets more accurate to your context over time without you having to repeat yourself.

---

## Setup (first time only)

Before this skill works, you need the Slack MCP connected to Claude Code.

**1. Install the Slack plugin**
In Claude Code, run:
```
/plugin install slack
```
Follow the prompts — select **user scope** and complete the OAuth flow. It will guide you through.

**2. Verify it's connected**
You should see `plugin:slack:slack` in your active MCP servers. Your own Slack permissions apply — you can only access channels and DMs you already have access to.

**3. Problems?**
Post in `#helpme-ai` (not `#ai-engineering`).

---

**Tip: run it on a schedule**
Use `/loop` to have Claude run this skill automatically on a recurring interval. Example:

```
/loop 1w /pm-weekly-product-pulse Resell Pricing post in #private-solution-engineering
```

This will re-run the report every week without you having to trigger it manually.

---

## Inputs

### Required: Product Definition

Ask the user upfront if not provided:

> *"Which product should I analyse? Please give me:*
> *1. The product name (e.g. 'Mollie Connect for Platforms', 'Payment Links', 'Subscriptions API')*
> *2. Any related products it interacts with (e.g. 'Business Account', 'Checkout') — I'll flag cross-product signals separately.*
> *3. The domain or team it belongs to (e.g. 'Acquiring', 'Payouts', 'Onboarding') — I'll include relevant domain-level discussions.*

If the user provides only a product name, proceed with that. Infer domain from the product name
when possible (e.g. "Connect for Platforms" → domain: "Platforms / Marketplace"). Ask for
related products only if the topic is broad enough that cross-product signals seem likely.

### Optional inputs

- **Slack channel to post to**: defaults to a DM to the user (ID: U07NVU1C12Q) if not specified.
- **Time window**: defaults to the last 30 days. Accept "last 7 days", "last 2 weeks", "this month", etc.

---

## Privacy Notice

Before running any searches, inform the user:
> "This report will search both public Slack channels and your DMs. It will surface patterns
> and themes — private messages will not be quoted verbatim or attributed to individuals."

Then proceed.

---

## Search Strategy

Compute the `after:YYYY-MM-DD` date from the time window (default: 30 days ago from today).

Run all searches using `slack_search_public_and_private` with `include_context: false` and `limit: 20`.

Organise searches into three tiers. Run all searches in parallel where possible.

---

### Tier 1 — Primary product signals

Target the specific product directly.

**Question signals:**
- `"[product name] ?" after:YYYY-MM-DD`
- `"how does [product name]" OR "how do I [product name]" after:YYYY-MM-DD`
- `"does [product name]" OR "can [product name]" after:YYYY-MM-DD`

**Feedback signals:**
- `"[product name]" bug OR broken OR issue OR error after:YYYY-MM-DD`
- `"[product name]" "feature request" OR "would be nice" OR "wish" OR "improvement" after:YYYY-MM-DD`
- `"[product name]" slow OR confusing OR unclear OR frustrating after:YYYY-MM-DD`
- `"[product name]" love OR great OR "works well" OR "easy to use" after:YYYY-MM-DD`

**General mention sweep:**
- `[product name] after:YYYY-MM-DD` (broad, limit: 20)

---

### Tier 2 — Cross-product interaction signals

For each related product the user listed, run:
- `"[related product]" "[primary product]" after:YYYY-MM-DD`
  — finds messages where both are mentioned together

If no related products were listed, skip Tier 2 and note in the output that cross-product
signals were not scoped.

**Goal:** Identify where conversations about a related product surface friction or dependency
with the primary product. These are separate, named signals in the report — not merged into
the primary product sections.

---

### Tier 3 — Domain-level signals

Search at the domain/team level to catch broader conversations that are topically relevant
even without explicitly naming the product.

**Domain discussion sweep:**
- `[domain term] after:YYYY-MM-DD` (e.g. "marketplace", "platform onboarding", "payout flow")
- `in:#[domain-channel] after:YYYY-MM-DD` — if a relevant domain channel exists, find it first
  using `slack_search_channels` with the domain name

**Filter rule:** From domain-level results, only retain messages that contextually relate to the
primary product or to a known pain point pattern. Discard general company-wide noise.

---

## Analysis: Clustering and Synthesis

After collecting all results, perform the following analysis before writing the output.

### Questions cluster
- Group all question-pattern messages by underlying intent (not surface wording)
- Rank by recurrence: how many distinct people / channels asked the same underlying question?
- Pick top 3-5 questions
- For each, note: is this a docs gap, UX confusion, pricing question, or technical edge case?
- Tag source: Primary product / Cross-product / Domain-level

### Feedback cluster
- Separate pain points (bug reports, friction) from suggestions (feature requests, wishlist items)
- For each, note: volume signal (isolated vs. repeated), channel type (internal team channel, customer-facing, DM), and whether it relates to a recent release
- Tag source: Primary product / Cross-product / Domain-level
- Pick top 3-5 items per sub-category

### Sentiment / Consensus
- Classify overall tone: Positive / Mixed-Positive / Neutral / Mixed-Negative / Negative
- Identify the dominant narrative: what is the prevailing feeling about the product this week?
- Note if any polarisation exists: e.g., "core users are satisfied but integration experience is a recurring friction point"
- Separate primary product sentiment from cross-product friction where they differ

---

## Output: Slack Canvas

Create a Canvas using `slack_create_canvas` with the structure below. Keep the entire Canvas under 400 words. Every line must be actionable — if it doesn't tell the PM what to do or decide, cut it.

After creating the Canvas, send a single Slack message to the target channel with the Canvas link and a one-line summary (max 20 words).

---

### Canvas Structure

**Title**: `[Product Name] — Pulse · [Month YYYY]`

```
# [Product Name] — Pulse · [Month YYYY]

[Sentiment lozenge] [One sentence: the single most important thing to know this month.]

---

## Top Questions

Max 3. One line each: what people are asking → what it signals (docs gap / UX / missing feature).

- **[Question]** — [signal type, 5 words max]
- **[Question]** — [signal type, 5 words max]
- **[Question]** — [signal type, 5 words max]

## Pain Points

Max 4 bullets. Lead with the action the PM should take, not the description of the problem.

- 🔴 **[Fix / Clarify / Escalate]: [thing]** — [one line: who is affected, how often]
- 🟡 **[Fix / Clarify / Escalate]: [thing]** — [one line]
- ...

## Suggestions & Requests

Max 3 bullets. Only include if signal is recurring (2+ people).

- **[Build / Consider / Deprioritise]: [thing]** — [one line: who asked, why]
- ...

## Cross-product friction

Only include if signals were found. One bullet per interaction.

- **[Product A] ↔ [Product B]**: [one line — what breaks and who feels it]

---

*[date range] · public channels + DMs*
```

---

## Output rules

- Total Canvas length: under 400 words. No exceptions.
- Every bullet leads with a verb (Fix, Clarify, Escalate, Build, Consider, Deprioritise).
- No background context, no "this is because", no multi-sentence explanations.
- Omit any section with nothing to report — don't write "Nothing this week."
- Sentiment lozenge: `<span data-type="status" data-color="green">Positive</span>` / `yellow` for mixed / `red` for negative.
- DM content: themes only, never paraphrase a specific private message.

---

## Step 0a — Check Slack is connected

Before doing anything else, attempt a lightweight Slack search to verify the MCP is connected:
- Call `slack_search_public` with `query: "test" limit: 1`
- If it succeeds → proceed to Step 0b.
- If it fails or the tool is not found → stop and guide the user:

> "It looks like the Slack plugin isn't connected yet. Here's how to set it up:
>
> 1. In Claude Code, run: `/plugin install slack`
> 2. Select **user scope** when prompted
> 3. Complete the OAuth flow in your browser
> 4. Come back and re-run this skill
>
> Your own Slack permissions apply at all times — you can only access what you normally can.
> Problems? Post in `#helpme-ai`."

Do not proceed further until the user confirms they've completed setup and re-runs the skill.

---

## Step 0b — First-run setup

Check if `~/.claude/skills/pm-weekly-product-pulse/PREFERENCES.md` exists.

**If it does not exist** → this is a first run. Guide the user through setup before searching:

> "Welcome! Let's get this set up — it'll take 30 seconds.
>
> **1. Which product do you want to track?**
> Be specific — e.g. 'Resell Pricing', 'Connect Billing', 'Payment Links'.
> A broad scope like 'Monetisation' will produce noise."

Wait for the user's answer. Then ask:

> "**2. Which Slack channel should reports be posted to?**
> E.g. `#private-solution-engineering`. Leave blank to receive them as a DM."

Wait for the user's answer. Then ask:

> "**3. Any related products to track for cross-product signals?**
> E.g. if you own Resell Pricing, you might want to track Connect Billing too.
> Leave blank to skip."

Once you have the answers, ask one more question:

> "**4. Would you like a weekly macOS reminder to run this report?**
> I'll set up a notification that fires on a day and time of your choice — opening Claude Code and running the skill takes about 30 seconds from there.
> Reply with a day and time (e.g. 'Monday 9am') or 'skip'."

**If the user provides a day/time:**

Install a cron job that fires a macOS notification at that time. Run this shell command (replacing DAY and HOUR with the user's choice, using cron weekday numbers: Mon=1, Tue=2, Wed=3, Thu=4, Fri=5):

```bash
(crontab -l 2>/dev/null; echo "0 HOUR * * DAY osascript -e 'display notification \"Time to run your PM Weekly Product Pulse. Open Claude Code and run /pm-weekly-product-pulse\" with title \"PM Weekly Pulse 📊\" sound name \"Default\"'") | crontab -
```

Confirm to the user:
> "Done — you'll get a macOS notification every [day] at [time]. When it fires, open Claude Code and run `/pm-weekly-product-pulse`. That's it."

Save the schedule to `PREFERENCES.md`:
```markdown
- **Reminder**: [day] at [time]
```

**If the user says 'skip':** proceed without installing a reminder.

---

Save all collected answers as defaults by creating `PREFERENCES.md`:

```markdown
## Defaults

- **Product**: [user's answer]
- **Channel**: [user's answer, or "DM" if blank]
- **Related products**: [user's answer, or "None"]
- **Reminder**: [day + time, or "None"]
```

Then confirm:
> "All set! Your defaults are saved. When your reminder fires, open Claude Code and run `/pm-weekly-product-pulse` — it'll pick up your settings automatically."

Then proceed with the report using the values just collected.

**If `PREFERENCES.md` already exists** → read it, load any saved defaults for product, channel, and related products. Use these if the user didn't pass arguments. Proceed to Step 0c.

---

## Step 0c — Resolve inputs

Determine the final values for product, channel, and related products using this priority order:
1. Arguments passed by the user when triggering the skill
2. Defaults saved in `PREFERENCES.md`
3. Ask the user if still missing

Confirm with the user before searching:

> "Running the pulse for **[product]**, posting to **[channel]**, covering the last 30 days. Starting now."

Then proceed.

---

## Step 0d — Load preferences (run before every search)

Before doing anything else, check if `PREFERENCES.md` exists in the skill directory
(`~/.claude/skills/pm-weekly-product-pulse/PREFERENCES.md`). If it does, read it.

Apply preferences during synthesis:
- **Exclude**: drop any signal that matches an excluded pattern — do not include it in the Canvas at all.
- **Reclassify**: move matching signals to the correct section (e.g. from Pain Point to Suggestion).
- **Deprioritise**: include matching signals only if no higher-priority signal fills the slot.
- **Note**: if a preference caused a signal to be dropped or reclassified, silently apply it — do not mention it in the Canvas.

---

## Step 4 — Collect feedback after posting

After posting the Canvas and the Slack message, ask the user:

> "Any feedback on this report? Answer in plain language and the skill will be updated accordingly for the next run. For example:
> - 'The capture rounding bug is not relevant for me' → I'll exclude that pattern next time
> - 'Engineering-level API discussions should be deprioritised' → I'll weight those lower
> - 'The VAT issue was wrongly classified as a Pain Point, it's a Suggestion' → I'll reclassify it"

If the user provides feedback:
1. Translate each piece of feedback into a structured preference entry.
2. Append it to `~/.claude/skills/pm-weekly-product-pulse/PREFERENCES.md` using this format:

```markdown
## [Product name or "Global"] — [date added]

- **Exclude**: [plain-language description of what to drop, e.g. "engineering API status code discussions"]
- **Reclassify**: [what signal] → [correct section]
- **Deprioritise**: [plain-language description of what to rank lower]
```

3. Confirm to the user: "Saved. I'll apply that from the next run."

If the user says nothing or says "looks good", do not create or modify the file.
