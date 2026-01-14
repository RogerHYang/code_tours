# Concurrency and Rate Limiting Deep Dive

This document explains how promptfoo manages concurrent execution and rate limiting during evaluations. These mechanisms are essential for balancing speed (running many tests in parallel) against resource constraints (API rate limits, memory, CPU). This deep dive covers the architecture, design rationale, and nuances of promptfoo's approach.

## Table of Contents

1. [The Problem: Why Concurrency Matters](#the-problem-why-concurrency-matters)
2. [High-Level Architecture](#high-level-architecture)
3. [The Three Layers of Concurrency Management](#the-three-layers-of-concurrency-management)
   - [Layer 1: Evaluation-Level Concurrency](#layer-1-evaluation-level-concurrency)
   - [Layer 2: Assertion-Level Concurrency](#layer-2-assertion-level-concurrency)
   - [Layer 3: Database-Level Concurrency (WAL Mode)](#layer-3-database-level-concurrency-wal-mode)
4. [Serial vs. Concurrent Execution](#serial-vs-concurrent-execution)
5. [Rate Limiting and Delays](#rate-limiting-and-delays)
6. [Timeout Mechanisms](#timeout-mechanisms)
7. [Cancellation and Abort Signals](#cancellation-and-abort-signals)
8. [Caching to Avoid Redundant Work](#caching-to-avoid-redundant-work)
9. [Design Trade-offs and Alternatives](#design-trade-offs-and-alternatives)
10. [Configuration Reference](#configuration-reference)

## The Problem: Why Concurrency Matters

### The Fundamental Tension

When running evaluations, you face a classic trade-off:

```
Speed ←────────────────────────────────→ Resource Constraints
 │                                              │
 │ Run all 1000 tests at once                   │ API rate limits (e.g., 60 req/min)
 │ Finish in seconds                            │ Memory exhaustion
 │ Maximum throughput                           │ Provider throttling/bans
 │                                              │ Cost spikes from parallel calls
 └──────────────────────────────────────────────┘
```

**Without concurrency limits**, you'd either:
1. Run tests sequentially (slow: 1000 tests × 2 seconds = 33+ minutes)
2. Run all tests at once (dangerous: API bans, OOM, unpredictable costs)

**With concurrency management**, you get the best of both worlds:
- Parallel execution for speed
- Bounded parallelism to respect constraints
- Graceful degradation when limits are hit

### Real-World Scenarios

| Scenario | Challenge | Solution |
|----------|-----------|----------|
| OpenAI API with rate limits | 60 requests/minute | `maxConcurrency: 1` + `delay: 1000` |
| Local model (Ollama) | Single GPU, sequential inference | `maxConcurrency: 1` |
| Expensive API (GPT-4 Turbo) | Cost control | `maxConcurrency: 2` + caching |
| Multi-turn conversations | Order matters | `runSerially: true` per test |
| Fast embeddings API | No rate limits | `maxConcurrency: 20` |

## High-Level Architecture

Promptfoo's concurrency management operates at three distinct layers:

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          EVALUATION LAYER                               │
│                                                                         │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │  async.forEachOfLimit(testCases, concurrency, ...)              │   │
│   │                                                                 │   │
│   │  Controls: How many test cases run simultaneously               │   │
│   │  Default: 4 concurrent tests                                    │   │
│   │  Config: maxConcurrency, -j flag, PROMPTFOO_MAX_CONCURRENCY     │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                              │                                          │
│                              ▼                                          │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │  Per Test Case                                                  │   │
│   │  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐       │   │
│   │  │ Provider     │ →  │ Assertions   │ →  │ Persist      │       │   │
│   │  │ API Call     │    │ (grading)    │    │ Results      │       │   │
│   │  └──────────────┘    └──────────────┘    └──────────────┘       │   │
│   │        │                    │                    │              │   │
│   │        ▼                    ▼                    ▼              │   │
│   │   delay/sleep         max 3 parallel       WAL mode             │   │
│   │   rate limiting       assertions           concurrent writes    │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

**Key Insight**: Each layer has its own concurrency limit because the bottlenecks are different. The provider API might allow 4 concurrent requests, but within each request, you might have 5 assertions that each call an LLM (for model-graded checks). Running all 5 at once would be 20 LLM calls—potentially problematic.

## The Three Layers of Concurrency Management

### Layer 1: Evaluation-Level Concurrency

This is the primary concurrency limit: how many test cases run simultaneously.

#### The Default: 4 Concurrent Tests

```typescript
// src/constants.ts
export const DEFAULT_MAX_CONCURRENCY = 4;
```

**Why 4?** This is a pragmatic default:
- Low enough to avoid most API rate limits
- High enough to provide meaningful speedup
- Works well with typical local LLM setups (1 inference + 3 queued)

[View in GitHub](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/constants.ts#L8)

#### How It Works: The async.forEachOfLimit Pattern

Promptfoo uses the `async` library's `forEachOfLimit` function, which is a battle-tested way to run async operations with bounded parallelism:

```typescript
// Simplified from src/evaluator.ts
await async.forEachOfLimit(
  concurrentRunEvalOptions,  // Array of test cases to run
  concurrency,                // Maximum parallel executions (e.g., 4)
  async (evalStep) => {
    checkAbort();
    await processEvalStepWithTimeout(evalStep, idx);
  }
);
```

**How forEachOfLimit works**:

```
Time ──────────────────────────────────────────────────────────────────▶

concurrency = 3, tests = [A, B, C, D, E, F]

Slot 1: │ A (2s) │        │ D (1s) │        │ F (1s) │
Slot 2: │ B (1s) ││ C (3s)          ││ E (2s)        │
Slot 3: (initially empty, fills as slots complete)

When A finishes → D starts
When B finishes → C starts
When D finishes → E starts
When C finishes → F starts
```

[View in GitHub](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/evaluator.ts#L1677)

#### Why Not Use Promise.all with Chunking?

An alternative approach would be:

```typescript
// Chunking approach (NOT what promptfoo does)
const chunks = chunkArray(tests, concurrency);
for (const chunk of chunks) {
  await Promise.all(chunk.map(test => runTest(test)));
  // Wait for ALL in chunk to finish before starting next chunk
}
```

**Problem with chunking**: If you have a mix of fast and slow tests, you waste time waiting:

```
Chunking (concurrency=3):
Chunk 1: │ A (1s) │ B (5s)          │ C (1s) │
                  └── idle 4s ──────┘        └── idle 4s ──┘
         
         Chunk 2 can't start until B finishes!

forEachOfLimit:
Slot 1: │ A (1s) ││ D ││ E ││ F │
Slot 2: │ B (5s)              │
Slot 3: │ C (1s) ││ ... │
         
         D starts immediately when A finishes!
```

**forEachOfLimit** keeps all slots busy, maximizing throughput.

### Layer 2: Assertion-Level Concurrency

When a test case has multiple assertions (grading criteria), they're also run in parallel—but with a separate, lower limit.

```typescript
// src/assertions/index.ts
const ASSERTIONS_MAX_CONCURRENCY = getEnvInt('PROMPTFOO_ASSERTIONS_MAX_CONCURRENCY', 3);

await async.forEachOfLimit(
  asserts,
  ASSERTIONS_MAX_CONCURRENCY,
  async ({ assertion, assertResult, index }) => {
    const result = await runAssertion(...);
    assertResult.addResult({ index, result, ... });
  }
);
```

[View in GitHub](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/assertions/index.ts#L102)

**Why a separate limit for assertions?**

Consider this config:

```yaml
tests:
  - vars:
      question: "What is 2+2?"
    assert:
      - type: llm-rubric      # Calls GPT-4 to grade
        value: "Answer is correct"
      - type: llm-rubric      # Another LLM call
        value: "Answer is concise"
      - type: factuality      # Another LLM call
        value: "4"
      - type: contains        # Fast, no LLM call
        value: "4"
```

With 4 test cases running at `maxConcurrency: 4`, and each having 3 model-graded assertions, you'd have:
- 4 tests × 3 LLM assertions = 12 simultaneous grading LLM calls

By limiting assertion concurrency to 3, you cap it at:
- 4 tests × 3 assertions per test = 12 assertions, but only 3 LLM calls per test at once = effectively 12 (worst case)

The separate limit prevents **assertion amplification** from overwhelming the grading provider.

### Layer 3: Database-Level Concurrency (WAL Mode)

As results come in, they need to be persisted. SQLite by default uses a single-writer model, which would serialize all writes. Promptfoo enables WAL (Write-Ahead Logging) mode:

```typescript
// src/database/index.ts
if (!isMemoryDb && !getEnvBool('PROMPTFOO_DISABLE_WAL_MODE', false)) {
  // Enable WAL mode for better concurrency
  sqliteInstance.pragma('journal_mode = WAL');
  sqliteInstance.pragma('wal_autocheckpoint = 1000');
  sqliteInstance.pragma('synchronous = NORMAL');
}
```

[View in GitHub](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/database/index.ts#L39-L62)

**How WAL enables concurrent writes**:

```
Without WAL (Rollback Journal):
┌─────────────────────────────────────────────────────────┐
│                                                         │
│  Writer 1 ────────────│ LOCK │───────────────▶          │
│  Writer 2             ▼      │ BLOCKED │────────▶       │
│  Writer 3                    │ BLOCKED │────────────▶   │
│  Reader   ──── BLOCKED ──────────────────────────────   │
│                                                         │
└─────────────────────────────────────────────────────────┘

With WAL Mode:
┌─────────────────────────────────────────────────────────┐
│                                                         │
│  Writer 1 ────────────────────────────────▶             │
│  Writer 2 ─────────────────────────────▶   (can queue)  │
│  Reader 1 ──────────────────────────────────────▶       │
│  Reader 2 ────────────────────────────────────▶         │
│                                                         │
│  Writes go to WAL file, reads see committed snapshot    │
└─────────────────────────────────────────────────────────┘
```

**Trade-off**: WAL mode increases disk usage (extra `-wal` and `-shm` files) but dramatically improves concurrent access. This is why promptfoo checkpoints the WAL on database close:

```typescript
// Before closing, merge WAL back into main database
sqliteInstance.pragma('wal_checkpoint(TRUNCATE)');
```

## Serial vs. Concurrent Execution

Not all tests can run concurrently. Promptfoo provides escape hatches for tests that require sequential execution.

### Automatic Serial Detection

The evaluator automatically forces serial execution when it detects dependencies:

```typescript
// src/evaluator.ts (simplified)
if (concurrency > 1) {
  const usesConversation = prompts.some((p) => p.raw.includes('_conversation'));
  const usesStoreOutputAs = tests.some((t) => t.options?.storeOutputAs);
  
  if (usesConversation) {
    logger.info('Setting concurrency to 1 because _conversation variable is used.');
    concurrency = 1;
  } else if (usesStoreOutputAs) {
    logger.info('Setting concurrency to 1 because storeOutputAs is used.');
    concurrency = 1;
  }
}
```

[View in GitHub](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/evaluator.ts#L1298-L1310)

**Why these force serial?**

1. **`_conversation`**: Multi-turn conversations need message 1's response before message 2 can be sent. Parallel execution would corrupt the conversation state.

2. **`storeOutputAs`**: Registers allow passing values between tests. If Test B depends on Test A's output, they must run in order.

### Explicit Serial Flag: runSerially

For cases the auto-detection misses, you can force individual tests to run serially:

```yaml
tests:
  - vars:
      input: "First message"
    options:
      runSerially: true
  - vars:
      input: "Follow-up that depends on first"
    options:
      runSerially: true
```

The evaluator separates serial and concurrent tests:

```typescript
// src/evaluator.ts (simplified)
const serialRunEvalOptions: RunEvalOptions[] = [];
const concurrentRunEvalOptions: RunEvalOptions[] = [];

for (const evalOption of runEvalOptions) {
  if (evalOption.test.options?.runSerially) {
    serialRunEvalOptions.push(evalOption);
  } else {
    concurrentRunEvalOptions.push(evalOption);
  }
}

// Run serial tests first, one at a time
for (const evalStep of serialRunEvalOptions) {
  await processEvalStepWithTimeout(evalStep, idx);
}

// Then run concurrent tests in parallel
await async.forEachOfLimit(concurrentRunEvalOptions, concurrency, async (evalStep) => {
  await processEvalStepWithTimeout(evalStep, idx);
});
```

[View in GitHub](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/evaluator.ts#L1628-L1683)

**Why serial tests run first?** This ensures any dependencies established by serial tests are available for concurrent tests that might reference them.

## Rate Limiting and Delays

Concurrency limits prevent *too many simultaneous requests*, but rate limits are often expressed as *requests per time unit* (e.g., 60/minute). Promptfoo addresses this with delays.

### The Delay Mechanism

```typescript
// src/evaluator.ts
provider.delay ??= delay ?? getEnvInt('PROMPTFOO_DELAY_MS', 0);

// After getting the response...
if (!response.cached && provider.delay > 0) {
  logger.debug(`Sleeping for ${provider.delay}ms`);
  await sleep(provider.delay);
}
```

[View in GitHub](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/evaluator.ts#L313)

**Key insight**: The delay happens **after** each API call, not before. This creates spacing between requests without adding unnecessary latency to the first call.

```
Without delay (burst):
│ Call ││ Call ││ Call ││ Call │  → 4 calls in 100ms → RATE LIMITED!

With delay: 1000ms
│ Call │──1s──│ Call │──1s──│ Call │──1s──│ Call │  → 4 calls in 4s → OK
```

### Delay Inheritance

Delays can be set at multiple levels, with the most specific taking precedence:

```yaml
# promptfoo.yaml
evaluateOptions:
  delay: 500  # Default for all tests

providers:
  - id: openai:gpt-4
    delay: 2000  # Override for expensive model
  - id: openai:gpt-3.5-turbo
    delay: 100   # Cheaper model, less delay needed
```

Or via environment variable for quick overrides:
```bash
PROMPTFOO_DELAY_MS=1000 promptfoo eval
```

### Cache Bypass for Delays

Crucially, delays are skipped for cached responses:

```typescript
if (!response.cached && provider.delay > 0) {
  await sleep(provider.delay);
} else if (response.cached) {
  logger.debug('Skipping delay because response is cached');
}
```

This optimization means re-running an evaluation (with cache hits) is much faster than the initial run.

## Timeout Mechanisms

Promptfoo provides two timeout levels to prevent runaway evaluations.

### Per-Test Timeout

Each individual test case can have a timeout:

```typescript
// src/evaluator.ts
const processEvalStepWithTimeout = async (evalStep, index) => {
  const timeoutMs = options.timeoutMs || getEvalTimeoutMs();  // From env: PROMPTFOO_EVAL_TIMEOUT_MS
  
  if (timeoutMs <= 0) {
    return processEvalStep(evalStep, index);  // No timeout
  }
  
  const abortController = new AbortController();
  
  return await Promise.race([
    processEvalStep({...evalStep, abortSignal: abortController.signal}, index),
    new Promise((_, reject) => {
      setTimeout(() => {
        abortController.abort();
        reject(new Error(`Evaluation timed out after ${timeoutMs}ms`));
      }, timeoutMs);
    }),
  ]);
};
```

[View in GitHub](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/evaluator.ts#L1464-L1510)

**On timeout**, the test is:
1. Aborted via AbortController (propagated to the provider)
2. Provider's `cleanup()` method is called (if present)
3. An error result is recorded with the timeout message
4. Execution continues with remaining tests

### Global Evaluation Timeout

For the entire evaluation run:

```typescript
// src/evaluator.ts
const maxEvalTimeMs = options.maxEvalTimeMs ?? getMaxEvalTimeMs();  // From env: PROMPTFOO_MAX_EVAL_TIME_MS

if (maxEvalTimeMs > 0) {
  globalAbortController = new AbortController();
  options.abortSignal = AbortSignal.any([options.abortSignal, globalAbortController.signal]);
  
  globalTimeout = setTimeout(() => {
    evalTimedOut = true;
    globalAbortController.abort();
  }, maxEvalTimeMs);
}
```

[View in GitHub](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/evaluator.ts#L855-L874)

**Global timeout use cases**:
- CI/CD pipelines with hard time limits
- Interactive demos where you want partial results
- Cost control (stop after N minutes regardless)

## Cancellation and Abort Signals

Promptfoo uses the standard Web `AbortController`/`AbortSignal` API for cancellation.

### Signal Propagation

```
User presses Ctrl+C
       │
       ▼
┌──────────────────┐
│ Global AbortCtrl │ ───▶ options.abortSignal
└──────────────────┘
          │
          ▼
    ┌───────────────────────────────────────┐
    │          forEachOfLimit loop          │
    │                                       │
    │  checkAbort() ──▶ throws if aborted   │
    │                                       │
    │       ┌─────────────────────┐         │
    │       │ processEvalStep     │         │
    │       │                     │         │
    │       │   ┌─────────────┐   │         │
    │       │   │ provider    │   │         │
    │       │   │ .callApi()  │   │         │
    │       │   │    │        │   │         │
    │       │   │    ▼        │   │         │
    │       │   │ Uses signal │   │         │
    │       │   │ for fetch   │   │         │
    │       │   └─────────────┘   │         │
    │       └─────────────────────┘         │
    └───────────────────────────────────────┘
```

### How Providers Use the Signal

```typescript
// In provider implementation
response = await activeProvider.callApi(
  renderedPrompt,
  callApiContext,
  abortSignal ? { abortSignal } : undefined,
);
```

The signal flows all the way to the HTTP fetch:

```typescript
// In fetch wrapper
const signal = options.signal
  ? AbortSignal.any([options.signal, timeoutController.signal])
  : timeoutController.signal;

const response = await fetch(url, { ...options, signal });
```

[View in GitHub](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/evaluator.ts#L421-L425)

### Graceful Degradation on Abort

When an abort occurs, promptfoo doesn't crash—it saves partial progress:

```typescript
} catch (err) {
  if (options.abortSignal?.aborted) {
    if (evalTimedOut) {
      logger.warn(`Evaluation stopped after reaching max duration (${maxEvalTimeMs}ms)`);
    } else {
      logger.info('Evaluation interrupted, saving progress...');
    }
    // Continue to save whatever results we have
  } else {
    throw err;  // Re-throw non-abort errors
  }
}
```

This means `Ctrl+C` during a long evaluation gives you the results computed so far, not a crash with no output.

## Caching to Avoid Redundant Work

Caching is a concurrency optimization: by avoiding redundant work, you effectively reduce the number of concurrent calls needed.

### Cache Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                       Cache Manager                             │
│                                                                 │
│   Disk Cache (default)         Memory Cache (testing)           │
│   ~/.promptfoo/cache/          In-process only                  │
│   cache.json                                                    │
│                                                                 │
│   TTL: 14 days (default)                                        │
│   Key: hash(url + options)                                      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### cache.wrap for Deduplication

The cache uses `wrap` for concurrent-safe access:

```typescript
// src/cache.ts
const cachedResponse = await cache.wrap(cacheKey, async () => {
  // This function is ONLY called if cache miss
  // And it's called ONLY ONCE even if multiple concurrent requests
  cached = false;
  const response = await fetchWithRetries(url, options, timeout);
  return response;
});
```

[View in GitHub](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/cache.ts#L165)

**Why wrap matters for concurrency**:

```
Without wrap:
Request 1 ──▶ cache.get() → miss ──▶ fetch() ──▶ cache.set()
Request 2 ──▶ cache.get() → miss ──▶ fetch() ──▶ cache.set()  (duplicate!)
Request 3 ──▶ cache.get() → miss ──▶ fetch() ──▶ cache.set()  (duplicate!)

With wrap:
Request 1 ──▶ wrap() → miss, fetching...
Request 2 ──▶ wrap() → locked, waiting...
Request 3 ──▶ wrap() → locked, waiting...
         ... fetch completes ...
All 3 requests get the same cached result, only 1 API call made!
```

### Cache Busting for Repeats

When running evaluations multiple times (via `repeat` option), you typically want fresh results. The evaluator handles this:

```typescript
if (repeatIndex > 0) {
  callApiContext.bustCache = true;
}
```

This ensures repeat runs hit the API, not the cache, giving you statistical variance for reliability testing.

## Design Trade-offs and Alternatives

### Why async.forEachOfLimit Over Other Approaches?

**Considered Alternatives**:

1. **Worker Threads**: Node.js worker threads for true parallelism
   - **Rejected**: Overhead of thread creation/communication outweighs benefits for I/O-bound LLM calls
   - **When it would be better**: CPU-bound local model inference

2. **Promise.allSettled with Batching**: Process in fixed-size batches
   - **Rejected**: Wasted slots while waiting for slow tests (see earlier diagram)
   - **When it would be better**: When you need "all or nothing" batch semantics

3. **No Limit (Promise.all on everything)**: Maximum parallelism
   - **Rejected**: Unpredictable resource usage, API bans
   - **When it would be better**: Never for production; maybe for testing with mocks

4. **p-limit or similar**: Another popular limiting library
   - **Why async library chosen**: Already widely used in the codebase, well-tested, familiar API

### Why Two Concurrency Limits (Eval + Assertions)?

**Alternative**: Single global limit across all operations

**Problem with single limit**:
- Setting it high enough for eval throughput = too many assertion LLM calls
- Setting it low for assertions = slow eval execution

**Current approach**: Independent limits let you tune each bottleneck separately.

### Why File-Based Cache, Not SQLite?

The result data goes to SQLite, but the API response cache uses a JSON file.

**Rationale**:
- Cache data is ephemeral (can be deleted without data loss)
- JSON file is human-inspectable for debugging
- No schema migrations needed for cache format changes
- Lower complexity than managing a second SQLite connection

## Configuration Reference

### Environment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `PROMPTFOO_MAX_CONCURRENCY` | 4 | Maximum concurrent test cases |
| `PROMPTFOO_ASSERTIONS_MAX_CONCURRENCY` | 3 | Maximum concurrent assertions per test |
| `PROMPTFOO_DELAY_MS` | 0 | Milliseconds to wait after each API call |
| `PROMPTFOO_EVAL_TIMEOUT_MS` | 0 | Per-test timeout (0 = no timeout) |
| `PROMPTFOO_MAX_EVAL_TIME_MS` | 0 | Total evaluation timeout (0 = no limit) |
| `PROMPTFOO_CACHE_ENABLED` | true | Enable/disable API response caching |
| `PROMPTFOO_CACHE_TTL` | 1209600 | Cache TTL in seconds (14 days) |
| `PROMPTFOO_DISABLE_WAL_MODE` | false | Disable SQLite WAL mode |

### Config File Options

```yaml
# promptfooconfig.yaml

# CLI equivalent: -j, --max-concurrency
evaluateOptions:
  maxConcurrency: 4
  delay: 500  # ms between API calls

providers:
  - id: openai:gpt-4
    delay: 2000  # Provider-specific delay

tests:
  - vars:
      input: "Test input"
    options:
      runSerially: true  # Force this test to run alone
```

### CLI Flags

```bash
# Set concurrency to 2
promptfoo eval -j 2

# Set concurrency to 1 (fully serial)
promptfoo eval -j 1

# Combine with delay
promptfoo eval -j 2 --delay 1000
```

## Summary

Promptfoo's concurrency management is a layered system designed to maximize throughput while respecting resource constraints:

1. **Evaluation Layer**: `forEachOfLimit` with configurable concurrency (default 4)
2. **Assertion Layer**: Separate limit (default 3) to prevent grader overload
3. **Database Layer**: WAL mode for concurrent writes without blocking
4. **Rate Limiting**: Post-call delays, skipped for cache hits
5. **Timeouts**: Per-test and global, with graceful abort handling
6. **Caching**: Deduplication via `wrap`, 14-day TTL

The system is designed to be tunable—you can adjust each knob independently based on your specific provider constraints, cost sensitivity, and throughput requirements.
