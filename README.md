# Coding Agent Skills

A collection of skills for everyday backend/SWE work — PR shepherding, code review, build debugging, and a few utilities. They started as personal workflow automations and have been generalized to work in any repo, not a specific codebase.

Each skill is a single markdown file: a name, a one-line description, and a step-by-step playbook in the body. They're authored in [Claude Code](https://claude.com/claude-code)'s skill format, but the body of every skill is just a self-contained, agent-neutral playbook — there's nothing Claude-specific about *how to debug a Gradle build* or *how to fact-check a review*. Any capable coding agent can run them.

## Installation

### Claude Code

Skills live under `~/.claude/skills/`. Clone the repo and symlink (so `git pull` keeps them current):

```bash
git clone https://github.com/adkumar-coursera/adkumar-skills.git ~/code/adkumar-skills
ln -s ~/code/adkumar-skills/* ~/.claude/skills/
```

Claude Code reads the descriptions and invokes the matching skill when your request fits — or trigger one explicitly with `/<skill-name>`. (Copy individual directories instead of symlinking if you only want some.)

### Other agents (Cursor, Cline, Copilot, Aider, …)

There's no auto-discovery, but each skill is a portable playbook — drop the body into whatever context mechanism your agent uses:

- **Cursor / Windsurf** — paste into a project rule (`.cursor/rules`) or `@`-reference the file.
- **Cline / Roo** — use as a custom workflow or paste into the task prompt.
- **Aider / Copilot / plain chat** — paste the skill body into your message, or keep it in a conventions/context file the agent reads.

The frontmatter (`name` / `description`) is only used by Claude Code's auto-invocation; other agents can ignore it and use the steps directly.

## Skills

### PR & review workflow

| Skill | What it does | When it helps |
|-------|--------------|---------------|
| **babysit-pr** | Loops a PR to merge-readiness: pulls `main`, refreshes the automated review, waits for checks to go green, evaluates review findings, runs Sonar (Java repos), and fixes or rebuts as it goes. | You opened a PR and don't want to babysit the review/CI cycle by hand. Stops and asks only when it hits something needing your judgment. |
| **write-pr-summary** | Writes or improves a PR description from the git diff. Detects the repo's own PR template, handles stacked and multi-repo companion PRs, and adds a context block for automated reviewers. | Turning a diff into a reviewer-friendly summary that explains *why*, not just *what*. |
| **check-llm-review** | Fact-checks an automated LLM PR review against the actual codebase and project docs, classifying each finding as valid / false-positive / already-handled / out-of-scope. | The review bot flagged things and you want a grounded second opinion before acting. |

### Code review & quality

| Skill | What it does | When it helps |
|-------|--------------|---------------|
| **consistency-review** | Diffs a branch against the established codebase to find deviations in naming, libraries, error handling, test patterns, and structure — judging the change against how the repo *actually* does things. | Catching "this works but doesn't match the codebase" issues a normal review misses. |
| **sonar-review** | Fetches SonarQube/SonarCloud issues for a project or PR via whatever Sonar MCP you have connected, and presents them as a clean table. | Pulling Sonar findings into your session without leaving the terminal. |

### Build & performance

| Skill | What it does | When it helps |
|-------|--------------|---------------|
| **java-build-debug** | Runs and/or diagnoses a Gradle/Java build. Reads the log bottom-up, finds the failed task, and isolates the root-cause error — including annotation-processor (Immutables/MapStruct/etc.) cascade failures where one real error spawns dozens of fake ones. | A build failed and you want the *actual* cause, not the 50 cascade errors hiding it. |
| **performance-analysis** | A log-based profiling loop: instrument with timing/memory markers, trigger the flow, collect logs, build a timeline + bottleneck breakdown, write up the findings, and revert the instrumentation. Language- and log-source-agnostic. | Profiling a backend flow when a real profiler isn't practical and log instrumentation is. |

### Utilities & communication

| Skill | What it does | When it helps |
|-------|--------------|---------------|
| **read-google-doc** | Reads and summarizes a Google Doc by URL or ID, chunking automatically for docs that blow past MCP token limits. | Pulling a design doc / PRD into context without manual copy-paste. |
| **caveman** | Ultra-compressed response mode — strips filler while keeping full technical accuracy. Several intensity levels. | Cutting token usage / noise when you want terse, signal-only answers. |

## Behavioral guidelines (`CLAUDE.md`)

`CLAUDE.md` holds a small set of working principles — think before coding, simplicity first, surgical changes, goal-driven execution. Drop it into a project (or merge into your agent's own conventions/context file — `CLAUDE.md`, `AGENTS.md`, `.cursorrules`, etc.) to bias toward smaller diffs and fewer speculative changes. Several skills reference these principles by name.

## Prerequisites

Most skills are self-contained, but some lean on external tooling:

| Skill | Needs |
|-------|-------|
| babysit-pr, check-llm-review, write-pr-summary | `gh` CLI and/or a GitHub MCP server |
| sonar-review | a Sonar MCP server; token via the `SONAR_TOKEN` env var if the tool requires one |
| read-google-doc | a Google Workspace MCP server (optional `docs-tools` helper for chunked reads) |
| java-build-debug | a Gradle-based Java project |

No secrets or credentials are stored in this repo — anything sensitive is read from the environment at runtime.
