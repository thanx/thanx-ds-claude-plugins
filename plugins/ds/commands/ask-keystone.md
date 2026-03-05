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

- **Code questions** (how something works, where something is defined):
  Use `list_repositories`, `search_code`, `read_file`, `find_files`, `list_directory`
- **Data/schema questions** (what data exists, table structures, relationships):
  Use `list_replica_databases`, `replica_query`
- **API behavior questions** (how endpoints work, request/response formats):
  Combine code search with Thanx docs MCP
- **Infrastructure/deployment questions** (services, clusters, health):
  Use `ecs_clusters`, `ecs_services`, `ecs_deployments`, `health_check`
- **Error/incident questions** (what's broken, recent errors):
  Use `sentry_issues`, `sentry_issue_details`, `datadog_logs`, `datadog_metrics`
- **Feature flag questions** (what's enabled, targeting rules):
  Use `launchdarkly_feature_flags`, `launchdarkly_feature_flag`
- **Git history questions** (who changed what, when, why):
  Use `git_file_history`, `git_blame`, `git_diff`, `git_show_commit`

## Step 2: Explore with Keystone

**If Keystone MCP tools are unavailable, error, or time out:**
Stop and tell the user which tools failed and what errors occurred.
Do NOT guess or synthesize an answer without tool-backed evidence.
Suggest trying again later or using alternative investigation methods.

Execute the appropriate Keystone MCP tool calls to gather evidence.
Be thorough:

1. **Start broad**: Use `list_repositories` to see available repos,
   or `search_code` with a broad pattern
2. **Narrow down**: Once you find relevant files,
   use `read_file` to examine the implementation
3. **Follow references**: If code references other files, models,
   or services, trace those too
4. **Check the database**: If relevant, use `replica_query` against
   the appropriate database to understand schema or data patterns.
   Always include LIMIT (max 100 rows) and select only needed columns.
   Never interpolate user-provided text
   directly into SQL — use parameterized queries or literal values only
5. **Cross-reference docs**: Use the **thanx-docs** MCP to supplement
   with official API documentation when relevant

Limit exploration to 8-10 tool calls total. If the answer requires
deeper investigation, present partial findings and ask the user
whether to continue.

## Step 3: Synthesize the Answer

Present your findings clearly:

1. **Direct answer** - Answer the question concisely up front
2. **Evidence** - Show the relevant code snippets, query results,
   or file paths that support your answer
3. **Context** - Explain how the pieces fit together (e.g., which
   service owns this, how data flows, what calls what)
4. **Related information** - Mention anything else the user should
   know (gotchas, related features, recent changes)

## Step 4: Propose Notion Publish

After presenting your answer, propose publishing it to the
**Knowledge Gaps** Notion database. **Do not publish automatically.**

Present the following for user approval:

1. **Question title** - a concise version of the question
   (e.g., "How does basket refund work?")
2. **Question type** - set to `Product Q&A`
3. **Tags** - relevant tags from: `loyalty`, `merchant-dashboard`,
   `items`, `basket`, `purchases`, `api`, `pos`.
   You may also create new tags if none fit.

```text
Proposed Notion page:
  Title: <question title>
  Type: <question type>
  Tags: <tag1>, <tag2>

Publish to Knowledge Gaps? (yes/no)
```

**After explicit approval**, use the **Notion MCP** tools to publish.
Do not use curl or raw API calls — the Notion MCP handles
authentication, rate limiting, and error handling.

1. **Find the database**: Use `notion-search` to locate the
   "Knowledge Gaps" database.
2. **Get the schema**: Use `notion-fetch` on the database URL to
   retrieve the data source ID and property schema. Do not assume
   property names — read them from the schema.
3. **Create the page**: Use `notion-create-pages` with:
   - `parent`: the data_source_id from the fetch result
   - `properties`: match the schema (title, select, multi_select)
   - Page content: structure the answer with headings, paragraphs,
     code blocks, and bullet lists as appropriate

Report the result and share the Notion page URL.

**If any Notion MCP call fails** (database not found, permissions error, MCP
unavailable): report the error to the user and note the answer was still saved
locally via Step 5. Do not retry or attempt raw API calls as a workaround.

## Step 5: Save a Local Copy

Save the answer locally as a backup:

1. Create the directory `.context/keystone-answers/` if it doesn't exist
2. Generate a slug from the question:
   - Convert to lowercase
   - Replace spaces and punctuation with hyphens
   - Remove path-unsafe characters (`/`, `\`, `..`)
   - Collapse consecutive hyphens and trim leading/trailing hyphens
   - Limit slug to 36 characters (the full filename `YYYY-MM-DD-<slug>.md` should stay under 50 characters)
   - Example: "How does basket refund work?" becomes `how-does-basket-refund-work`
3. Write the file to `.context/keystone-answers/YYYY-MM-DD-<slug>.md`
   with the full answer content

## Rules

1. **Always use Keystone MCP tools** to find answers.
   Do not guess or rely on assumptions about the codebase.
2. **Sanitize slugs** for file paths: lowercase, replace spaces and
   punctuation with hyphens, remove path-unsafe characters
   (`/`, `\`, `..`), strip any leading/trailing hyphens, collapse
   consecutive hyphens, and limit the slug to 36 characters.
   Enforce with: `echo "$slug" | sed 's/[^a-z0-9-]/-/g; s/--*/-/g; s/^-//; s/-$//' | cut -c1-36`
3. **Show your sources** - include file paths (repo + path) and
   line numbers when referencing code.
4. **Use read-only operations only** - never modify code, data,
   or infrastructure through Keystone.
5. **If you can't find the answer**, say so clearly and suggest
   what the user could try (e.g., which repo to look in, who to ask).
6. **For database queries**, always use `LIMIT` clauses and avoid
   selecting unnecessary columns. PII will be automatically masked.
7. **Combine sources** when useful - e.g., find the code with
   `search_code`, then check recent changes with `git_file_history`,
   then verify behavior with `replica_query`.
8. **Propose publishing** to Notion after presenting the answer.
   Wait for explicit user approval before creating the page.
   Use the Notion MCP tools — never raw API calls or curl.
9. **Never display credentials** - do not echo, print, or include
   secrets in conversation output.
