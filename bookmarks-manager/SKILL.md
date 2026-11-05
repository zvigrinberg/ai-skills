---
name: bookmarks-manager
description: "Search, add, classify, and manage Chrome bookmarks from the user's exported bookmarks.html file. Supports keyword search, bookmark insertion with auto-inferred descriptions, classification tagging, and stale bookmark detection."
---

# Bookmarks Manager

Manage the user's Chrome-exported bookmarks file as a personal knowledge base.

## Trigger

Use when the user asks to:
- Search bookmarks by keyword, topic, or technology
- Add a new bookmark (URL) to the collection
- Classify or tag bookmarks
- Find stale or deprecated bookmarks
- List bookmarks in a specific folder or category
- Count or summarize bookmark statistics

## Configuration

- **Bookmarks file**: `~/git/my-knowledge/bookmarks.html`
- **Format**: Netscape Bookmark File (standard Chrome export)

### Classification Rules

Bookmarks are classified by their folder location:

| Folder | Classification |
|---|---|
| **Technology Stack** (and subfolders) | **Broad Knowledge** — general tech, open-source, languages, frameworks, publicly available |
| **Cloud** (and subfolders) | **Mixture of Broad and Internal** — public cloud/k8s/container docs mixed with Red Hat-internal resources |
| **All other folders** | **Internal Knowledge and Data** — Red Hat systems, company resources, projects, HR/admin, internal docs |

## Tools

### Tool 1: Search Bookmarks

Search bookmarks by one or more keywords. Returns matching bookmark names, URLs, and folder context.

**Procedure:**

1. Run a case-insensitive grep on the bookmarks file, matching `<DT><A HREF` lines against the user's keywords.
2. Use multiple grep patterns joined with `|` (OR) for broader searches, or piped greps for AND logic.
3. Extract and format results as: **Name** + **URL**, using this sed extraction:
   ```bash
   grep -i '<DT><A HREF' ~/git/my-knowledge/bookmarks.html \
     | grep -i -E 'keyword1|keyword2' \
     | sed 's/.*HREF="\([^"]*\)"[^>]*>\(.*\)<\/A>/\2\n  \1\n/'
   ```
4. Present results in a markdown table grouped by relevance or sub-topic.
5. If the user's classification scheme applies, tag results with their classification.

### Tool 2: Add Bookmark

Add a new bookmark entry to the bookmarks file.

**Procedure:**

1. **Infer description**: If the user doesn't provide a title/description:
   - Try to fetch the URL with `WebFetch` to extract the page title and purpose.
   - If the URL is behind authentication (returns 400/401/403), infer from the URL path segments (domain, path components, common acronyms).
   - Format the title as: `<Name> - <Short Description> - <Site Name>`
2. **Determine placement**: Place the bookmark in the folder that best matches:
   - If the user specifies a classification or folder, use that.
   - Otherwise, infer from the URL domain and path (e.g., `source.redhat.com` → Company Systems).
3. **Generate timestamp**: Get the current Unix timestamp with `date +%s` for the `ADD_DATE` attribute.
4. **Insert the entry**: Use the `Edit` tool to insert the new `<DT><A>` line in the correct folder location:
   ```html
   <DT><A HREF="<url>" ADD_DATE="<timestamp>"><title></A>
   ```
5. Confirm the addition to the user with the bookmark name, URL, classification, and folder placement.

### Tool 3: Classify Bookmarks

Tag or group bookmarks by their classification.

**Procedure:**

1. Identify which folder(s) the target bookmarks reside in using grep with surrounding context.
2. Apply the classification rules from the Configuration section above.
3. Present results grouped by classification: **Broad Knowledge**, **Mixture of Broad and Internal**, or **Internal Knowledge and Data**.

### Tool 4: Detect Stale Bookmarks

Find bookmarks that may be outdated or point to deprecated services.

**Procedure:**

1. Search for known deprecated domains or services (e.g., `labs.opentlc.com` is deprecated).
2. Optionally, attempt to fetch URLs with `WebFetch` or `curl -sI` to check for HTTP errors (404, 410, 5xx).
3. Report stale bookmarks with their name, URL, and reason for flagging.

**Known stale entries:**
- `labs.opentlc.com` (OpenTLC) — deprecated, replaced by demo.redhat.com (RHDP)

### Tool 5: Bookmark Statistics

Provide summary statistics about the bookmarks collection.

**Procedure:**

```bash
# Total bookmarks
grep -c '<DT><A HREF' ~/git/my-knowledge/bookmarks.html

# Total folders
grep -c '<DT><H3' ~/git/my-knowledge/bookmarks.html

# List all folder names
grep '<DT><H3' ~/git/my-knowledge/bookmarks.html | sed 's/.*<H3[^>]*>//; s/<\/H3>//'
```

## Output Format

- Always return results in **markdown tables** with columns: Bookmark Name | Link
- Group results by sub-topic or classification when there are many matches
- For searches with no results, suggest alternative keywords or broader search terms
- When adding bookmarks, confirm with: name, URL, classification, and folder placement
