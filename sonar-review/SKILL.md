---
name: sonar-review
description: Fetch and display SonarQube/SonarCloud issues for a project or PR via an available Sonar MCP server. Use when the user asks to check sonar, show sonar issues, or review sonar for a PR or project.
---

# SonarQube / SonarCloud Issue Fetch

Fetch issues from SonarQube/SonarCloud and present them to the user.

## Step 0 — Pick a Sonar MCP server

This skill needs a Sonar MCP tool. Discover what's connected (e.g. via ToolSearch with a query like `sonar`) and pick a suitable one. Common shapes:
- An official SonarQube MCP (tools named like `*sonarqube*search_sonar_issues*`).
- A custom Sonar wrapper MCP exposing a `search_issues`-style tool.

Use whichever is available. If none is, tell the user this skill needs a Sonar MCP server configured, and stop.

### Authentication

**Never hardcode a token.** If the chosen MCP requires a token argument, read it from the `SONAR_TOKEN` environment variable. If the MCP authenticates via its own server configuration, no token is needed here.

## Step 1 — Resolve the project key and PR number

### Project key

SonarCloud's default project key is `<org>_<repo>` (e.g. GitHub org `myorg` + repo `my-service` → `myorg_my-service`). Self-hosted SonarQube keys may differ — if you can't derive it confidently, ask the user.

### From a GitHub PR URL

`https://github.com/<org>/<repo>/pull/<N>` → project key `<org>_<repo>`, PR number `<N>`.

### If you cannot determine the project key

Stop and ask the user. Do not guess.

### PR vs branch

- **PR number provided** (via URL or explicitly): use it.
- **Not provided**: ask whether to query the main branch or a specific PR.
- Note: many setups only run Sonar analysis on PRs, not branches, so main-branch results may be stale or absent for recent changes.

## Step 2 — Determine filters (optional)

Check whether the user specified:
- **Severity** — e.g. "only critical and blocker"
- **Type** — e.g. "only bugs"
- **Tags** — e.g. "security"
- **Include resolved** — only if explicitly asked. Default: unresolved only.

If no filters are mentioned, fetch everything.

## Step 3 — Call the MCP tool

Call the chosen Sonar tool with the resolved project key, the PR number (or omit for the main branch), and any filters. Pass the token from `SONAR_TOKEN` only if the tool requires an explicit token argument.

## Step 4 — Output

Present the results as a readable markdown table. Keep the table itself neutral — don't editorialize on each row. If the result is empty, say so clearly and confirm the filters and PR number used.

## Step 5 — Offer to fix

After presenting the results (and only if there are issues), ask the user whether you should fix them — e.g.:

> Want me to fix these? I can address all of them, or you can tell me which ones (by number/rule) to take.

Wait for the user's answer before changing any code. If they decline, stop here. If they accept, fix the issues they selected, keeping changes surgical and scoped to what Sonar flagged.
