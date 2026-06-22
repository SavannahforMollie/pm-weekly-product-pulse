# PM Weekly Product Pulse — Setup Guide

A Claude Code skill that scans Slack and delivers a weekly product intelligence report as a Canvas: top questions, pain points, feature requests, and overall sentiment — anchored to a product you own.

---

## Prerequisites

- **Claude Code** installed ([download here](https://claude.ai/download))
- A Mollie Slack account
- ~5 minutes

---

## Step 1 — Install the skill

Unzip the file and place it in your Claude skills directory:

```bash
unzip pm-weekly-product-pulse.zip -d ~/.claude/skills/
```

Restart Claude Code (or start a new session). The skill is picked up automatically — no further configuration needed.

---

## Step 2 — Connect Slack

The skill reads your Slack messages, so Claude Code needs access to Slack via MCP.

In Claude Code, run:

```
/plugin install slack
```

Follow the prompts:
- Select **user scope** when asked
- Complete the OAuth flow in your browser
- Come back to Claude Code once authenticated

**How to verify it worked**: you should see `plugin:slack:slack` listed when you run `claude mcp list`.

> Your own Slack permissions apply at all times. You cannot access private channels or DMs you don't already have access to.

> Problems? Post in `#helpme-ai`.

---

## Step 3 — First run

Trigger the skill by typing something like:

```
/pm-weekly-product-pulse Resell Pricing post in #private-solution-engineering
```

Or just:

```
/pm-weekly-product-pulse
```

...and Claude will ask you two questions:

1. **Which product?** — be specific. "Resell Pricing" or "Connect Billing" works well. "Monetisation" is too broad and will produce noise.
2. **Where to post?** — the Slack channel where the Canvas should be delivered. Defaults to a DM to yourself if you skip this.

Claude will then search Slack for the past 30 days, synthesise the findings, and post a Canvas to your chosen channel.

---

## Step 4 — Give feedback

After every report, Claude will ask:

> *"Any feedback on this report?"*

Answer in plain language:

| What you say | What happens |
|---|---|
| *"The API status code thread isn't relevant to me"* | That pattern is excluded from future reports |
| *"Don't include engineering-only discussions"* | Those are deprioritised going forward |
| *"That was a suggestion, not a pain point"* | Reclassified in future runs |

Your feedback is saved to a `PREFERENCES.md` file inside the skill directory and applied silently on every future run. The skill gets more accurate over time without you repeating yourself.

To review or edit your preferences manually:

```bash
cat ~/.claude/skills/pm-weekly-product-pulse/PREFERENCES.md
```

---

## Step 5 — Set up a weekly reminder (optional)

During first-run setup, the skill will ask if you want a weekly macOS notification. Say yes and give it a day and time — e.g. *"Monday 9am"*.

The skill installs a cron job that fires a notification at that time every week:

> **PM Weekly Pulse 📊** — *"Time to run your PM Weekly Product Pulse. Open Claude Code and run /pm-weekly-product-pulse"*

When the notification fires, open Claude Code and run `/pm-weekly-product-pulse`. Your saved defaults (product, channel) are applied automatically — no arguments needed.

**To remove the reminder later**, run:

```bash
crontab -l | grep -v "pm-weekly-product-pulse\|PM Weekly Pulse" | crontab -
```

---

## Troubleshooting

| Problem | Fix |
|---|---|
| Skill not found after unzipping | Check that the folder is at `~/.claude/skills/pm-weekly-product-pulse/` and restart Claude Code |
| Slack search returns no results | Confirm the Slack plugin is connected (`claude mcp list`) and try re-authenticating with `/plugin install slack` |
| Canvas posts to the wrong channel | Re-run and specify the channel explicitly: `/pm-weekly-product-pulse [product] post in #channel-name` |
| Too much noise in the report | Give feedback after the run — Claude will learn your preferences |
| Slack MCP issues | Post in `#helpme-ai` |
