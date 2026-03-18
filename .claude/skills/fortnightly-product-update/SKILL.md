---
name: fortnightly-product-update
description: Generate the fortnightly product update by consolidating Headlines from all product team documents in Notion. Use when the user asks to create a product update, generate the fortnightly update, or consolidate team headlines.
---

# Fortnightly Product Update

Automates the creation of the biweekly product update by duplicating the previous update and filling in the new content from each team's leadership sync.

## When to run

Run the week after the fortnightly product leadership syncs (which happen every two weeks on Wednesday and Thursday). Pull all team syncs dated within the last 14 days.

## Notion Configuration

| Resource | Value |
|----------|-------|
| Team syncs data source | `collection://f31a0c99-1466-4cdc-844a-344a9c4cd84a` |
| Updates parent page | `1d522837308b803a93f1c7c6f369ff13` |
| Previous update (reference) | `https://www.notion.so/31722837308b80be919ac8498ef8f01a` |

## MCP Tools Reference

| Tool | Purpose |
|------|---------|
| `notion-fetch` | Get page content or database schema |
| `notion-query-data-sources` | Query team syncs database |
| `notion-duplicate-page` | Duplicate previous update as starting point |
| `notion-update-page` | Update page title |
| `notion-search` | Find pages by title |

## Team Emoji Map

Use these fixed emojis in the "Highlights per team" section. Match by the `Team` field in the database.

| Team (DB field) | Display Name | Emoji |
|-----------------|--------------|-------|
| Fundamentals | Fundamentals *(AKA LevFin)* | 🕴️ |
| Permissions | Permissions | 🔐 |
| Distressed | Distressed | 💼 |
| Search | AI Workflows | 🤖 |
| PCRED | Private Credit | 💳 |
| SCRED | Structured Credit | 📊 |
| Deals | Deals | 🤝 |
| Onboarding & Activation | Onboarding & Activation | 👋 |
| Automated News | News | 📰 |
| Chat | AI Chat | 💬 |
| Retrieval | Retrieval | 🔍 |

---

## Workflow

### Step 1: Find the most recent syncs per team

Query the team syncs database and retrieve all syncs from the last 14 days. Deduplicate by team — keep only the most recent entry per team.

```sql
SELECT url, Name, Team, "Team Name", "date:Date:start", Status
FROM "collection://f31a0c99-1466-4cdc-844a-344a9c4cd84a"
WHERE "date:Date:start" >= date('now', '-14 days')
ORDER BY "date:Date:start" DESC
```

If a team has multiple entries within the window, use the one with the latest date.

### Step 2: Extract Headlines from each team sync page

For each team sync page, call `notion-fetch` and extract content in this priority order:

1. **`# Headlines` section** — primary source. Use this if it contains concise bullets.
2. **`# Headlines TL;DR` section** — some teams (e.g. SCRED) have a separate TL;DR. Use this alongside Headlines for additional structure.
3. **Full OKR sections** — scan as a fallback only if Headlines is missing or empty.

**Verbosity rule:**
- If the Headlines section is already concise (short bullets, no deep nesting) → copy verbatim.
- If it is verbose (long paragraphs, deep sub-bullets, lots of context) → summarise to 2–5 bullets per team.
- Flag verbose teams in your working notes so Moises can give them feedback.

### Step 3: Duplicate the previous update page

Call `notion-duplicate-page` on `https://www.notion.so/31722837308b80be919ac8498ef8f01a`.

This preserves the "Product vision & strategy Q1 roadmaps" section (images and Google Slides link) without needing to recreate it.

Then rename the page using `notion-update-page`:
- Title format: `Product Update | {D} {Mon} {YYYY}` (e.g. `Product Update | 27 Mar 2026`)

### Step 4: Update the date header

Replace the gray introductory line at the top of the page:
> *This update covers our progress between {start date} to {end date}.*

Use the date range corresponding to the current fortnight.

### Step 5: Rewrite the TLDR section

The TLDR has three parts. Synthesise from all team headlines:

**🚢 Shipped** (max 5–7 items)
- Customer-facing features that went live
- Format: `**Team name:** Description`
- Prioritise completed, released items only

**👩🏻‍💻 Internal release (9fin only)** (max 3–5 items)
- Internal tools, beta/alpha features, feature-flagged releases
- Format: `**Team name:** Description`

**🔜 Coming up** (max 4–6 items)
- Near-term items explicitly flagged as "next" or "coming up" in team headlines
- Format: `**Team name:** Description`

Keep each bullet to one line. Omit bug fixes, maintenance tasks, and planning activities unless they are significant milestones.

### Step 6: Rewrite the "Highlights per team" section

Clear the existing team sections and replace with the new content.

For each team (in the same order as the 27 Feb 2026 reference update):
1. Fundamentals, 2. Permissions, 3. Distressed, 4. AI Workflows, 5. Private Credit, 6. Structured Credit, 7. Deals, 8. Onboarding & Activation, 9. News, 10. AI Chat

Format each team as a **toggle heading** (matching the reference format):

```
### {emoji} {Display Name} {toggle=true}
  - Headline bullet 1
  - Headline bullet 2
  - ...
```

If a team has no sync within the 14-day window, skip them and note it in your output to Moises.

### Step 7: Update the Insights section

For each team sync, look for an `# Insights` section (or equivalent: `# Insights Dashboard`, `# Metrics Dashboard` used by AI Workflows). Extract substantive content grounded in either (a) qualitative customer feedback or (b) product analytics. Do not include OKR delivery status, vendor decisions, or team/capacity updates — these are not insights.

**What counts as an insight:**
- Usage metrics with meaningful context (e.g. like rates, retention rates, MAUs, feature engagement %)
- Customer feedback summaries from user interviews, SME testing, or structured feedback rounds

**What to skip:**
- Empty Insights sections (just a dash or blank)
- Sections that only contain links to dashboards (Metabase, Amplitude URLs) with no narrative
- OKR delivery status (e.g. "2 of 3 KRs at 100%", "migration at 50%") — these belong in Headlines, not Insights
- Vendor/partnership decisions (e.g. "Pivoting to agency over internal build — Vendor X recommended") — strategic decisions, not insights
- Team capacity updates, planning activities, or process changes

**Decision logic:**
- If **1 or more teams** have substantive insights → replace the entire `## Insights` section with new toggles (one per team that contributed), using the format `<details><summary>**{Team}: {Topic}**</summary>...</details>`
- If **no teams** have substantive insights → remove the `## Insights` section entirely (replace with empty string)

**Important:** The `## Insights` section often contains image blocks (charts, heatmaps) and synced block references from the previous update. These cannot be matched and removed programmatically via `update_content` (Notion stores images as internal block IDs, not URLs). Replace only the text-only toggle content first, then flag any orphaned image or synced block elements for manual cleanup.

### Step 8: Leave unchanged

Do **not** modify:
- `### 🎯 Product vision strategy & Q1 roadmaps` — images and deck link carry over from the duplicate

### Step 9: Send Slack review request

Post a message to channel `C07DU9RR7FV` (product trios channel) asking the team to review the draft before it goes out. Use `slack_post_message`.

**Message template:**

```
Good morning. I have put together the fortnightly product update here: {notion_url}

This was created by pulling the Headlines from each team's leadership sync.

Can you please review this and make any edits by EOD today?

Thank you!
```

Replace `{notion_url}` with the full URL of the newly created page.

Post the review message immediately after the page is created. When running via the scheduled cron job, the script fires at 10am so no timing check is needed.

### Step 10: Return output

Return to Moises:
1. The URL of the new Notion page
2. A brief summary of which teams were included
3. A list of any teams that were **missing** (no sync in the last 14 days)
4. A list of any teams whose Headlines were **verbose and summarised** (so Moises can follow up)
5. Which teams contributed to the Insights section (or note that it was removed if none)
6. Any orphaned image/synced block elements in the Insights section that need manual deletion in Notion
7. Confirmation that the Slack review message was sent (or a note if it's scheduled for 10am)

---

## Output Page Structure Reference

```
[gray] This update covers our progress between {dates}.

[toggle] Table of Contents

### ✉️ TLDR  [blue_bg]
**🚢 Shipped:**
- **Team:** Item

**👩🏻‍💻 Internal release (9fin only):**
- **Team:** Item

**🔜 Coming up:**
- **Team:** Item

### 🎯 Product vision strategy & Q1 roadmaps  [blue_bg]
[images + deck link — copied from previous, do not edit]

## Insights  [blue_bg]
[copied from previous, do not edit]

### Highlights per team for the last 2 weeks  [blue_bg]

### 🕴️ Fundamentals *(AKA LevFin)*  [toggle]
  - ...

### 🔐 Permissions  [toggle]
  - ...

### 💼 Distressed  [toggle]
  - ...

### 🤖 AI Workflows  [toggle]
  - ...

### 💳 Private Credit  [toggle]
  - ...

### 📊 Structured Credit  [toggle]
  - ...

### 🤝 Deals  [toggle]
  - ...

### 👋 Onboarding & Activation  [toggle]
  - ...

### 📰 News  [toggle]
  - ...

### 💬 AI Chat  [toggle]
  - ...
```
