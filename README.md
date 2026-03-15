# Sync Repository Example — GitHub

This directory contains a complete example of a repository structure that can be synced by the `SyncService`. It demonstrates a Page MCP package targeting **GitHub repository detail pages** (e.g. `github.com/owner/repo`).

## Directory Structure

```
examples/
├── README.md
├── repository.json              # Repository metadata
└── mcp/
    ├── prompts.json              # Prompt definitions
    ├── resources.json            # Resource definitions
    └── tools.json                # Tool definitions
```

---

## `repository.json`

Root manifest for the repository.

```json
{
  "name": "github-repo-mcp",
  "siteDomain": "github.com",
  "version": "1.0.0"
}
```

| Field         | Required | Description                         |
| ------------- | -------- | ----------------------------------- |
| `name`        | ✅       | Display name                        |
| `description` | ❌       | Short description                   |
| `siteDomain`  | ✅       | Target website domain               |
| `version`     | ✅       | Semantic version                    |

---

## `mcp/resources.json` — Page DOM Resources

Resources use the Page MCP URI scheme to extract data directly from the page DOM.

### URI Conventions

| Mode         | URI Format                         | Description            |
| ------------ | ---------------------------------- | ---------------------- |
| CSS Selector | `page://selector/<css-selector>`   | Uses `querySelector`   |
| XPath        | `page://xpath/<xpath-expression>`  | Uses `evaluate` (XPath)|

### Included Resources

#### `username` — CSS Selector

Extracts the repository owner's username from the page header.

```json
{
  "name": "username",
  "uri": "page://selector/a.url.fn",
  "mimeType": "text/plain",
  "path": "^/[^/]+/[^/]+/?$"
}
```

> On a GitHub repo page, the element `<a class="url fn">` in the header breadcrumb contains the owner name (e.g. "anthropics").

#### `project_name` — XPath

Extracts the repository name using an XPath expression.

```json
{
  "name": "project_name",
  "uri": "page://xpath//strong[@itemprop='name']/a",
  "mimeType": "text/plain",
  "path": "^/[^/]+/[^/]+/?$"
}
```

> The `<strong itemprop="name"><a>` element in the repo header contains the repo name (e.g. "anthropic-sdk-python").

---

## `mcp/prompts.json` — Prompt Templates

### `explain-readme` — With Argument

Interprets and explains the repository's README. Accepts an optional `language` argument.

```json
{
  "name": "explain-readme",
  "arguments": [
    { "name": "language", "description": "...", "required": false }
  ],
  "messages": [
    { "role": "user", "content": { "type": "text", "text": "Please read the README..." } }
  ]
}
```

### `summarize-repo` — Without Arguments

Gives a quick 2-3 sentence project summary based on the README.

```json
{
  "name": "summarize-repo",
  "messages": [
    { "role": "user", "content": { "type": "text", "text": "Based on the README..." } }
  ]
}
```

---

## `mcp/tools.json` — Browser-Executed Tools

Tools use the `execute` field to run JavaScript in the browser context, extracting data from the page DOM.

### `get-readme` — Without InputSchema

Extracts the rendered README content from `article.markdown-body`.

```json
{
  "name": "get-readme",
  "execute": "() => { const article = document.querySelector('article.markdown-body'); ... }"
}
```

### `search-repo` — With InputSchema

Searches within the current repository on GitHub. Accepts `query` (required) and `type` (optional: code/issues/discussions).

```json
{
  "name": "search-repo",
  "inputSchema": {
    "type": "object",
    "properties": {
      "query": { "type": "string" },
      "type":  { "type": "string" }
    },
    "required": ["query"]
  },
  "execute": "async () => { ... window.location.href = searchUrl; }"
}
```

---

## Path Matching

All items use the path pattern `^/[^/]+/[^/]+/?$`, which matches GitHub repository detail pages like:

- ✅ `/anthropics/anthropic-sdk-python`
- ✅ `/facebook/react/`
- ❌ `/anthropics/anthropic-sdk-python/issues`
- ❌ `/settings`

The `path` field is a **regex** validated by `isValidRegexPattern()` in the sync service.

---

## How the Sync Service Uses These Files

The `SyncService` fetches these files from the GitHub repository's default branch:

1. **Fetches** `repository.json` and all `mcp/*.json` files in parallel
2. **Validates** required fields and path patterns (must be valid regex)
3. **Stores** the data into the database — creating/updating prompts, resources, and tools

Refer to [`backend/src/services/sync.service.ts`](../backend/src/services/sync.service.ts) for the full implementation.
