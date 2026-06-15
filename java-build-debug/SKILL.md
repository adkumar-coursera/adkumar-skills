---
name: java-build-debug
description: Run and diagnose Gradle build failures in a Java project. Reads the build log bottom-up, identifies the failed task, and finds the root-cause error (including annotation-processor cascade failures). Use when the user asks to build, test, or debug a failing Gradle/Java build, or to investigate a build.log.
---

# Java/Gradle Build — Run & Diagnose

Diagnose (and optionally run) a Gradle build for a Java project. The diagnosis flow applies to any Gradle + Java setup; a few sections call out gotchas common to particular stacks (annotation processors like Immutables, GraphQL/proto codegen, Spotless/Checkstyle).

## Step 0 — Run the build (optional)

If the user wants you to run it (not just diagnose an existing log), redirect all output to a log file and read that — Gradle's console output is noisy and ANSI-laden.

| Goal | Command |
|------|---------|
| Full build + format | `./gradlew spotlessApply build --console=plain > build.log 2>&1` |
| Build, skip tests/checks | `./gradlew build -x check --console=plain > build.log 2>&1` |
| Tests only | `./gradlew test --console=plain > build.log 2>&1` |
| Format only | `./gradlew spotlessApply --console=plain > build.log 2>&1` |

Run from the repo root. `--console=plain` strips progress bars that clutter the log. Avoid `--info` unless debugging Gradle itself — it roughly 10x's the output and makes grepping harder.

Then check the result:
```bash
grep -E "BUILD (SUCCESSFUL|FAILED)" build.log
```
`SUCCESSFUL` → report success. `FAILED` → diagnose below.

## Step 1 — Read the log bottom-up

The failure summary is at the end. Read the last ~80 lines first and look for:
- `BUILD FAILED` / `BUILD SUCCESSFUL`
- `Task :someTask FAILED` — which Gradle task failed
- `FAILURE: Build failed with an exception` — Gradle's summary block

## Step 2 — Identify the failed task

Common failure points:

| Task | Typical cause |
|------|--------------|
| `:compileJava` | Java compilation errors — the most common |
| `:compileTestJava` | Test compilation errors |
| `:test` | Unit test failures |
| `:checkstyleMain` / `:checkstyleTest` | Style violations |
| `:spotlessJava` / `:spotlessJavaCheck` | Formatting violations (usually fixed by re-running `spotlessApply`) |
| `:generateJava` | Codegen failure (e.g. a GraphQL/DGS schema issue) |
| `:generateProto` | Protobuf codegen failure |
| `:jacocoTestCoverageVerification` | Code coverage below threshold |

Task names vary by project — adapt to whatever your build defines.

## Step 3 — For `:compileJava` failures (most common)

Search the log for `error:` lines — these are the real errors. Key patterns:
- **Duplicate method/constructor**: same signature appears twice. Remove the duplicate.
- **cannot find symbol (generic)**: missing import, deleted class, or typo.
- **incompatible types / method not found**: signature mismatch after a refactor.

### Annotation-processor cascade (important)

If the project uses an annotation processor that generates classes — **Immutables** (`Immutable*`), MapStruct, Dagger, Lombok, etc. — a single real error stops codegen, which then produces dozens of *secondary* `cannot find symbol` errors for the generated classes that never got created.

Don't chase those. **Find the first non-generated error in the log — it appears earliest — that's the root cause.** One real bug can spawn 50+ `cannot find symbol Immutable*` errors; fix the root and they all vanish.

## Step 4 — For `:test` failures

Search for `FAILED` in the test output to find the failing class + method, then read the assertion message.

Gradle writes detailed HTML reports under `build/reports/`:
- `tests/test/index.html` — pass/fail per class and method
- `jacoco/test/html/index.html` — coverage
- `checkstyle/main.html` / `checkstyle/test.html` — checkstyle violations

If the log's stack trace is truncated, open these reports for the full picture.

## Step 5 — Report

Summarize:
1. Which task failed
2. The root-cause error (not the cascade errors)
3. The file and line number
4. A fix, or a clear explanation of what's wrong

## Notes

- If the log is too large to read fully, `grep` for specific patterns rather than reading the whole thing.
- Spotless/formatting failures usually clear by re-running `spotlessApply` (the first run reformats, the second confirms).
