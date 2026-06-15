---
name: write-pr-summary
description: Write or improve a GitHub PR summary by analyzing the git diff. Handles stacked/chained PRs (branch -> branch -> main) and multi-repo companion sets. Use when the user asks to write, rewrite, or improve a PR description, PR summary, or pull request body.
---

# Write PR Summary

## Before writing — gather context

1. **Ticket reference** — ask for the issue/ticket ID (e.g. `TICKET-123`) if the project uses a tracker. Skip if it doesn't.
2. **Project docs** — ask if there are design docs, ADRs, or a project README relevant to this work. Read them to understand *why* the change exists, not just *what* changed. If pointed at a directory, read every relevant doc in it.
3. **PR chain** — ask if this is part of a stacked chain (same repo, branch -> branch -> main) and/or a multi-repo companion set (e.g. a schema PR paired with an implementation PR). Note both.
4. **Screenshots** — for UI or template changes, ask if the user has before/after (or state-specific) images to embed.

## Gathering the diff

Prefer local git diffs — the GitHub API may 404 on private repos.

### Identify the PR's base and head branches

Ask the user or infer from context:
- **Simple PR:** `feature -> main` → `git diff main...feature`
- **Stacked PR:** `branch2 -> branch1` → `git diff branch1...branch2`

### Commands

```bash
git diff <base>...<head> --stat                           # file-level overview
git log <base>..<head> --oneline                          # commit messages for context
git diff <base>...<head> -- '*.java' '*.ts' '*.py' '*.go' # full diff (filter to code files)
```

For stacked PRs the three-dot diff (`...`) is critical — it shows only what the head branch added on top of the base, excluding anything already in the base.

## Detect the repo's PR template

Before writing, check whether the repo prescribes a structure:
- `.github/PULL_REQUEST_TEMPLATE.md` (or `.github/PULL_REQUEST_TEMPLATE/` for multiple)
- A `CONTRIBUTING.md` or contributing guide that dictates PR sections

If a template exists, **follow its structure** — fill in its sections rather than imposing your own. If none exists, use the default template below.

## Default template

```markdown
[**Ticket**](<tracker-url>)
<!-- Include only if the project uses an issue tracker -->

> **Stacked on #NNN** — review that first.
<!-- Include this line ONLY for chained PRs that depend on another PR -->

> **Companion PRs:** <repo> #NNN (merge first), <repo> #NNN
<!-- Include this line ONLY when this PR is part of a multi-repo set -->

## Summary

[1-2 sentence overview of WHAT this PR does and WHY. Use project-doc context to explain the motivation -- don't just describe the diff.]

[Optional **Key behaviors:** bullets if there are multiple non-obvious changes. Scale to the PR -- small PRs don't need bullets, the sentence is enough.]

[Screenshots here, if applicable -- embed inline after Key behaviors.]

<details>
<summary>n files changed</summary>

| Area | File | Change |
|------|------|--------|
| ... | `FileName.java` | Brief description |

</details>

## Test Plan

- [x] Unit tests pass
  - [brief note on what was added/changed, e.g. "7 parameterized filter combos"]
- [ ] [Manual/preview testing step, left unticked for the user to confirm after testing]
```

## Writing guidelines

- **ASCII only.** Use `--` not em-dashes, `->` not arrows, plain quotes not curly quotes. Use unicode only when the content genuinely requires it (e.g. a field name contains it).
- **Keep it short.** A good summary is 5-15 lines above the `<details>` fold. If it's getting long, the PR is probably too big -- recommend a split instead of writing a novel.
- **Context over diff narration.** Explain *why* something exists, not just *what* changed. "Uses composite aggregation because the index doesn't support cardinality estimates" beats "Adds composite aggregation".
- **Lead with behavior, not files.** Describe what the code *does*. The files-changed table goes in `<details>`; use `n files changed` (e.g. `8 files changed`) as the summary text so reviewers expand only if they want the file map.
- **Scale to the PR.** A 10-line config change needs one sentence. A 500-line new module might need 3-4 bullets. Don't pad small PRs.
- **Stacked PRs:** note the parent dependency at the top with a blockquote.
- **Multi-repo companions:** note the companion PRs and their merge order at the top with a blockquote.
- **Referencing other PRs:** link them as `repo-name/number` (e.g. `my-service/100`), always in this format.
- **Test-plan honesty:** for schema/config/template-only PRs, use `N/A` or a real verification line -- do NOT write a fake "unit tests pass" bullet.

## Context comments for automated reviewers

Many orgs run an automated LLM-based PR reviewer that sees only the repo's contributor guide (AGENTS.md / CONTRIBUTING.md) and the diff -- no broader project context. This produces false positives: flagging "resource leaks" that are already handled, demanding input validation on internal-only code, insisting on DEBUG over INFO without knowing call frequency, requesting tests that live in a follow-up PR.

If your repo has such a reviewer, preempt it with an HTML comment block near the top of the PR body. Use `<!-- -->` so it's invisible to humans but parsed by the bot:

```markdown
<!--
Context for reviewers: [what this repo/service is and does -- the bot may not know].
[Project phase, what the parent PR built, why this approach was chosen.]
Intentional decisions:
- [e.g. INFO logging is fine here -- runs at most once per job, not per-request]
- [e.g. no input validation -- private method called only from an already-validated entry point]
- [e.g. tests for the new path are in a follow-up PR; existing tests are updated to compile]
-->
```

Scale the comment to the PR. Omit it entirely if your repo has no automated reviewer.

## Updating an existing summary

- **Preserve user-filled content.** Check what's already there first (`gh pr view` or the GitHub MCP). Ticked checkboxes, hand-written notes, a previously-written perf section -- don't overwrite these unless something actually changed or the user asks. When in doubt, preserve.
- **`<details>` tags are stripped by the GitHub MCP read.** When you read an existing PR body via the GitHub MCP, `<details>` / `</details>` tags are silently removed from the response. Do NOT interpret their absence as evidence they're missing from the real PR. **Always emit `<details>` / `</details>` wrappers** around the files-changed table (and any long section) in your output, regardless of what the read showed.

## Performance analysis section (if applicable)

If the work involved performance profiling and a perf-analysis doc exists, add a **Performance Analysis** section after the Test Plan, wrapped in `<details>` to keep the PR scannable:

```markdown
<details>
<summary>Performance Analysis</summary>

[Methodology: environment, commit, instrumentation approach, params tested]

[1-sentence verdict on whether the optimization works as expected]

**Before vs after**

Old: **Xms** -> New: **Yms** = **N% reduction (Mx faster)**

| Query/Component | Old | New |
|---|---|---|
| ... | Xms | Yms |

**Key findings**

- [what dominates, what's negligible, impact on the broader system]

</details>
```

- **Always `<details>`** -- perf data is valuable but secondary to the summary.
- **Read the perf doc -- don't invent numbers.** Pull exact timings and findings from it.
- **Include before vs after** -- reviewers want the delta, not just absolute numbers.
