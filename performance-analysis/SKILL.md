---
name: performance-analysis
description: Instrument code with timing/memory logs, run it in a test environment, collect the logs, and produce a detailed performance analysis document. Use when the user asks to profile, perf-test, performance-analyse, or benchmark a backend component.
---

# Performance Analysis

A log-based profiling loop: instrument the code with timing/memory markers, trigger it, collect the logs, analyze the timeline, and revert. Works in any language and any environment where you can read logs back — the specifics of *where* logs land and *how* you trigger the code are gathered from the user up front.

## Phase 0 -- Gather prerequisites (BLOCKING)

You MUST know all of the following before doing any work. If any are missing, ask the user.

| Requirement | Why | Example |
|-------------|-----|---------|
| **Component(s) to analyse** | Determines what to instrument | A specific service method, request flow, or batch job |
| **Depth of tracing** | How many layers of calls to instrument | "Just the top-level method" vs "all the way down into the I/O client calls" |
| **How to find logs** | Where instrumentation output will appear and how to read it back | A log-aggregator query (e.g. SumoLogic/Datadog/CloudWatch/Loki), a local log file, a CI/job-runner run history, etc. |
| **How to trigger the code** | What invocation exercises the flow | A sample API/GraphQL/gRPC request, a CLI command, a job trigger |
| **Sample inputs** | Real inputs to drive the run | IDs, params, payloads -- ask the user for realistic values |
| **Environment** | Where it's safe to deploy instrumented code | A staging/preview environment, ideally not production |

### If the log path isn't obvious, resolve it upfront

Ask the user:
1. Where will the logs appear (which aggregator/file/run history)?
2. What log format and logging library is used?
3. Any special deploy steps to get instrumented code running?

Resolve these before proceeding -- the whole loop depends on being able to read the logs back.

---

## Phase 1 -- Snapshot current state

Record the HEAD commit of every repo you will touch, so you can cleanly revert:

```
Repos to instrument:
- <repo>: <commit SHA>
- (others if applicable)
```

Tell the user you have noted the commits and will revert at the end.

---

## Phase 2 -- Instrument with timing/memory logs

### What to log

At each tracing point, emit a log line with a unique, greppable prefix (`[PERF]`) containing:

| Field | Purpose |
|-------|---------|
| `[PERF]` prefix | Easy filter when searching logs |
| Step label | e.g. `buildQuery START`, `buildQuery END` |
| Elapsed ms | Delta from flow entry |
| Step ms | Time for this specific step |
| Heap/RSS MB | Used memory at this point |
| Discriminator fields | Whatever uniquely identifies this execution (request ID, input IDs, trace ID) |

### Java example

The technique is language-agnostic -- use your language's monotonic clock and memory API. A Java example:

```java
long flowStart = System.currentTimeMillis();
Runtime rt = Runtime.getRuntime();
long heapAtEntry = (rt.totalMemory() - rt.freeMemory()) / (1024 * 1024);
logger.warn("[PERF] MyMethod entry | heap={}MB | id={} traceId={}",
    heapAtEntry, id, traceId);

// ... do work ...
long stepEnd = System.currentTimeMillis();
long heapNow = (rt.totalMemory() - rt.freeMemory()) / (1024 * 1024);
logger.warn("[PERF] MyMethod someStep END | elapsed={}ms step={}ms heap={}MB | id={}",
    stepEnd - flowStart, stepEnd - stepStart, heapNow, id);
```

### Rules for instrumentation

1. **Log BOTH entry and exit** of every method/block in the traced depth.
2. **Include discriminator fields** in every line so you can correlate logs later.
3. **Place a START log before and an END log after** each significant block: validation, query/request building, I/O calls, aggregation, serialisation.
4. **Don't instrument trivially fast lines** (simple assignments, null checks) -- focus on I/O, computation, and allocations.
5. **Use a warn-level log** -- warnings surface quickly in most aggregators and are unlikely to be swallowed by verbosity filters, while not triggering alerts or failure states the way error-level would. (Adapt to your stack's conventions.)

---

## Phase 3 -- Provide a trigger and hand off to the user

After instrumenting, give the user:

1. A ready-to-paste **trigger** (HTTP/GraphQL/gRPC request, curl, CLI command, or job invocation) using the sample inputs they provided.
2. Clear instructions: "Deploy this to <environment> and run the trigger. Let me know when done."

### Trigger template

Produce a concrete, copy-pasteable invocation with the real inputs filled in. For example, a curl request:

```bash
curl -sS -X POST "$BASE_URL/<endpoint>" \
  -H "Content-Type: application/json" \
  -d '{ "<param>": "<REAL_VALUE>" }'
```

(Use whatever protocol the component actually exposes -- the point is that the user can run it without editing it.)

Then tell the user:
> I've finished instrumenting. Please deploy to <environment> and run the trigger above (ideally twice -- once cold, once warm). Let me know when done so I can pull the logs.

Then **stop and wait** for the user to confirm.

---

## Phase 4 -- Collect and analyse logs

### Collect

Pull the `[PERF]` lines from wherever Phase 0 established the logs land:
- **Log aggregator:** query for your `[PERF]` prefix, narrowed to the time window when the user ran the trigger.
- **Local file / run history:** read the file or ask the user to paste the relevant run's output.

### Analyse

1. **Parse the `[PERF]` lines** into a timeline: step label, elapsed ms, step ms, heap MB.
2. **Build a layered breakdown** table (see output template below).
3. **Compute percentages** of total time for each step.
4. **Compare runs** if multiple (cold vs warm, different inputs, etc.).
5. **Identify bottlenecks**: anything >20% of total time, or large heap deltas.

---

## Phase 5 -- Write the analysis document

Create (or overwrite, with user permission) a markdown document wherever the project keeps design/analysis docs.

**If a perf analysis already exists for this component**, ask: "There's an existing analysis at `<path>`. Overwrite it or create a new one?"

### Document template

```markdown
# <Component> -- Performance Analysis (<Month Day, Year>)

## Summary

<1-3 sentence overview with a comparison table>

| | Run 1 | Run 2 |
|--|--|--|
| **Total time** | **Xms** | **Yms** |
| **Heap delta** | **+NMB** | **+MMB** |

## Methodology and Test Params

- **Environment:** <environment> (<repo>)
- **Log source / query:** `<query or path used>`
- **Trace/request IDs:** ...
- **Input:** ...

## Run 1 -- Detailed Breakdown

Total: **Xms**, heap A -> B (+CMB)

(table with LAYER, STEP, TIME(ms), HEAP(MB), NOTES columns)

### Where did Xms go?

| Chunk | ms | % of total |
|-------|-----|-----------|
| ... | ... | ... |

## Key Findings and Bottlenecks

1. ...
2. ...

## Recommendations

| # | Fix | Expected impact | Effort |
|---|-----|-----------------|--------|
| 1 | ... | ... | ... |
```

---

## Phase 6 -- Revert instrumentation

Ask the user: "Want me to `git reset` to the commits I noted, or should I manually revert the log changes?"

- **If git reset:** give them the command: `cd <repo> && git reset --hard <commit>`
- **If manual revert:** remove every `[PERF]` log line and any helper variables (`flowStart`, `heapAtEntry`, etc.) you added. Verify with a diff.

**Do not leave instrumentation in the codebase.**
