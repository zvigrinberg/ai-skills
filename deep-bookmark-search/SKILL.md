---
name: deep-bookmark-search
description: "Search for a concept or idea across bookmarked sources using the appropriate tool per source type: WebFetch for websites, git clone+grep for repos, Atlassian Rovo MCP for Jira/Confluence, Google Workspace MCP for Google Docs. Presents candidates for user selection, searches in parallel, and saves results."
---

# Deep Bookmark Search

Search for a concept, idea, or technical topic across bookmarked sources. Each source is searched using the method appropriate to its type.

## Trigger

Use when the user asks to:
- Deep search across their bookmarks for a concept or idea
- Find information about a topic across multiple bookmarked sources
- Research a concept using their saved knowledge base
- "Search my bookmarks deeply for X"

## Inputs

The workflow requires **two separate queries** that serve different purposes:

1. **Target query**: keywords used to filter bookmarks and find *which sources to search in*. This is typically broader (e.g., "agent skills", "agentic marketplace", "LangChain").
2. **Search query**: the specific concept, idea, or question to search *within* those sources. This is typically more focused (e.g., "how to register a skill", "skill discovery protocol", "tool vs skill distinction").

These two queries are intentionally different. The target query casts a wide net to find relevant sources; the search query digs deep within those sources for specific information.

- **Scope**: optional — user may specify a folder, classification, or subset of bookmarks

**Procedure for collecting inputs:** Always ask the user for both queries explicitly. Suggest defaults if the user provides only one:
- If only a broad topic is given, use it as the target query and ask the user for a more specific search query.
- If only a specific question is given, infer a broader target query and confirm with the user.

## Procedure

Follow these steps in order.

### Step 1: Collect Inputs

Ask the user for two queries:
- **Target query**: "What keywords should I use to find relevant bookmarked sources?" (broad, for filtering bookmarks)
- **Search query**: "What specific concept or question should I search for within those sources?" (focused, for deep search)

If the user provides a single input, determine which type it is and ask for the other.

### Step 2: Bookmark Discovery

Search `~/git/my-knowledge/bookmarks.html` for bookmarks matching the **target query** keywords:

```bash
grep -i '<DT><A HREF' ~/git/my-knowledge/bookmarks.html \
  | grep -i -E 'target_keyword1|target_keyword2|target_keyword3' \
  | sed 's/.*HREF="\([^"]*\)"[^>]*>\(.*\)<\/A>/\2\n  \1\n/'
```

### Step 3: Classify Candidates

For each matched bookmark URL, classify by source type using URL pattern matching:

| URL Pattern | Source Type |
|---|---|
| `github.com/<org>/<repo>*` | **Git Repo** |
| `redhat.atlassian.net/browse/*` or `issues.redhat.com/*` | **Jira Issue** |
| `redhat.atlassian.net/wiki/*` | **Confluence Page** |
| `docs.google.com/*` or `drive.google.com/*` or `slides.google.com/*` | **Google Resource** |
| `source.redhat.com/*` | **RH Internal Web** |
| `*.redhat.com/*` (other) | **RH Internal Web** |
| Everything else | **Public Web** |

Also apply the bookmark classification:
- **Technology Stack** folder → Broad Knowledge
- **Cloud** folder → Mixture of Broad and Internal
- **All other folders** → Internal Knowledge and Data

**Git repos span both classifications.** A single repo can contain both broad and internal knowledge — even a public upstream repo may hold content directly related to Red Hat core business. Determine each repo's classification by combining these signals:
1. **Bookmark folder**: Technology Stack = likely Broad; Cloud = Mixed; others = likely Internal
2. **GitHub org**: Red Hat-specific orgs (e.g., `RHEcosystemAppEng`, `opendatahub-io`, `nv-morpheus/*-RH`) are strongly Internal. But public orgs (e.g., `kubernetes-sigs`, `langchain-ai`) can still contain Internal knowledge if the repo is directly tied to Red Hat projects or business context.
3. **Repo visibility**: Private repos are always Internal, regardless of org.
4. **Content context**: If the repo name, description, or bookmark title references Red Hat products, projects, or internal initiatives, classify as Internal even if the org is public.

When uncertain, classify as **Mixed** and include in both report groups. In the report, split Git repo findings into: "Git Repositories (Broad Knowledge)" and "Git Repositories (Internal)"

### Step 4: Present Candidates & Refine Scope

Present the classified candidates to the user in a table:

```
| # | Name | Source Type | Classification | URL |
```

Ask the user:
1. Which targets to include (all, specific numbers, top N)
2. Whether to override Git search depth (default: docs only; option: full codebase)
3. Whether to also run a bonus web search (`WebSearch`) beyond bookmarks

### Step 5: Execute Deep Searches

For each selected target, search using the **search query** (not the target query). Run independent searches in parallel where possible.

#### 5a: Git Repositories

```bash
mkdir -p /tmp/bookmark-search
git clone --depth 1 <repo_url> /tmp/bookmark-search/<repo_name>
```

**Default (docs only):**
```bash
find /tmp/bookmark-search/<repo_name> -type f \( -name "README*" -o -name "*.md" -o -path "*/docs/*" \) \
  | xargs grep -i -l '<search_query_keyword>'
```
Then read matching files for relevant content.

**Full codebase (if user overrides):**
```bash
grep -i -r '<search_query_keyword>' /tmp/bookmark-search/<repo_name> --include='*' -l
```
Then read matching files for relevant content.

#### 5b: Jira Issues

Use the Atlassian Rovo MCP search tool:
```
mcp__plugin_atlassian_atlassian__search
  cloudId: <from getAccessibleAtlassianResources>
  query: "<search_query>"
  mode: "AGENTIC"
```

Or use the `acli` skill for JQL searches:
```bash
acli jira workitem search --jql "text ~ '<search_query>' ORDER BY updated DESC" --json
```

Alternatively, if a specific Jira issue key is in the bookmark URL (e.g., `APPENG-123`), fetch it directly using the `getJiraIssue` skill.

#### 5c: Confluence Pages

Use Rovo MCP search with `targetApp: "CONFLUENCE"`, or CQL:
```
mcp__plugin_atlassian_atlassian__searchConfluence
  cloudId: <cached>
  cql: "text ~ '<search_query>' ORDER BY lastmodified DESC"
```

Or if a specific page ID is in the URL, fetch it with the `getConfluencePage` skill.

#### 5d: Google Resources (Docs, Slides, Sheets, Drive)

**When Google Workspace MCP is installed:** use it to read and search the document content.

**Fallback (MCP not available):** Prompt the user:
> "This is a Google Doc/Slides resource that requires authentication. Please open it manually or authenticate with:
> `! gcloud auth login`
> Then I can retry."

List the Google URLs that couldn't be searched so the user can check them manually.

#### 5e: Public Web Sites

Use `WebFetch`:
```
WebFetch(url: "<bookmark_url>", prompt: "Search for information about: <search_query>. Extract relevant sections, definitions, examples, or references.")
```

**Fallback (if WebFetch fails):** Try `WebSearch` with the site domain:
```
WebSearch(query: "<search_query> site:<domain>")
```

#### 5f: RH Internal Web (source.redhat.com, etc.)

Try `WebFetch` first. If it fails with 400/401/403:
> Prompt the user: "This internal Red Hat page requires authentication. You can try:
> `! curl -sS '<url>' | head -100`
> or open it manually in your browser."

#### 5g: Bonus Web Search (if user opted in)

```
WebSearch(query: "<search_query>")
```

Present web results separately, labeled as "Additional sources (not in bookmarks)".

### Step 6: Aggregate & Report

Compile all findings into a structured report:

```markdown
# Deep Search Results: <search_query>
**Date:** <today>
**Target query (bookmark filter):** <target_query>
**Search query (deep search):** <search_query>
**Sources searched:** <count>
**Sources skipped (auth required):** <count>

## Findings

### From Git Repositories (Broad Knowledge)
| Source | Finding | File/Location |
|---|---|---|

### From Git Repositories (Internal)
| Source | Finding | File/Location |
|---|---|---|

### From Jira/Confluence (Internal)
| Source | Finding | Link |
|---|---|---|

### From Web Sites
| Source | Finding | Link |
|---|---|---|

### Skipped (Authentication Required)
| Source | Type | URL |
|---|---|---|

## Bonus: Web Search Results (if applicable)
| Source | Finding | Link |
|---|---|---|
```

**Output both:**
1. Print the report inline in the conversation
2. Save to `~/search-results/<topic-slug>.md` (kebab-case the concept for the filename)

### Step 7: Cleanup

Remove all cloned repos:
```bash
rm -rf /tmp/bookmark-search/
```

## Error Handling

- **Auth failures**: Never silently skip. Always inform the user and prompt for authentication.
- **Clone failures**: Report the repo URL and error, continue with other sources.
- **WebFetch failures**: Try `WebSearch` with `site:<domain>` as fallback, then report if still failing.
- **Rate limits (Rovo credits)**: Warn the user about Rovo credit consumption before executing Jira/Confluence searches. Each Rovo search can consume up to 10 credits.

## Notes

- Git repos are cloned with `--depth 1` to minimize bandwidth and disk usage.
- The `/tmp/bookmark-search/` directory is always cleaned up at the end, even if the search is interrupted.
- Google Workspace MCP support is pending installation — the skill will automatically use it once available.
- The bookmark classification (Broad/Mixed/Internal) is applied per the rules in the `bookmarks-manager` skill.
