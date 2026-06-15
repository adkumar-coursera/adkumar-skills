---
name: read-google-doc
description: Read and summarize a Google Doc by URL or document ID. Handles large docs that exceed MCP token limits by chunking automatically. Use when the user shares a Google Docs link or asks to read/review/check a Google Doc.
---

# Read Google Doc

## Input

The user provides either:
- A Google Docs URL (e.g. `https://docs.google.com/document/d/SOME_ID/edit...`)
- A raw document ID

Extract the document ID from the URL. The ID is the segment between `/d/` and the next `/`.

## Step 1: Fetch the document

Use `mcp__google-workspace__docs__get_document` with the extracted document ID. If you need the deferred tool schema, fetch it first with ToolSearch.

## Step 2: Handle oversized results

The MCP tool returns JSON `{id, title, content, webViewLink}`. For large docs, the result gets saved to a file under `.claude/projects/` because it exceeds MCP token limits.

When this happens, an optional `docs-tools` MCP server makes chunked reading easy. (It's a small helper MCP — one such implementation lives at `github.com/adkumar-coursera/adkumar-mcp-tools`. If you don't have it, skip to the Bash fallback below, which needs no extra tooling.) With `docs-tools` available, read the content in chunks:

1. First, get doc metadata and size:
   - Use `mcp__docs-tools__doc_info` with the saved file path to get the title, content length, and suggested chunk count.

2. Then read in chunks:
   - Use `mcp__docs-tools__read_doc` with `start=0, length=7000`, then `start=7000, length=7000`, etc.
   - Run multiple chunk reads in parallel when possible to speed things up.

3. To find specific sections quickly:
   - Use `mcp__docs-tools__search_doc` with a query string to jump to relevant sections instead of reading the entire doc sequentially.

**If docs-tools is not available** (deferred tool not loaded, or MCP server not running), fall back to Bash:
```bash
cat THE_FILE | python3 -c "import json,sys; d=json.load(sys.stdin); print(d['content'][START:END])"
```

## Step 3: Respond to the user

After reading the full document, respond to whatever the user asked about it. If they just said "read this doc," provide a concise summary of the key points. If they asked a specific question, answer it.

Do NOT narrate the chunking process. Just answer naturally.
