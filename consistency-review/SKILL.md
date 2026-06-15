---
name: consistency-review
description: Deep analysis of a branch's diff against the rest of the codebase to find deviations in conventions, libraries, naming, patterns, algorithms, and style. Use when the user asks to review code consistency, check conventions, or audit a branch for codebase conformance.
---

# Codebase Consistency Review

## Phase 0 -- Prerequisites (BLOCKING)

You MUST know all of the following before doing any work. If any are missing, ask the user.

| Requirement | Why | Example |
|-------------|-----|---------|
| **Repo path** | Which repository to analyse | `~/projects/my-service` |
| **Working branch** | The branch with changes to review | `feature/TICKET-123-new-feature` |
| **Target branch** | The base branch to diff against (the "established" codebase) | `main` |

Optional but useful:
- **Focus areas** -- specific concerns the user wants checked (e.g. "mainly worried about error handling" or "check the test patterns"). If not provided, do a full-spectrum review.
- **AGENTS.md / style guide** -- check if the repo has an `AGENTS.md`, `.editorconfig`, linter configs, or style guides. Read them first -- they are the ground truth for conventions.

---

## Phase 1 -- Gather the diff and changed files

```bash
cd <repo_path>
git fetch origin
git diff origin/<target_branch>...origin/<working_branch> --stat          # file overview
git diff origin/<target_branch>...origin/<working_branch> --name-only     # just file paths
git log origin/<target_branch>..origin/<working_branch> --oneline         # commit context
git diff origin/<target_branch>...origin/<working_branch>                 # full diff
```

Capture:
1. **List of changed files** with their language/type (Java, TypeScript, Python, config, etc.)
2. **The full diff** -- you will cross-reference every piece of it against established patterns
3. **Commit messages** -- sometimes reveal intent that code doesn't

---

## Phase 2 -- Deep exploration of existing codebase conventions

This is the most important phase. You need to build a mental model of how this codebase does things BEFORE judging the diff. Spend time here.

For EACH language/area touched by the diff, explore the existing codebase to establish the conventions. Use the patterns below as a checklist -- not all will apply to every repo.

### 2a. Naming conventions

| Check | How to explore |
|-------|---------------|
| Variable naming (camelCase, snake_case, UPPER_CASE) | Read 3-5 existing files in the same package/directory as the changed files |
| Method naming patterns | Check method names in the same class or sibling classes |
| Class/file naming | Look at the directory listing and naming pattern |
| Constants naming | Grep for `static final` / `const` / `UPPER_CASE` patterns |
| Test naming | Check existing test files: `should_X_when_Y`, `testX`, `XTest`, etc. |
| Package/module naming | ls the directory structure |

### 2b. Library and API usage

| Check | How to explore |
|-------|---------------|
| Which libraries are used for common tasks | Check `build.gradle` / `package.json` / `requirements.txt` for dependencies |
| HTTP client choice | Grep for HTTP call patterns -- does the repo use RestTemplate, WebClient, OkHttp, fetch, axios? |
| Logging library/patterns | Grep for `logger.` / `log.` / `console.` to see the established pattern |
| Testing framework | Check test files -- JUnit 5, TestNG, Jest, pytest? What assertion style? |
| Serialization | Jackson annotations, Gson, custom serializers? |
| Collection utilities | Guava, Apache Commons, Streams, Lodash? |

### 2c. Structural patterns

| Check | How to explore |
|-------|---------------|
| Error handling | How do existing methods handle errors? Checked exceptions, unchecked, Result types, try-catch granularity |
| Null handling | Optional, @Nullable, null checks, Kotlin nullability? |
| Method size/complexity | Typical method length in the package |
| Class organization | Field ordering, constructor patterns, method grouping (public first? alphabetical?) |
| Import ordering/style | Wildcard imports vs explicit? Grouping? |
| Comment style | Javadoc on public methods? Inline comments? None? |

### 2d. Architectural patterns

| Check | How to explore |
|-------|---------------|
| Layering | Does the codebase follow controller -> service -> repository? Manager -> client? |
| Dependency injection | Constructor injection, field injection, provider pattern? |
| Configuration | How are configs loaded? Environment variables, config files, Spring profiles? |
| DTO/model patterns | Records, data classes, builders, immutable objects? |
| Validation | Where does validation happen? Bean validation annotations, manual checks, dedicated validator classes? |

### 2e. Test patterns

| Check | How to explore |
|-------|---------------|
| Test structure | Arrange-Act-Assert? Given-When-Then? |
| Mocking approach | Mockito, mock constructors, test doubles, fakes? |
| Test data setup | Builders, fixtures, factory methods, inline construction? |
| Assertion style | AssertJ, Hamcrest, JUnit assertions, assertEquals vs assertThat? |
| Test granularity | Unit tests, integration tests, or both? What's the boundary? |

### How to explore efficiently

1. **Start with AGENTS.md** in the repo root -- it may explicitly state conventions.
2. **Read 3-5 sibling files** in the same package as each changed file. Siblings are the strongest signal for local conventions.
3. **Grep for patterns** across the repo when you need to confirm a convention is consistent (e.g., `grep -r "logger\." --include="*.java" | head -20` to check logging patterns).
4. **Check config files** -- `.editorconfig`, `checkstyle.xml`, `eslint.json`, `spotless` config -- these encode enforced conventions.
5. **Read test files** in the same test package to understand testing conventions.

**Document your findings as you go.** You'll reference them in Phase 3.

---

## Phase 3 -- Cross-reference the diff against conventions

Go through the diff methodically. For EACH changed file:

1. **Read the full diff** for that file.
2. **Compare every aspect** against the conventions established in Phase 2.
3. **Categorize findings** by severity:

| Severity | Meaning | Example |
|----------|---------|---------|
| **Inconsistency** | Clearly deviates from established codebase pattern | Using snake_case where codebase uses camelCase |
| **Concern** | Potentially deviates or could be improved for consistency | Using a different library than what the rest of the codebase uses for the same task |
| **Suggestion** | Minor style or pattern preference | A method is longer than typical for the codebase, but not egregiously so |
| **Good** | Follows conventions well or improves upon them | Worth calling out to reinforce good patterns |

### What specifically to check in the diff

- [ ] Variable/method/class names follow established conventions
- [ ] Imported libraries match what the codebase uses for similar tasks (no redundant deps)
- [ ] Error handling follows the codebase pattern
- [ ] Logging uses the same logger, level conventions, and message format
- [ ] Test structure matches existing test patterns
- [ ] Null handling is consistent
- [ ] Method size is reasonable for the codebase
- [ ] Import style matches (wildcard vs explicit, ordering)
- [ ] Comment style matches
- [ ] Configuration access follows the established pattern
- [ ] New classes follow the naming and organizational patterns of sibling classes
- [ ] Algorithms/data structures are appropriate (e.g., not using a HashMap where the codebase consistently uses LinkedHashMap for ordering)
- [ ] No reinvention of utilities that already exist in the codebase

---

## Phase 4 -- Write the report

Present findings to the user in this format:

```markdown
# Consistency Review: <branch> -> <target>

## Summary

<1-3 sentences: overall assessment. Is the code generally consistent? Are there systemic issues or just minor nits?>

**Files reviewed:** N
**Findings:** X inconsistencies, Y concerns, Z suggestions

## Established Conventions (reference)

<Brief summary of the key conventions you discovered in Phase 2, so the user understands what you're comparing against. Keep this short -- bullet points, not paragraphs.>

## Findings

### Inconsistencies

<For each inconsistency:>
#### [File:line] Brief title
- **What:** Description of the deviation
- **Codebase pattern:** What the rest of the codebase does (with examples/file references)
- **In the diff:** What the diff does differently
- **Suggestion:** How to align

### Concerns

<Same format as above, but for medium-severity items>

### Suggestions

<Same format, briefer -- these are minor>

### Good Patterns

<Optional: call out 1-3 things the diff does well for positive reinforcement>

## Checklist

- [ ] Variable naming consistent
- [ ] Library usage consistent
- [ ] Error handling consistent
- [ ] Logging consistent
- [ ] Test patterns consistent
- [ ] Null handling consistent
- [ ] Import style consistent
- [ ] No reinvented utilities
<Mark each as [x] if passing, [ ] if not, with a brief note for failures>
```

### Report guidelines

- **Be specific.** Always reference actual file paths, line numbers, and concrete examples from the existing codebase.
- **Don't nitpick enforced rules.** If a linter/formatter already handles it (e.g., Spotless, ESLint, Checkstyle), skip it -- it'll get caught automatically.
- **Distinguish convention from preference.** If the codebase is itself inconsistent on something, note it but don't flag the diff for it.
- **Prioritize.** Lead with the most impactful inconsistencies. If there are only suggestions and good patterns, say so upfront -- don't bury the lede.
- **Respect repo-specific AGENTS.md.** If the repo's AGENTS.md explicitly endorses a pattern that the diff follows, don't flag it as inconsistent even if other parts of the codebase haven't caught up yet.
