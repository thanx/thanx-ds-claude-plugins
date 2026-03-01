---
description: Research a Thanx platform question using Keystone's code exploration and data tools, then publish the answer to the Knowledge Gaps Notion database.
---

# Ask Keystone

Answer a question about the Thanx platform using Keystone's code exploration, data, and infrastructure tools.

## Usage

```bash
/ds:ask-keystone <question>
```

**Required:**

- `question` - A question about how the Thanx platform works, API behavior, data models, infrastructure, or implementation details

---

You are executing the `/ds:ask-keystone` command.

## Step 1: Identify What to Search

Arguments: $ARGUMENTS

Determine the best approach to answer the question:

- **Code questions** (how something works, where something is defined, implementation details): Use `list_repositories`, `search_code`, `read_file`, `find_files`, `list_directory`
- **Data/schema questions** (what data exists, table structures, relationships): Use `list_replica_databases`, `replica_query`
- **API behavior questions** (how endpoints work, request/response formats): Combine code search with Thanx docs MCP
- **Infrastructure/deployment questions** (services, clusters, health): Use `ecs_clusters`, `ecs_services`, `ecs_deployments`, `health_check`
- **Error/incident questions** (what's broken, recent errors): Use `sentry_issues`, `sentry_issue_details`, `datadog_logs`, `datadog_metrics`
- **Feature flag questions** (what's enabled, targeting rules): Use `launchdarkly_feature_flags`, `launchdarkly_feature_flag`
- **Git history questions** (who changed what, when, why): Use `git_file_history`, `git_blame`, `git_diff`, `git_show_commit`

## Step 2: Explore with Keystone

Execute the appropriate Keystone MCP tool calls to gather evidence. Be thorough:

1. **Start broad**: Use `list_repositories` to see available repos, or `search_code` with a broad pattern
2. **Narrow down**: Once you find relevant files, use `read_file` to examine the implementation
3. **Follow references**: If code references other files, models, or services, trace those too
4. **Check the database**: If relevant, use `replica_query` against the appropriate database to understand schema or data patterns (SELECT only, read-only)
5. **Cross-reference docs**: Use the **thanx-docs** MCP to supplement with official API documentation when relevant

## Step 3: Synthesize the Answer

Present your findings clearly:

1. **Direct answer** - Answer the question concisely up front
2. **Evidence** - Show the relevant code snippets, query results, or file paths that support your answer
3. **Context** - Explain how the pieces fit together (e.g., which service owns this, how data flows, what calls what)
4. **Related information** - Mention anything else the user should know (gotchas, related features, recent changes)

## Step 4: Publish to Notion

After presenting your answer to the user, publish it to the **Knowledge Gaps** Notion database.

1. **Read the `.env` file** at the project root to get `NOTION_API_TOKEN` and `NOTION_DATABASE_ID`. If no `.env` file exists, ask the user for the Notion credentials or skip this step.
2. **Choose a question title** - a concise version of the question (e.g., "How does basket refund work?")
3. **Choose a question type** - best match from: `Product Q&A` (default for most questions)
4. **Choose tags** - pick relevant tags from: `loyalty`, `merchant-dashboard`, `items`, `basket`, `purchases`, `api`, `pos`. You may also create new tags if none fit.
5. **Build the page body** - convert your answer into Notion blocks. Use paragraph blocks for text, heading blocks for sections, and code blocks for code snippets. Keep it structured and readable.
6. **Create the page** using `curl`:

```bash
curl -s -X POST "https://api.notion.com/v1/pages" \
  -H "Authorization: Bearer $NOTION_API_TOKEN" \
  -H "Notion-Version: 2022-06-28" \
  -H "Content-Type: application/json" \
  -d '{
    "parent": {"database_id": "$NOTION_DATABASE_ID"},
    "properties": {
      "Name": {"title": [{"text": {"content": "<question title>"}}]},
      "Question Type": {"select": {"name": "<question type>"}},
      "Tags": {"multi_select": [{"name": "<tag1>"}, {"name": "<tag2>"}]}
    },
    "children": [<Notion block objects>]
  }'
```

7. **Report the result** - tell the user whether the page was created successfully and share the Notion URL from the response.

**Notion block format reference** for the `children` array:

- Paragraph: `{"object": "block", "type": "paragraph", "paragraph": {"rich_text": [{"type": "text", "text": {"content": "text here"}}]}}`
- Heading 2: `{"object": "block", "type": "heading_2", "heading_2": {"rich_text": [{"type": "text", "text": {"content": "heading"}}]}}`
- Heading 3: `{"object": "block", "type": "heading_3", "heading_3": {"rich_text": [{"type": "text", "text": {"content": "heading"}}]}}`
- Code block: `{"object": "block", "type": "code", "code": {"rich_text": [{"type": "text", "text": {"content": "code here"}}], "language": "ruby"}}`
- Bulleted list: `{"object": "block", "type": "bulleted_list_item", "bulleted_list_item": {"rich_text": [{"type": "text", "text": {"content": "item"}}]}}`
- Bold text within rich_text: `{"type": "text", "text": {"content": "bold text"}, "annotations": {"bold": true}}`
- Divider: `{"object": "block", "type": "divider", "divider": {}}`

**Important**: Notion has a limit of 100 blocks per request and 2000 characters per rich_text element. If your answer is very long, split text across multiple paragraph blocks.

## Step 5: Save a Local Copy

Save the answer locally as a backup:

1. Create the directory `.context/keystone-answers/` if it doesn't exist
2. Generate a slug from the question (e.g., "How does basket refund work?" becomes `basket-refund-behavior`)
3. Write the file to `.context/keystone-answers/YYYY-MM-DD-<slug>.md` with the full answer content

## Rules

1. **Always use Keystone MCP tools** to find answers. Do not guess or rely on assumptions about the codebase.
2. **Show your sources** - include file paths (repo + path) and line numbers when referencing code.
3. **Use read-only operations only** - never modify code, data, or infrastructure through Keystone.
4. **If you can't find the answer**, say so clearly and suggest what the user could try (e.g., which repo to look in, who to ask, what to search for).
5. **For database queries**, always use `LIMIT` clauses and avoid selecting unnecessary columns. PII will be automatically masked by Keystone.
6. **Combine sources** when useful - e.g., find the code with `search_code`, then check recent changes with `git_file_history`, then verify behavior with `replica_query`.
7. **Always publish the answer** to Notion and save a local copy after presenting it. If Notion credentials are unavailable, save locally and inform the user.
