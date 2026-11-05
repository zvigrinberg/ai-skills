---
name: biweekly-status-deck
description: "Query Jira for APPENG Agentic_MarketPlace tickets updated in the past 2 weeks, summarize them into a categorized table, and generate a Red Hat-branded HTML slide deck for a bi-weekly status meeting."
---

# Bi-Weekly Status Deck Generator

Generate a bi-weekly status presentation for the Agentic MarketPlace stream from Jira data.

## Trigger

Use when the user asks to generate, create, or refresh a bi-weekly status deck, sprint status slides, or status presentation for the Agentic MarketPlace / ATI team.

## Procedure

Follow these steps **in order**. Do not skip steps.

### Step 1: Get the Jira cloudId

Use the MCP tool `mcp__plugin_atlassian_atlassian__getAccessibleAtlassianResources` to get the cloudId for `redhat.atlassian.net`. Cache it for the session.

If the tool is not loaded, use `ToolSearch` with query `select:mcp__plugin_atlassian_atlassian__getAccessibleAtlassianResources` first.

### Step 2: Query Jira for tickets

Use the MCP tool `mcp__plugin_atlassian_atlassian__searchJiraIssuesUsingJql` with:

```
cloudId: <from step 1>
jql: project = APPENG AND updated >= -2w AND labels = "Agentic_MarketPlace" ORDER BY updated DESC
maxResults: 50
view: evidence
```

If the tool is not loaded, use `ToolSearch` with query `select:mcp__plugin_atlassian_atlassian__searchJiraIssuesUsingJql` first.

If the result is too large and gets saved to a file, parse it with a Python script via Bash:

```bash
python3 -I -c "
import json, sys

with open('<PATH_TO_RESULT_FILE>') as f:
    data = json.loads(f.read())

if 'data' in data:
    data = data['data']

issues = data.get('issues', [])
print(f'Total issues: {len(issues)}')
print(f'isLast: {data.get(\"isLast\")}')
print()

for issue in issues:
    key = issue.get('key', '?')
    fields = issue.get('fields', {})
    summary = fields.get('summary', '?')
    status = fields.get('status', {}).get('name', '?') if isinstance(fields.get('status'), dict) else '?'
    priority = fields.get('priority', {}).get('name', '?') if isinstance(fields.get('priority'), dict) else '?'
    issue_type = fields.get('issuetype', {}).get('name', '?') if isinstance(fields.get('issuetype'), dict) else '?'
    assignee = fields.get('assignee', {})
    assignee_name = assignee.get('displayName', 'Unassigned') if isinstance(assignee, dict) and assignee else 'Unassigned'
    updated = fields.get('updated', '?')[:10]
    desc = fields.get('description', '') or ''
    # Truncate description for display
    if len(desc) > 200:
        desc = desc[:200] + '...'

    print(f'{key} | {issue_type} | {status} | {assignee_name} | {updated} | {summary}')
    if desc:
        print(f'  DESC: {desc}')
    print()
"
```

If `isLast` is `false`, paginate using the `nextPageToken` from the response until all results are retrieved.

### Step 3: Categorize tickets

**Categories must be determined dynamically from the ticket data — do NOT use a hardcoded list.** Read every ticket's summary, description, parent/epic context, and labels, then cluster them into coherent thematic groups. Each group should have a short descriptive name (e.g., "ATI Agent Core", "Observability / OTEL", "Proactive Classifier Agent", "Research & Exploration").

Rules:
- **No "Other" / catch-all category.** Every ticket must belong to a meaningful, named group. If a ticket doesn't fit an existing cluster, create a new one for it.
- Group by the *work theme*, not by issue type or status.
- If a parent/epic relationship exists, tickets under the same parent should generally land in the same group.
- A category can have as few as 1 ticket — that's fine; a real label is better than a dump bucket.
- Categories will vary from sprint to sprint as the team's work shifts. Do not force-fit tickets into categories from previous sprints.

For each ticket, extract:
- **Key** (e.g., APPENG-6639)
- **Type** (Feature, Epic, Story, Task, Sub-task)
- **Status** (New, In Progress, Review, Pull Request, Closed)
- **Owner** (assignee display name)
- **Summary** (ticket summary field)
- **Description** (succinct 1-2 sentence summary derived from the ticket description field; if description is empty, derive from summary and parent context)

### Step 4: Produce the summary table

Present the categorized tickets to the user as a markdown table grouped by category, like this:

```
### Category Name

| Key | Type | Status | Owner | Summary | Description |
|-----|------|--------|-------|---------|-------------|
| APPENG-XXXX | Task | In Progress | Name | Short summary | Succinct description |
```

Include status distribution counts:
- Total tickets
- Closed / In Progress / New / Review / Pull Request

### Step 5: Generate the HTML slide deck

Create an HTML file at `docs/biweekly-status-YYYY-MM-DD.html` (using today's date).

The slide deck structure is:

1. **Title Slide** (always first) — "Agentic MarketPlace — Bi-Weekly Status Update", sprint date range (past 2 weeks), team members (derive from assignees in the ticket data), project APPENG Board 14076
2. **Sprint Snapshot** (always second) — Stats grid (total, closed, in progress, review/PR, new) + 3-4 key highlights as bullet points summarizing the most important things that happened
3. **One slide per category** (dynamic, one per category from Step 3) — Each category gets its own slide with:
   - Slide header = category name
   - Table of tickets in that category (Key, Status pill, Summary + succinct description)
   - If the category has a natural sub-structure (e.g., completed vs. in-progress items, or two distinct work streams), use a two-column layout
   - A highlight box at the bottom with "Next steps" or key takeaways for that category
   - If a category has too many tickets (>8) to fit one slide, split across two slides
4. **Next Sprint Priorities** (always last) — Two-column layout with focus areas derived from in-progress and new tickets, grouped by category. Include a Risks/Blockers/Dependencies box at the bottom.

Total slide count = 3 fixed (title, snapshot, next sprint) + N category slides. Adjust slide numbering accordingly.

**Slide design requirements:**

- 960x540px per slide, white background, dark header bar with red accent bar below
- Use Red Hat brand fonts (`Red Hat Display` for headings, `Red Hat Text` for body) via Google Fonts import
- Color palette: `--rh-red: #EE0000`, `--rh-dark: #151515`, `--rh-blue: #0066CC`, `--rh-green: #3E8635`, `--rh-orange: #F0AB00`, `--rh-purple: #5E40BE`
- Status pills with color coding: PR (blue), Review (amber), In Progress (indigo), New (gray), Closed (green)
- Each slide has a slide number bottom-left and date top-right in the header
- `page-break-after: always` on each slide for print support
- Self-contained single HTML file (no external JS dependencies)

### Step 6: Present results

Tell the user:
1. The summary table (in the conversation)
2. The path to the generated HTML file
3. Remind them to open it in a browser to view/present
4. Ask if they want any adjustments (status overrides, wording changes, additional notes)

## Notes

- The label filter is `Agentic_MarketPlace` (exact spelling with underscore)
- If `acli` is authenticated, prefer it over MCP tools. Test with `acli jira auth status`. If not authenticated, use MCP tools as fallback.
- Categories are derived fresh each sprint — never reuse a previous sprint's category list without re-evaluating the current tickets.
- The "Next Sprint Priorities" slide should be derived from in-progress and new tickets, not invented.
- Always ask the user for status overrides or corrections after presenting the initial summary, before generating the final deck. The user may want to change statuses or add notes that aren't reflected in Jira yet.
