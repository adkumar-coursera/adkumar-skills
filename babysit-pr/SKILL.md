---
name: babysit-pr
description: Watch a PR through the review/merge readiness cycle. Each cycle pulls main, refreshes the automated review, waits for checks to go green, evaluates the latest review via the check-llm-review skill, and (in Java repos) runs sonar-review against the latest commit. Loops until all exit conditions are met. Use when the user asks to babysit, watch, monitor, tend, or shepherd a PR until it is ready to merge.
---

# Babysit PR Until Solid

## When to use

When the user asks to babysit, watch, monitor, or shepherd a PR through the final review/merge cycle. The intent is "keep this PR healthy until it's a clean merge candidate" — not just "check it once."

This skill defers to two siblings, so updates flow through them automatically:
- **`check-llm-review`** — read it before each review evaluation so any change there is picked up here.
- **`sonar-review`** — used to fetch SonarCloud issues for the latest commit, when the repo runs SonarCloud.

## Step 0 — Identify the PR and the expected branch

Prefer to infer from conversation context (recent PR number, branch name, repo path). If unsure, ask once:
> "Which PR? `<repo>/<number>` or full GitHub URL."

Resolve the PR's head branch, base, and author — you'll need all three for the rest of the cycle:
```bash
gh pr view <number> --json number,headRefName,baseRefName,url,headRepository,author -R <repo>
```

**Record:**
- `headRefName` → `$EXPECTED_BRANCH`: source of truth for which branch the local repo must be on for any read/write.
- `author.login` → `$PR_AUTHOR`: used to filter out the PR author's own comments when scanning for human reviewer input.

## Step 1 — Read the `check-llm-review` skill once

Before starting the loop, read the `check-llm-review` skill and follow its phases when you evaluate any automated review during the cycle. Do not duplicate its logic here — defer to it.

## Step 2 — The cycle (repeat until exit conditions hold)

Each iteration does these steps in order. If any step decides "exit", stop the loop and report.

### Track the phases with a todo list

Use `TaskCreate` to record the phases of the **current iteration** at the start. Suggested phase titles (skip steps that don't apply this iteration):

1. Verify branch + check for human input
2. Pull `origin/main` and resolve any merge conflicts
3. Refresh automated review (rerun job)
4. Wait for checks to go green
5. Evaluate automated review (🔴 + 🟡)
6. Run sonar-review and address new issues (Java repos only)
7. Decide exit / loop

Mark each phase `in_progress` when you start it and `completed` when it finishes cleanly.

**Going back a phase (very important).** Any time the cycle effectively rewinds — you applied a fix and pushed (which restarts checks), main was repulled, you switched branches, you re-triggered the review job, you discovered a sonar issue that needs a fix — **un-complete every downstream task** by setting it back to `pending` via `TaskUpdate`. The state of the world after a push is "we don't know if checks pass, we don't know what the new review says, we don't know what sonar will say" — the tasks must reflect that. Tasks that are *upstream* of the rewind stay completed if they were genuinely done.

If a cycle iteration completes the full chain without rewinding, mark all tasks completed and the loop exits via Step 2f.

### General precondition — verify branch before any local file operation

Before **every** step that reads or writes local files (everything in section 2 except read-only `gh` API calls against the PR), confirm the working tree is on `$EXPECTED_BRANCH`:

```bash
current=$(git rev-parse --abbrev-ref HEAD)
[ "$current" = "$EXPECTED_BRANCH" ] || git checkout "$EXPECTED_BRANCH"
```

If the checkout itself fails (dirty tree with unrelated changes, branch missing locally), that's a state you can't safely infer through — surface to the user and stop.

### General precondition — check for human reviews/comments (stop if any)

At the **start of every cycle iteration**, before doing anything else with the PR, check for any review or comment from a non-bot, non-user author. If any are found, **stop the loop and surface them to the user** — this skill only auto-handles Sonar and Claude bot signals.

```bash
# Formal reviews (APPROVE / REQUEST_CHANGES / COMMENT submitted via the Reviews flow)
gh api repos/<repo>/pulls/<number>/reviews \
  --jq '.[] | select(.user.type != "Bot") | {id, user: .user.login, state, submitted_at, body_preview: (.body | .[0:200])}'

# Inline review comments (left on specific lines)
gh api repos/<repo>/pulls/<number>/comments \
  --jq '.[] | select(.user.type != "Bot") | {id, user: .user.login, path, line, created_at, body_preview: (.body | .[0:200])}'

# Issue-style comments on the PR thread
gh api repos/<repo>/issues/<number>/comments \
  --jq '.[] | select(.user.type != "Bot") | {id, user: .user.login, created_at, body_preview: (.body | .[0:200])}'
```

Bots are filtered by `.user.type == "Bot"`. **Also filter out the PR author** (your own rebuttal comments and acks don't count as new input) — get the author once at Step 0 via `gh pr view ... --json author --jq .author.login` and store as `$PR_AUTHOR`. Treat any record whose `.user.login != $PR_AUTHOR` and `.user.type != "Bot"` as a human-reviewer signal.

If anything matches: stop the loop, print a tight summary (who, when, where, the first ~200 chars of the body, plus a permalink to each comment), and wait for the user's direction. Do not attempt to fix, rebut, or address human review content automatically — that's outside this skill's mandate.

If nothing matches: continue to step 2a.

### General postcondition — run the local build after any write

After **any** local write operation in section 2 (resolving a merge conflict, applying a 🔴 fix, applying a sonar fix), run the repo's local build and tests before committing/pushing. The goal: catch random or unrelated breakages locally instead of letting CI burn 8-10 minutes finding them.

**Java/Gradle (the default for most JVM backend repos):**
```bash
./gradlew spotlessApply build --console=plain > build.log 2>&1
grep -E "BUILD (SUCCESSFUL|FAILED)" build.log
```

Read `build.log` only if it failed; `BUILD SUCCESSFUL` is the entire signal you need otherwise. If it failed, invoke `/java-build-debug` to diagnose it. (Spotless/formatting violations resolve themselves on a second run -- the first reformats, the second confirms; compile or test failures need real fixes.)

After a clean `BUILD SUCCESSFUL`, `git add` and `git commit -m "<short subject>"`. Then push.

**Scala/sbt:**
```bash
# sbt may not be installed locally; if it isn't, push and let CI verify.
which sbt && sbt "test:compile; test" || echo "sbt unavailable — push and let CI verify"
```
This is the rare exception — most of the time we can't build sbt locally, so we rely on CI. If CI then fails, treat that as the build-failure path: investigate the failing job's logs, fix, re-push.

**Repos with neither gradle nor sbt** (e.g. schema-only repos): use whatever the repo's CI runs, typically a `make test` or `npm test`. If unclear, check the repo's CI workflow file.

Never push without running the build (when available) — finding "you broke `ItemAttemptsManagerImplTest`" via local build takes 30s; via CI it takes 10 minutes per round-trip.

### 2a. Pull `origin/main` into the branch

Make sure we're not drifting away from main and that no conflicts have appeared.

```bash
git fetch origin main
git merge origin/main --no-edit
```

**If `git merge` reports conflicts: try to resolve them yourself first.** Most main-into-branch conflicts in this codebase have an evident resolution from context — read both versions of each conflicting hunk, check git history of the file (`git log --oneline -- <file>`) to understand what each side changed, and apply the obvious merge. Examples of "evident":
- Import lists where both sides added different imports — keep both, sorted.
- Test files where main added a constructor param and our branch updated callers — combine straightforwardly.
- HOCON/config blocks where both sides added new top-level keys — keep both.

After resolving, `git add` the files and `git commit --no-edit` (uses the default merge commit message). Then re-run the build/test if local tooling is available.

**Only stop the loop and surface to the user if the conflict genuinely needs human judgment** — e.g. both sides edited the same logic in semantically incompatible ways, or the conflict touches code you don't have context for and can't reason about safely from `git log` and surrounding files alone.

### 2b. Refresh the automated review

The automated PR review bot re-runs automatically on each new commit, but if this cycle iteration started without a new commit, manually re-trigger its workflow so the latest review reflects current state:

```bash
gh run list --repo <repo> --workflow "Claude Code PR Assistant" \
  --branch <head_branch> --limit 1 --json databaseId --jq '.[0].databaseId'
gh run rerun <runId> --repo <repo>
```

(Exact workflow name varies by repo — `gh workflow list -R <repo>` to find it. Common: "Claude Code PR Assistant", "Claude Code Review".)

The bot edits the **same comment** on the PR each run, so once the rerun finishes, read the same comment ID to get the fresh review.

### 2c. Wait for all checks to go green

```bash
gh pr checks <number> --repo <repo> --json name,state,conclusion
```

Poll every ~5 minutes (`sleep 300`-style background command, or chain a short wait + re-check). All check runs should reach `state: COMPLETED` and `conclusion: SUCCESS` or `SKIPPED`. **Don't read the SonarCloud comment until all checks are green** — sonar lags and the comment may still reflect a stale commit otherwise.

If any check finishes with `FAILURE` or `CANCELLED`: investigate the failing job's logs (`gh run view <runId> --log-failed`). If the fix is mechanical (e.g. missing test update for a new constructor param), apply it, commit, push, and the cycle restarts naturally. If it's not obvious, surface to the user.

### 2d. Evaluate the automated review via `check-llm-review`

Fetch the bot's PR comment and run it through the **`check-llm-review`** skill's phases (don't re-implement). Classify items, gather evidence, decide what to fix vs. rebut.

**Severity discipline:**
- **🔴 (Must Fix):** required to fix or rebut before the loop can exit. These gate completion.
- **🟡 (Should Fix / Important):** address them — fix if the change is genuinely surgical and the issue is real, otherwise a one-line rebuttal in the PR comment is enough — but their existence does **not** gate completion. The PR can ship with open 🟡 items as long as each has been touched (fixed or briefly responded to).
- **🟢 (Consider / Nit):** ignore by default unless the user explicitly opts in.

For each 🔴 or 🟡 item being addressed:
- **Worth fixing per minimal-surgical-change principles** (this repo's CLAUDE.md: surgical changes, minimum sufficient, no speculative defenses): apply the fix, run tests if available, commit, push. Restart cycle.
- **False positive or out of scope**: rebut in a brief, evidence-led PR comment (file:line citations, grep results, code references) -- keep the tone factual and concise. Edit your existing rebuttal comment if you have one; otherwise create a new one.

For 🟡 specifically: the rebuttal can be terser than for 🔴 — a single clause stating the reason is fine (e.g. "🟡 N: pre-existing, out of scope for this PR" or "🟡 N: false positive — pattern matches DE's prod usage at <ref>"). Don't spend the same effort on a 🟡 you would on a 🔴.

### 2e. Run sonar-review if this is a Java repo

A Java repo has a SonarCloud check run on PRs. Detect via:
```bash
gh pr checks <number> --repo <repo> --json name | jq '.[] | select(.name | test("sonar"; "i"))'
```
Or check for `*.java` files in the diff. If no sonar check, skip this step.

If sonar check is present (and **only after all checks are green** so the comment is for the latest commit), invoke the `sonar-review` skill against this PR. It returns the issues list and overall report. Two pass conditions:

**(a) Coverage ≥ 80%** — read from the SonarCloud bot comment on the PR (look for the "Coverage" line) or fall back to `sonar-review`'s output if it surfaces coverage.

**(b) No unaddressed "new issues"** — every issue in the "New code" section of the sonar report needs an outcome:
- **Fixed** — code change applied, committed, pushed (restart cycle).
- **Addressed as false positive** — surfaced in a PR comment with a one-line rationale grounded in the code. Common false positives that almost always belong here without further investigation: `java:S107` ("too many parameters"), `java:S1192` ("define a constant" for one-shot string literals), `java:S138` ("function too long"), `java:S1448` ("too many methods in class"), cognitive-complexity warnings on generated/structural code. Other rules need actual investigation before being declared false positive.

If either pass condition fails: act on it (fix or comment), then restart cycle.

### 2f. Exit decision

Continue looping if any of these is still false. Exit only when **all** are true:

- ✅ `git merge origin/main` is clean (no conflicts).
- ✅ All check runs on the latest commit are `SUCCESS` or `SKIPPED`.
- ✅ Latest automated review (refreshed this cycle) has no 🔴 items, **or** every 🔴 item has been either fixed or rebutted in a PR comment. (🟡 items should also be touched — fix or brief response — but their existence is *not* a blocker for exit. 🟢 ignored.)
- ✅ If Java repo: SonarCloud coverage ≥ 80% **and** every new issue is fixed-or-addressed.

When exiting, post a brief final status message to the user with:
- PR URL
- Bullet list of what was fixed (commits with subjects)
- Bullet list of what was rebutted (with links to the rebuttal comments)
- The final state of each exit condition

## Important notes

- **Break the loop only when you genuinely need user input you can't derive from context.** The default is "figure it out and keep going." Examples of legitimate break points: merge conflict where both sides changed the same logic in incompatible ways and `git log` / surrounding code doesn't disambiguate; a 🔴 review item that asks for product/UX judgment; a sonar issue whose false-positive-vs-real status genuinely depends on intent you don't know. Examples of *not* legitimate break points: missing import to fix, test that needs the same trivial update the production code got, mechanical merge conflicts, sonar warnings on rules from the always-false-positive list.
- **Never force-push, never rebase main into a published branch, never skip hooks.** All commits go through normal CI. If a pre-commit hook fails, fix the underlying issue and create a new commit.
- **Do not approve or merge the PR.** This skill stops at "ready to merge" — the human decides whether to actually merge.
- **Severity defaults:** 🔴 must be fixed or rebutted (blocks exit). 🟡 should be touched — fixed when surgical, briefly rebutted otherwise — but doesn't block exit. 🟢 ignored unless the user opts in. Don't let 🟡 turn into a refactor; if the fix isn't a one-or-two-line change, rebut it as out of scope and move on.
- **Minimal-surgical-change principles apply to any fix you make.** Surgical changes only. No speculative defenses. No refactors of adjacent code. Every changed line should trace to a specific review item or check failure. If a fix would require touching code unrelated to the review, rebut instead and let it be a follow-up PR.
- **Cycle pacing.** Each iteration's main delay is waiting for checks to finish (~3-10 min on most repos). Use background `sleep N && gh pr checks` rather than polling tightly — the harness blocks long leading `sleep` commands when run inline, but a backgrounded sleep+check fires fine.
- **Comment hygiene.** Edit your own existing rebuttal comments rather than stacking new ones. Find your existing comment via:
  ```bash
  gh api repos/<repo>/issues/<number>/comments \
    --jq '.[] | select(.user.login == "<your-gh-login>") | .id'
  ```
  Update with `gh api repos/<repo>/issues/comments/<id> -X PATCH -f body=...` or `gh pr comment <number> --edit-last`.
