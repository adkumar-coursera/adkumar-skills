---
name: check-claude-review
description: Fact-check an automated Claude PR review by exploring the actual codebase, wiki, and project context. Use when the user asks to check, verify, respond to, or evaluate a Claude code review comment on a PR.
---

# Check Claude PR Review

## What this skill does

An automated LLM-based reviewer bot reviews PRs in many orgs. It runs with minimal context -- typically just the repo's contributor guide (AGENTS.md / CONTRIBUTING.md), the README, and the diff. It has **no knowledge** of:

- The broader project (why something is being built, what phase it's in)
- Project docs that live outside the repo (architecture decisions, design rationale, data flow)
- How the repo fits into the larger system (e.g. that a given service is internal-only, not a user-facing API)
- Intentional design decisions documented outside the diff
- What's already been reviewed in parent PRs of a stack

This leads to false positives: flagging "resource leaks" that are already handled, demanding input validation on internal-only code, insisting on DEBUG over INFO without understanding call frequency, requesting tests that exist in follow-up PRs, suggesting redundant null checks where `@NonNullApi` already applies, etc.

**You have the upper hand.** You have full codebase access, the project docs, and the ability to explore deeply. But this advantage is ONLY real if you actually use it. Do not just read the review comment and the diff and wing it -- that's exactly what the bot did. You must go deeper.

---

## Phase 0 -- Get the review

You MUST have one of:
- **A direct link** to the review comment (e.g. `https://github.com/<org>/<repo>/pull/123#issuecomment-456`)
- **A PR number + repo** so you can fetch comments via `pull_request_read` with `get_comments` and/or `get_review_comments`

If neither is provided, ask the user.

Fetch the review and parse out the individual items/findings. Each numbered finding or section is a separate item to evaluate.

---

## Phase 1 -- Build context (BEFORE evaluating any item)

Read these in order. Do not skip any.

1. **Project docs** -- any architecture docs, ADRs, design docs, or project wiki relevant to the PR's work. Read the full doc if one exists. This is the context the reviewer most likely lacked.
2. **Repo contributor guide** (AGENTS.md / CONTRIBUTING.md) -- the reviewer read this too, so you need to know what it says and whether the reviewer is correctly applying it or misreading it.
3. **The PR summary/body** -- check for `<!-- -->` context comments that may explain intentional decisions.
4. **Parent PRs** (if stacked) -- read their summaries to understand what was already built and reviewed.

This gives you the context the reviewer lacked. Now you can evaluate each item fairly.

---

## Phase 2 -- Evaluate each item (the core of this skill)

Go through EVERY item in the review. Do not skip items just because they look reasonable at a glance -- the bot's reasoning often sounds plausible but is wrong on the facts.

For each item, do the following:

### 2a. Understand what the reviewer claims

Extract:
- What code is being flagged (file, line, pattern)
- What the reviewer says is wrong
- What the reviewer suggests instead

### 2b. Investigate the actual codebase

This is where you earn your keep. Investigate thoroughly -- read callers, grep for patterns, check the wiki.

For each claim, verify it against reality:

| Reviewer claim | How to verify |
|---------------|---------------|
| "This is a resource leak" | Read the full method + callers. Check finally blocks, try-with-resources, calling code. Is the resource actually leaked? |
| "Use DEBUG instead of INFO" | Grep for similar logging in the repo. Check call frequency -- is this once-per-job or once-per-row? What does AGENTS.md say? |
| "Add input validation" | Is this a public API or an internal/private method? Who calls it? Is the input already validated upstream? |
| "Add null check" | Is the package annotated with `@NonNullApi`? Does the type system prevent null? Is `@Nullable` used? |
| "No test coverage" | Are tests in a follow-up PR? Check the PR description, branch names, or ask the user. |
| "Use X library instead of Y" | What does the rest of the codebase use? Check `build.gradle`/`package.json`. Is the reviewer suggesting a dep that isn't even available? |
| "This pattern is not thread-safe" | Read the threading model. Is this code actually accessed concurrently? Check callers. |
| "Consider making this configurable" | Is this a constant that realistically needs to change? Or is the reviewer adding unnecessary complexity? |

### 2c. Form your verdict

For each item, classify it:

| Verdict | Meaning |
|---------|---------|
| **Valid -- worth fixing** | The reviewer found a real issue. Explain why and suggest the fix. |
| **Valid but low priority** | Real issue, but minor. Note it for future cleanup. |
| **Partially valid** | The reviewer has a point, but their suggestion is wrong or there's a better approach. Explain the better path. |
| **Invalid -- false positive** | The reviewer is wrong. Explain why with evidence from the codebase. |
| **Already handled** | The concern is addressed by code the reviewer didn't see or context they lacked. Point to the evidence. |
| **Out of scope** | The suggestion is fine in theory but belongs in a different PR/task. |

---

## Phase 3 -- Write the report

Present findings to the user:

```markdown
# Claude Review Check: PR #NNN

## Context

<1-2 sentences: what the PR does, what project it's part of, what context the reviewer was missing>

## Item-by-item evaluation

### Item N: <brief title from reviewer>

**Reviewer says:** <1-2 sentence summary of the finding>
**Severity claimed:** <what the reviewer marked it as>

**Investigation:** <What you found by exploring the codebase. Reference specific files, lines, grep results, wiki docs. Show your work.>

**Verdict:** <Valid/Invalid/Partially valid/etc.>
**Recommendation:** <What to actually do -- fix it, ignore it, file a follow-up, or do something different than what the reviewer suggested>

---

<repeat for each item>

## Summary

| # | Item | Reviewer severity | Verdict | Action |
|---|------|------------------|---------|--------|
| 1 | ... | Critical | Invalid | Ignore |
| 2 | ... | Medium | Valid | Fix now |
| ... | | | | |

**Bottom line:** <N of M items are valid. The reviewer was [mostly right / mostly wrong / mixed]. Key takeaways: ...>
```

### Report guidelines

- **Show your evidence.** Every verdict must reference something concrete -- a file path, a grep result, a wiki doc, a caller chain. "I think it's fine" is not a verdict.
- **Be honest when the reviewer is right.** The goal isn't to dismiss everything -- it's to separate signal from noise. If the bot found a real bug, say so clearly.
- **Suggest better fixes.** When the reviewer is partially right but their suggestion is wrong, propose the correct approach. Sometimes the reviewer identifies a real smell but prescribes the wrong medicine.
- **Note deeper root causes.** Sometimes a review item is technically valid but is a symptom of a broader pattern. Flag these -- e.g. "the reviewer flagged inconsistent error handling in this file, but the real issue is that the repo has no consistent error handling convention at all -- worth a contributor-guide update."
