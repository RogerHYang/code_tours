# Promptfoo Result Model Deep Dive

## Table of Contents

1. [The Central Question: What Data Must an Evaluation Produce?](#the-central-question-what-data-must-an-evaluation-produce)
   - [The Multiple Audiences for Results](#the-multiple-audiences-for-results)
   - [The Hierarchy of Information](#the-hierarchy-of-information)
2. [Architecture Overview](#architecture-overview)
   - [The Result Hierarchy](#the-result-hierarchy)
   - [Data Flow: From LLM Response to Persisted Result](#data-flow-from-llm-response-to-persisted-result)
3. [Core Types: Understanding the Building Blocks](#core-types-understanding-the-building-blocks)
   - [ProviderResponse: The Raw LLM Output](#providerresponse-the-raw-llm-output)
   - [GradingResult: The Assertion Verdict](#gradingresult-the-assertion-verdict)
   - [EvaluateResult: The Complete Picture](#evaluateresult-the-complete-picture)
   - [TokenUsage: Tracking the Cost](#tokenusage-tracking-the-cost)
4. [The EvalResult Persistence Model](#the-evalresult-persistence-model)
   - [Why a Separate Persistence Layer?](#why-a-separate-persistence-layer)
   - [Field-by-Field Breakdown](#field-by-field-breakdown)
   - [Sanitization: Preparing Data for Storage](#sanitization-preparing-data-for-storage)
5. [The Eval Model: Container for Results](#the-eval-model-container-for-results)
   - [The Version Evolution](#the-version-evolution)
   - [Loading Results: Lazy vs Eager](#loading-results-lazy-vs-eager)
   - [Batched Operations for Scale](#batched-operations-for-scale)
6. [Aggregate Metrics: Summarizing the Evaluation](#aggregate-metrics-summarizing-the-evaluation)
   - [EvaluateStats: Top-Level Summary](#evaluatestats-top-level-summary)
   - [PromptMetrics: Per-Prompt Performance](#promptmetrics-per-prompt-performance)
   - [Named Scores: Custom Dimensions](#named-scores-custom-dimensions)
7. [Result Failure Reasons: Why Did It Fail?](#result-failure-reasons-why-did-it-fail)
   - [The Three Failure States](#the-three-failure-states)
   - [Why Distinguish Failure Types?](#why-distinguish-failure-types)
8. [The AssertionsResult Aggregator](#the-assertionsresult-aggregator)
   - [Accumulating Individual Assertion Results](#accumulating-individual-assertion-results)
   - [Weighted Scoring Logic](#weighted-scoring-logic)
   - [Threshold-Based Pass/Fail Determination](#threshold-based-passfail-determination)
9. [Table Representation: Results for Display](#table-representation-results-for-display)
   - [EvaluateTable Structure](#evaluatetable-structure)
   - [EvaluateTableRow and EvaluateTableOutput](#evaluatetablerow-and-evaluatetableoutput)
   - [Why a Separate Table Format?](#why-a-separate-table-format)
10. [Filtering and Querying Results](#filtering-and-querying-results)
    - [Filter Modes](#filter-modes)
    - [Building Filter WHERE Clauses](#building-filter-where-clauses)
    - [Pagination and Performance](#pagination-and-performance)
11. [Serialization and Export](#serialization-and-export)
    - [EvaluateSummary Versions](#evaluatesummary-versions)
    - [ResultsFile: The Complete Export](#resultsfile-the-complete-export)
    - [Privacy-Aware Stripping](#privacy-aware-stripping)
12. [Practical Examples](#practical-examples)
    - [Example 1: Simple Pass](#example-1-simple-pass)
    - [Example 2: Assertion Failure](#example-2-assertion-failure)
    - [Example 3: API Error](#example-3-api-error)
    - [Example 4: Multiple Assertions with Named Scores](#example-4-multiple-assertions-with-named-scores)
13. [File Reference](#file-reference)
14. [Key Design Decisions](#key-design-decisions)

---

## The Central Question: What Data Must an Evaluation Produce?

Before diving into types and code, let's understand what problem the result model solves. When you run `promptfoo eval`, you're asking a fundamental question: "How good are my prompts and providers?"

The answer to that question needs to be:
- **Actionable**: You need enough detail to know *what* to fix
- **Comparable**: You need to compare this run to previous runs
- **Storable**: You need to persist results for later analysis
- **Shareable**: You need to export and share with teammates

### The Multiple Audiences for Results

Different consumers need different views of the same data:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           Who Needs Results?                                │
│                                                                             │
│  ┌───────────────┐     ┌───────────────┐     ┌───────────────┐              │
│  │ Developer     │     │ CI Pipeline   │     │ Analyst       │              │
│  │               │     │               │     │               │              │
│  │ Needs:        │     │ Needs:        │     │ Needs:        │              │
│  │ - Why failed? │     │ - Pass/fail   │     │ - Trends      │              │
│  │ - Full output │     │ - Exit code   │     │ - Comparisons │              │
│  │ - Debug info  │     │ - Summary     │     │ - Aggregates  │              │
│  └───────────────┘     └───────────────┘     └───────────────┘              │
│          │                    │                    │                        │
│          └────────────────────┼────────────────────┘                        │
│                               ▼                                             │
│                    ┌──────────────────────┐                                 │
│                    │ Result Model         │                                 │
│                    │                      │                                 │
│                    │ Must satisfy ALL     │                                 │
│                    │ these audiences      │                                 │
│                    └──────────────────────┘                                 │
└─────────────────────────────────────────────────────────────────────────────┘
```

### The Hierarchy of Information

Results exist at multiple levels of granularity:

```
Level 1: Eval (Entire Evaluation Run)
├── Stats: 950 passed, 45 failed, 5 errors
├── Duration: 127 seconds
├── Total cost: $3.42
│
├── Level 2: Prompt (Each prompt template tested)
│   ├── Prompt 1 metrics: 320 passed, 15 failed
│   ├── Prompt 2 metrics: 315 passed, 17 failed
│   └── Prompt 3 metrics: 315 passed, 13 failed
│
│   └── Level 3: Test Case (Each input tested)
│       ├── Test 1: { vars: {...}, pass: true, score: 0.92 }
│       ├── Test 2: { vars: {...}, pass: false, score: 0.45 }
│       └── ...
│
│       └── Level 4: Assertion (Each check applied)
│           ├── contains "hello": pass
│           ├── llm-rubric "helpful": score 0.8
│           └── latency < 3000ms: pass
```

The result model must represent all these levels while allowing you to drill down from "overall evaluation passed" to "this specific assertion on this specific test case failed because..."

---

## Architecture Overview

### The Result Hierarchy

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              Eval                                           │
│  (One per evaluation run, contains configuration and aggregate stats)       │
│                                                                             │
│  id: "eval-abc-2024-01-15T10:30:00"                                         │
│  config: { prompts: [...], providers: [...], tests: [...] }                 │
│  prompts: CompletedPrompt[] (with metrics)                                  │
│  durationMs: 127000                                                         │
└──────────────────────────────────┬──────────────────────────────────────────┘
                                   │
                                   │ has many
                                   ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                            EvalResult                                       │
│  (One per test case × prompt × provider combination)                        │
│                                                                             │
│  id: "uuid"                                                                 │
│  evalId: "eval-abc-..."                                                     │
│  testIdx: 0, promptIdx: 0                                                   │
│  testCase: { vars: {...}, assert: [...] }                                   │
│  provider: { id: "openai:gpt-4o", ... }                                     │
│  response: { output: "...", latencyMs: 450, ... }                           │
│  success: true/false                                                        │
│  score: 0.85                                                                │
│  gradingResult: GradingResult                                               │
└──────────────────────────────────┬──────────────────────────────────────────┘
                                   │
                                   │ contains
                                   ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                           GradingResult                                     │
│  (Aggregated result from all assertions on this test case)                  │
│                                                                             │
│  pass: true/false                                                           │
│  score: 0.85                                                                │
│  reason: "Aggregate score 0.85 ≥ 0.75 threshold"                            │
│  namedScores: { accuracy: 0.9, helpfulness: 0.8 }                           │
│  tokensUsed: { total: 150, prompt: 100, completion: 50 }                    │
│  componentResults: GradingResult[]                                          │
└──────────────────────────────────┬──────────────────────────────────────────┘
                                   │
                                   │ contains many
                                   ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                     GradingResult (Component)                               │
│  (One per individual assertion)                                             │
│                                                                             │
│  pass: true                                                                 │
│  score: 0.9                                                                 │
│  reason: "Response contains 'hello'"                                        │
│  assertion: { type: "contains", value: "hello" }                            │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Data Flow: From LLM Response to Persisted Result

Understanding how data flows through the system helps you understand what each type represents:

```
1. PROVIDER CALL
   ┌────────────────────────────────────────────────────────────────────────┐
   │ provider.callApi(prompt) → ProviderResponse                            │
   │                                                                        │
   │ Contains:                                                              │
   │ - output: "The capital of France is Paris..."                          │
   │ - latencyMs: 450                                                       │
   │ - tokenUsage: { prompt: 50, completion: 120 }                          │
   │ - cost: 0.002                                                          │
   │ - cached: false                                                        │
   └────────────────────────────────┬───────────────────────────────────────┘
                                    │
                                    ▼
2. ASSERTION EVALUATION
   ┌────────────────────────────────────────────────────────────────────────┐
   │ runAssertions(providerResponse, test) → GradingResult                  │
   │                                                                        │
   │ For each assertion in test.assert:                                     │
   │   - Run assertion handler                                              │
   │   - Collect individual GradingResult                                   │
   │                                                                        │
   │ Aggregate into final GradingResult with:                               │
   │   - Weighted average score                                             │
   │   - Overall pass/fail                                                  │
   │   - componentResults array                                             │
   └────────────────────────────────┬───────────────────────────────────────┘
                                    │
                                    ▼
3. RESULT CONSTRUCTION
   ┌────────────────────────────────────────────────────────────────────────┐
   │ Construct EvaluateResult                                               │
   │                                                                        │
   │ Combines:                                                              │
   │ - Test case (inputs, vars, assertions)                                 │
   │ - Provider response (output, latency, cost)                            │
   │ - Grading result (pass/fail, score, reasons)                           │
   │ - Metadata (prompt index, test index, provider info)                   │
   └────────────────────────────────┬───────────────────────────────────────┘
                                    │
                                    ▼
4. PERSISTENCE
   ┌────────────────────────────────────────────────────────────────────────┐
   │ EvalResult.createFromEvaluateResult(evalId, result)                    │
   │                                                                        │
   │ - Sanitizes provider (removes circular references)                     │
   │ - Extracts binary data to blob storage                                 │
   │ - Inserts into eval_results table                                      │
   │ - Updates signal file for real-time UI updates                         │
   └────────────────────────────────────────────────────────────────────────┘
```

[📂 src/evaluator.ts#L461-L591](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/evaluator.ts#L461-L591)

---

## Core Types: Understanding the Building Blocks

### ProviderResponse: The Raw LLM Output

`ProviderResponse` captures everything the LLM provider returns—not just the text output, but all the metadata that accompanies it.

[📂 src/types/providers.ts#L137-L199](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/types/providers.ts#L137-L199)

```typescript
interface ProviderResponse {
  // The actual generated content
  output?: string | any;        // Can be string, object, or array
  
  // Performance and cost
  latencyMs?: number;           // How long the API call took
  cost?: number;                // Estimated cost in dollars
  tokenUsage?: TokenUsage;      // Detailed token breakdown
  
  // Caching
  cached?: boolean;             // Was this served from cache?
  
  // Error handling
  error?: string;               // Error message if call failed
  
  // Debugging information
  raw?: string | any;           // Raw API response before parsing
  
  // Provider-specific metadata
  metadata?: {
    redteamFinalPrompt?: string;  // For red team: final attack prompt
    http?: {                       // HTTP-level details
      status: number;
      statusText: string;
      headers: Record<string, string>;
    };
    [key: string]: any;           // Provider-specific fields
  };
  
  // Model behavior indicators
  logProbs?: number[];          // Log probabilities (for perplexity)
  isRefusal?: boolean;          // Did the model refuse to answer?
  finishReason?: string;        // "stop", "length", "content_filter"
  
  // Guardrail results
  guardrails?: {
    flaggedInput?: boolean;
    flaggedOutput?: boolean;
    reason?: string;
  };
  
  // Multimodal outputs
  audio?: { ... };              // Audio generation results
  video?: { ... };              // Video generation results
}
```

**Design Rationale: Why So Many Optional Fields?**

Different providers return vastly different data:
- OpenAI returns `finish_reason` and detailed token counts
- Anthropic returns `stop_reason`
- Local models might return nothing beyond the output
- Multimodal models return audio/video metadata

Rather than creating separate response types for each provider, promptfoo uses a single interface with optional fields. This means:
- Unified handling in evaluation code
- Easy addition of new providers
- Consumers check for field existence before use

**The `output` Field Complexity**:

`output` is typed as `string | any` because LLMs can return:
- Plain text: `"The answer is 42"`
- JSON objects: `{ "answer": 42, "confidence": 0.95 }`
- Arrays: `["option1", "option2", "option3"]`
- Function call structures: `{ "name": "get_weather", "arguments": {...} }`

The evaluation system coerces this to string for text-based assertions (`outputString`) while preserving the original type for structural assertions like `is-json`.

### GradingResult: The Assertion Verdict

`GradingResult` represents the outcome of evaluating assertions. It's used at two levels:
1. Individual assertion results (one per assertion)
2. Aggregated result (combining all assertions on a test case)

[📂 src/types/index.ts#L421-L463](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/types/index.ts#L421-L463)

```typescript
interface GradingResult {
  // Core verdict
  pass: boolean;              // Binary: did it pass?
  score: number;              // Continuous: how well? (typically 0-1)
  reason: string;             // Human-readable explanation
  
  // Metrics
  namedScores?: Record<string, number>;   // Custom metric values
  tokensUsed?: TokenUsage;    // Tokens consumed by model-graded assertions
  
  // Composition
  componentResults?: GradingResult[];     // For aggregated results
  assertion?: Assertion;      // The assertion that produced this (for components)
  
  // User interaction
  comment?: string;           // User-added comment (from UI)
  suggestions?: ResultSuggestion[];  // AI-suggested improvements
  
  // Extended information
  metadata?: {
    pluginId?: string;        // For red team: which plugin
    strategyId?: string;      // For red team: which strategy
    context?: string | string[];  // For context-based assertions
    renderedAssertionValue?: string;  // Computed assertion value
    renderedGradingPrompt?: string;   // For debugging model-graded
  };
}
```

**Why Both `pass` and `score`?**

This is a crucial design decision that enables different use cases:

| Use Case | Needs `pass` | Needs `score` |
|----------|--------------|---------------|
| CI gate: should build fail? | ✓ | |
| Compare prompts: which is better? | | ✓ |
| Track improvement over time | | ✓ |
| Debugging: why did it fail? | ✓ | |
| Threshold-based passing | ✓ | ✓ (compared to threshold) |

Consider this scenario:
```
Assertion: similar
Threshold: 0.75

Response A: score = 0.76 → pass = true (barely)
Response B: score = 0.99 → pass = true (excellent)
```

Both pass, but `score` reveals that B is significantly better. Without `score`, you'd lose this signal.

**The `componentResults` Pattern**:

For a test with multiple assertions, the aggregated `GradingResult` contains `componentResults`—an array of individual assertion results:

```
Aggregated GradingResult:
  pass: true
  score: 0.85
  reason: "All assertions passed"
  componentResults: [
    { pass: true, score: 1.0, reason: "Contains 'hello'", assertion: { type: "contains" } },
    { pass: true, score: 0.9, reason: "Similarity 0.9", assertion: { type: "similar" } },
    { pass: true, score: 0.65, reason: "Score 6.5/10", assertion: { type: "llm-rubric" } }
  ]
```

This enables:
- Debugging: Which assertion failed?
- Analysis: Which assertion type is weakest?
- UI: Display individual assertion results

### EvaluateResult: The Complete Picture

`EvaluateResult` is the primary data structure for a single test execution. It combines everything: input, output, and judgment.

[📂 src/types/index.ts#L296-L317](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/types/index.ts#L296-L317)

```typescript
interface EvaluateResult {
  // Identity
  id?: string;                    // Unique identifier
  promptIdx: number;              // Which prompt (0-indexed)
  testIdx: number;                // Which test case (0-indexed)
  promptId: string;               // Hash of the prompt
  
  // Inputs
  prompt: Prompt;                 // The prompt template used
  testCase: AtomicTestCase;       // The test case with vars and assertions
  vars: Vars;                     // Substituted variables
  
  // Provider
  provider: Pick<ProviderOptions, 'id' | 'label'>;
  
  // Output
  response?: ProviderResponse;    // Full LLM response
  error?: string | null;          // Error message if any
  
  // Verdict
  success: boolean;               // Did the test pass?
  score: number;                  // Aggregate score
  failureReason: ResultFailureReason;  // Why it failed
  gradingResult?: GradingResult | null;  // Full grading details
  namedScores: Record<string, number>;  // Named metrics
  
  // Performance
  latencyMs: number;              // Response time
  cost?: number;                  // Estimated cost
  tokenUsage?: Required<TokenUsage>;  // Token breakdown
  
  // Metadata
  metadata?: Record<string, any>; // Custom metadata
  description?: string;           // Human-readable description
}
```

**Field-by-Field Explanation**:

**`promptIdx` and `testIdx`**: These indices are crucial for reconstructing the evaluation grid. If you have 3 prompts and 100 tests, there are 300 results. Each result's position in the grid is `(testIdx, promptIdx)`.

**`prompt` vs `promptId`**: The `prompt` field contains the full prompt object. `promptId` is a hash of the prompt for deduplication. If two evaluations use the same prompt, they'll have the same `promptId`, enabling prompt-level comparisons across evals.

**`testCase` vs `vars`**: `testCase` is the full test case including assertions. `vars` is the resolved variable values after any file loading or computation. They're stored separately because:
- `testCase.vars` might be `{ query: "file://questions.txt" }`
- `vars` would be `{ query: "What is 2+2?" }` (the loaded value)

**`success` vs `gradingResult.pass`**: These should always match, but `success` is denormalized for easier querying. You can filter results by `success = true` without parsing the `gradingResult` JSON.

### TokenUsage: Tracking the Cost

Token usage is critical for cost tracking and optimization.

[📂 src/types/shared.ts#L15-L43](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/types/shared.ts#L15-L43)

```typescript
interface TokenUsage {
  // Core token counts
  prompt?: number;          // Input tokens
  completion?: number;      // Output tokens
  total?: number;           // Sum of prompt + completion
  
  // Caching
  cached?: number;          // Tokens served from cache (cost reduction)
  
  // Request tracking
  numRequests?: number;     // Number of API calls made
  
  // Assertion-specific
  assertions?: {            // Tokens used by model-graded assertions
    total?: number;
    prompt?: number;
    completion?: number;
    cached?: number;
    numRequests?: number;
  };
}
```

**Why Separate Assertion Tokens?**

Model-graded assertions (like `llm-rubric`) make their own API calls. You need to know:
- How much did the tested model cost? (main tokens)
- How much did evaluation cost? (assertion tokens)

If assertion tokens exceed generation tokens, you might be over-evaluating or need to switch to a cheaper grader model.

**Concrete Example**:

```
Evaluating 1000 test cases with:
- GPT-4o as the tested model
- gpt-4o-mini as the grader for llm-rubric assertions

Token breakdown:
  Generation: 50M prompt tokens + 10M completion tokens
  Assertions: 100M prompt tokens + 5M completion tokens

Cost:
  Generation: (50M × $2.50/M) + (10M × $10/M) = $225
  Assertions: (100M × $0.15/M) + (5M × $0.60/M) = $18

Without this breakdown, you'd see $243 and not know the split.
```

---

## The EvalResult Persistence Model

While `EvaluateResult` is the in-memory type, `EvalResult` is the Active Record model for database persistence.

[📂 src/models/evalResult.ts](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/models/evalResult.ts)

### Why a Separate Persistence Layer?

The separation exists because:

1. **Serialization concerns**: `EvaluateResult` can contain circular references, functions, and non-serializable objects (like `ApiProvider` instances). `EvalResult` handles sanitization.

2. **Database-specific optimizations**: `EvalResult` knows about database queries, batching, and indexes. `EvaluateResult` is pure data.

3. **Two-way conversion**: You need to convert `EvaluateResult → EvalResult` when saving and `EvalResult → EvaluateResult` when loading.

### Field-by-Field Breakdown

```typescript
class EvalResult {
  id: string;                           // UUID, primary key
  evalId: string;                       // Foreign key to evals table
  
  // Position in the evaluation grid
  promptIdx: number;
  testIdx: number;
  
  // Inputs (serialized)
  testCase: AtomicTestCase;             // JSON column
  prompt: Prompt;                       // JSON column
  promptId: string;                     // Hash, for indexing
  
  // Provider (sanitized)
  provider: ProviderOptions;            // Circular refs removed
  
  // Output
  response: ProviderResponse | undefined;  // Binary data externalized
  error?: string | null;
  
  // Verdict
  success: boolean;
  score: number;
  failureReason: ResultFailureReason;
  gradingResult: GradingResult | null;
  namedScores: Record<string, number>;
  
  // Performance
  latencyMs: number;
  cost: number;
  
  // Extensibility
  metadata: Record<string, any>;
  
  // Internal state
  persisted: boolean;                   // Has this been saved to DB?
}
```

### Sanitization: Preparing Data for Storage

Before storing, data must be sanitized to:
1. Remove circular references
2. Convert functions to their outputs
3. Handle non-serializable types

[📂 src/models/evalResult.ts#L24-L62](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/models/evalResult.ts#L24-L62)

```typescript
function sanitizeProvider(provider: ApiProvider | ProviderOptions | string): ProviderOptions {
  if (isApiProvider(provider)) {
    // ApiProvider has methods like id() - we need to call them
    return {
      id: provider.id(),        // Call the method
      label: provider.label,    // Copy static property
      config: provider.config ? 
        JSON.parse(safeJsonStringify(provider.config)) : undefined
    };
  }
  // ... handle other cases
}
```

**Why `safeJsonStringify`?**

`JSON.stringify` fails on circular references. `safeJsonStringify` handles them by:
1. Detecting cycles during traversal
2. Replacing repeated references with `[Circular]`
3. Never throwing, always returning a string

**Binary Data Extraction**:

Large binary data (audio, images) is extracted before storage:

```typescript
const processedResponse = await extractAndStoreBinaryData(result.response, {
  evalId,
  testIdx,
  promptIdx,
});
```

This replaces inline base64 data with `promptfoo://blob/{hash}` references, preventing database bloat.

---

## The Eval Model: Container for Results

The `Eval` class represents an entire evaluation run and manages its results.

[📂 src/models/eval.ts](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/models/eval.ts)

### The Version Evolution

Promptfoo's data model has evolved:

```
Version 2/3 (Legacy):
  - Results stored inline in evals.results JSON column
  - Table representation pre-computed and stored
  - Fast reads, but expensive writes and updates
  - Limited scalability (entire results in memory)

Version 4 (Current):
  - Results stored in separate eval_results table
  - Table generated dynamically
  - Efficient streaming of large evaluations
  - Supports incremental updates
```

```typescript
version() {
  // Check if old-style results exist
  return this.oldResults && 'table' in this.oldResults ? 3 : 4;
}

useOldResults() {
  return this.version() < 4;
}
```

**Why Keep Supporting Old Versions?**

Users have historical evaluations. Dropping support would make old data inaccessible. The version check allows transparent reading of old data while writing new data in the current format.

### Loading Results: Lazy vs Eager

Results are loaded lazily to avoid memory issues with large evaluations:

```typescript
class Eval {
  results: EvalResult[];          // Empty until loaded
  _resultsLoaded: boolean = false;
  
  async loadResults() {
    this.results = await EvalResult.findManyByEvalId(this.id);
    this._resultsLoaded = true;
  }
  
  async getResults() {
    if (!this._resultsLoaded) {
      await this.loadResults();
    }
    return this.results;
  }
}
```

**Why Lazy Loading?**

Consider an evaluation with 100,000 results. Loading all of them:
- Consumes hundreds of MB of memory
- Takes seconds to deserialize
- Is unnecessary if you only need statistics

By loading on demand, you can:
- List evaluations quickly (metadata only)
- Paginate through results efficiently
- Stream results for export

### Batched Operations for Scale

For very large evaluations, even lazy loading might be too much. Batched iteration helps:

```typescript
static async *findManyByEvalIdBatched(
  evalId: string,
  opts?: { batchSize?: number }
): AsyncGenerator<EvalResult[]> {
  const batchSize = opts?.batchSize || 100;
  let offset = 0;

  while (true) {
    const results = await db
      .select()
      .from(evalResultsTable)
      .where(
        and(
          eq(evalResultsTable.evalId, evalId),
          gte(evalResultsTable.testIdx, offset),
          lt(evalResultsTable.testIdx, offset + batchSize),
        ),
      )
      .all();

    if (results.length === 0) break;
    yield results.map(r => new EvalResult({ ...r, persisted: true }));
    offset += batchSize;
  }
}
```

Usage:
```typescript
for await (const batch of eval.fetchResultsBatched(100)) {
  // Process 100 results at a time
  for (const result of batch) {
    // ...
  }
}
```

**Why Batch by Test Index?**

Batching by `testIdx` ensures all results for a given test case are in the same batch (since there's one result per prompt×provider combination for each test). This simplifies downstream processing that groups by test.

---

## Aggregate Metrics: Summarizing the Evaluation

### EvaluateStats: Top-Level Summary

`EvaluateStats` provides a quick overview of the entire evaluation:

[📂 src/types/index.ts#L379-L385](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/types/index.ts#L379-L385)

```typescript
interface EvaluateStats {
  successes: number;        // Tests that passed
  failures: number;         // Tests that failed assertions
  errors: number;           // Tests that errored (API failure, timeout)
  tokenUsage: Required<TokenUsage>;  // Aggregate token consumption
  durationMs?: number;      // Total evaluation time
}
```

**Computed from Prompt Metrics**:

Stats are derived from `PromptMetrics` to ensure consistency:

```typescript
getStats(): EvaluateStats {
  const stats = {
    successes: 0,
    failures: 0,
    errors: 0,
    tokenUsage: createEmptyTokenUsage(),
    durationMs: this.durationMs,
  };

  for (const prompt of this.prompts) {
    stats.successes += prompt.metrics?.testPassCount ?? 0;
    stats.failures += prompt.metrics?.testFailCount ?? 0;
    stats.errors += prompt.metrics?.testErrorCount ?? 0;
    accumulateTokenUsage(stats.tokenUsage, prompt.metrics?.tokenUsage);
  }

  return stats;
}
```

### PromptMetrics: Per-Prompt Performance

Each prompt template has its own metrics:

[📂 src/types/index.ts#L230-L250](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/types/index.ts#L230-L250)

```typescript
interface PromptMetrics {
  // Aggregate score across all tests
  score: number;
  
  // Test outcomes
  testPassCount: number;
  testFailCount: number;
  testErrorCount: number;
  
  // Assertion outcomes (can be higher than test counts due to multiple assertions)
  assertPassCount: number;
  assertFailCount: number;
  
  // Performance
  totalLatencyMs: number;
  tokenUsage: TokenUsage;
  cost: number;
  
  // Named metrics (averaged across tests)
  namedScores: Record<string, number>;
  namedScoresCount: Record<string, number>;  // For computing averages
  
  // Red team specific
  redteam?: {
    pluginPassCount: Record<string, number>;
    pluginFailCount: Record<string, number>;
    strategyPassCount: Record<string, number>;
    strategyFailCount: Record<string, number>;
  };
}
```

**Why Separate Assertion Counts?**

A single test can have multiple assertions:
```yaml
tests:
  - vars: { ... }
    assert:
      - type: contains
        value: "hello"
      - type: llm-rubric
        value: "helpful"
      - type: latency
        threshold: 3000
```

This test has 3 assertions. It might:
- Pass overall (2/3 assertions pass, aggregate score ≥ threshold)
- Have assertPassCount = 2, assertFailCount = 1
- Have testPassCount = 1

Tracking both levels lets you analyze:
- "How many tests passed?" (testPassCount)
- "How reliable is my similarity assertion?" (assertPassCount for that type)

### Named Scores: Custom Dimensions

Named scores track custom metrics across the evaluation:

```yaml
assert:
  - type: llm-rubric
    value: "factually accurate"
    metric: accuracy
  - type: llm-rubric
    value: "helpful"
    metric: helpfulness
```

Results:
```typescript
promptMetrics.namedScores = {
  accuracy: 8.5,      // Sum across all tests
  helpfulness: 7.2,
};

promptMetrics.namedScoresCount = {
  accuracy: 100,      // Number of tests contributing
  helpfulness: 100,
};

// Average accuracy = 8.5 / 100 = 0.085
```

**Why Track Counts?**

Some tests might not have all metrics (e.g., accuracy only checked for factual questions). The count lets you compute correct averages:

```
Test 1: accuracy=0.9, helpfulness=0.8
Test 2: accuracy=0.8  (no helpfulness check)
Test 3: helpfulness=0.7

namedScores = { accuracy: 1.7, helpfulness: 1.5 }
namedScoresCount = { accuracy: 2, helpfulness: 2 }

Average accuracy = 1.7/2 = 0.85
Average helpfulness = 1.5/2 = 0.75
```

---

## Result Failure Reasons: Why Did It Fail?

### The Three Failure States

[📂 src/types/index.ts#L280-L288](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/types/index.ts#L280-L288)

```typescript
const ResultFailureReason = {
  NONE: 0,    // Test passed, or we don't know exactly why it failed
  ASSERT: 1,  // Test failed because an assertion rejected it
  ERROR: 2,   // Test failed due to an error (API failure, timeout, exception)
} as const;
```

### Why Distinguish Failure Types?

Different failure types require different remediation:

| Failure Reason | Meaning | Action |
|----------------|---------|--------|
| `NONE` | Test passed | Celebrate! |
| `ASSERT` | LLM responded, but response didn't meet criteria | Improve prompt or adjust assertion |
| `ERROR` | LLM didn't respond properly | Check API key, rate limits, timeout settings |

**Filtering by Failure Reason**:

The UI and API support filtering by failure reason:

```typescript
// Show only assertion failures (not errors)
if (mode === 'failures') {
  conditions.push(`success = 0 AND failure_reason != ${ResultFailureReason.ERROR}`);
}

// Show only errors
if (mode === 'errors') {
  conditions.push(`failure_reason = ${ResultFailureReason.ERROR}`);
}
```

**Concrete Example**:

```
100 test cases:
  - 85 passed (NONE)
  - 10 failed assertions (ASSERT)
  - 5 errored (ERROR)

If you see 15 "failures":
  - 10 are assertion failures → analyze LLM responses
  - 5 are errors → check infrastructure

Different problems, different solutions.
```

---

## The AssertionsResult Aggregator

`AssertionsResult` is the class that accumulates individual assertion results and computes the final verdict.

[📂 src/assertions/assertionsResult.ts](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/assertions/assertionsResult.ts)

### Accumulating Individual Assertion Results

```typescript
class AssertionsResult {
  private totalScore: number = 0;
  private totalWeight: number = 0;
  private componentResults: GradingResult[] = [];
  private namedScores: Record<string, number> = {};
  private tokensUsed = { total: 0, prompt: 0, completion: 0, ... };
  private failedReason: string | undefined;
  
  addResult({
    index,
    result,
    metric,
    weight = 1,
  }) {
    // Accumulate weighted score
    this.totalScore += result.score * weight;
    this.totalWeight += weight;
    
    // Store for later
    this.componentResults[index] = result;
    
    // Track named metrics
    if (metric) {
      this.namedScores[metric] = (this.namedScores[metric] || 0) + result.score;
    }
    
    // Accumulate tokens
    if (result.tokensUsed) {
      this.tokensUsed.total += result.tokensUsed.total || 0;
      // ...
    }
    
    // Track first failure
    if (!result.pass && !this.failedReason) {
      this.failedReason = result.reason;
    }
  }
}
```

### Weighted Scoring Logic

The aggregate score is a weighted average:

```
score = Σ(result_i.score × weight_i) / Σ(weight_i)
```

**Example**:

```yaml
assert:
  - type: contains
    value: "hello"
    weight: 1
  - type: llm-rubric
    value: "helpful"
    weight: 3
```

Results:
```
contains: score=1.0, weight=1
llm-rubric: score=0.7, weight=3

aggregate = (1.0×1 + 0.7×3) / (1+3)
          = (1.0 + 2.1) / 4
          = 0.775
```

### Threshold-Based Pass/Fail Determination

```typescript
async testResult(scoringFunction?: ScoringFunction): Promise<GradingResult> {
  const score = this.totalWeight > 0 ? this.totalScore / this.totalWeight : 0;
  
  let pass = !this.failedReason;
  let reason = this.failedReason ? this.failedReason : 'All assertions passed';
  
  // Threshold overrides individual pass/fail
  if (this.threshold) {
    pass = score >= this.threshold;
    reason = pass
      ? `Aggregate score ${score.toFixed(2)} ≥ ${this.threshold} threshold`
      : `Aggregate score ${score.toFixed(2)} < ${this.threshold} threshold`;
  }
  
  return {
    pass,
    score,
    reason,
    namedScores: this.namedScores,
    tokensUsed: this.tokensUsed,
    componentResults: this.componentResults,
  };
}
```

**Two Pass/Fail Modes**:

| Mode | Behavior |
|------|----------|
| No threshold | Pass if ALL assertions pass |
| With threshold | Pass if aggregate score ≥ threshold |

The threshold mode is more forgiving—a weak assertion can be compensated by strong ones.

---

## Table Representation: Results for Display

### EvaluateTable Structure

The evaluation UI displays results as a table with prompts as columns and tests as rows:

[📂 src/types/index.ts#L371-L377](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/types/index.ts#L371-L377)

```typescript
interface EvaluateTable {
  head: {
    prompts: CompletedPrompt[];  // Column headers
    vars: string[];              // Variable names shown
  };
  body: EvaluateTableRow[];      // Rows of data
}
```

```
Visual representation:

          │ Prompt 1    │ Prompt 2    │ Prompt 3    │
          │ (gpt-4o)    │ (claude)    │ (gpt-mini)  │
──────────┼─────────────┼─────────────┼─────────────┤
Test 1    │ ✓ 0.95      │ ✓ 0.88      │ ✗ 0.45      │
query:... │ "Paris..."  │ "Paris..."  │ "London..." │
──────────┼─────────────┼─────────────┼─────────────┤
Test 2    │ ✓ 0.92      │ ✗ 0.65      │ ✓ 0.78      │
query:... │ "Yes..."    │ "No..."     │ "Yes..."    │
```

### EvaluateTableRow and EvaluateTableOutput

[📂 src/types/index.ts#L363-L368](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/types/index.ts#L363-L368)

```typescript
interface EvaluateTableRow {
  description?: string;          // Test description
  outputs: EvaluateTableOutput[];  // One per prompt (column)
  vars: string[];                // Variable values (for display)
  test: AtomicTestCase;          // Full test case
  testIdx: number;               // Position in test list
}

interface EvaluateTableOutput {
  // Identity
  id: string;
  
  // Display
  text: string;                  // Truncated output for display
  pass: boolean;
  score: number;
  
  // Details (expandable)
  response?: ProviderResponse;
  gradingResult?: GradingResult;
  testCase: AtomicTestCase;
  
  // Metadata
  latencyMs: number;
  cost: number;
  tokenUsage?: Partial<TokenUsage>;
  namedScores: Record<string, number>;
  error?: string;
  failureReason: ResultFailureReason;
  
  // Provider info
  provider?: string;
  prompt: string;
  
  // Multimodal
  audio?: { ... };
  video?: { ... };
}
```

### Why a Separate Table Format?

`EvaluateResult[]` is a flat list. `EvaluateTable` is a 2D grid. The transformation:

```
EvaluateResult[] (flat):
  [
    { testIdx: 0, promptIdx: 0, ... },
    { testIdx: 0, promptIdx: 1, ... },
    { testIdx: 0, promptIdx: 2, ... },
    { testIdx: 1, promptIdx: 0, ... },
    { testIdx: 1, promptIdx: 1, ... },
    ...
  ]

EvaluateTable (grid):
  {
    head: { prompts: [p0, p1, p2], vars: [...] },
    body: [
      { testIdx: 0, outputs: [result00, result01, result02] },
      { testIdx: 1, outputs: [result10, result11, result12] },
    ]
  }
```

**Benefits of Table Format**:
- Natural for tabular display
- Easy to iterate by row (test) or column (prompt)
- Supports comparison: "How did the same test perform across prompts?"

---

## Filtering and Querying Results

### Filter Modes

The UI supports filtering results by status:

```typescript
type EvalResultsFilterMode = 'all' | 'passes' | 'failures' | 'errors' | 'highlights';
```

| Mode | Shows |
|------|-------|
| `all` | Every result |
| `passes` | Results where `success = true` |
| `failures` | Results where `success = false AND failureReason != ERROR` |
| `errors` | Results where `failureReason = ERROR` |
| `highlights` | Results with user-added highlight comments |

### Building Filter WHERE Clauses

[📂 src/models/eval.ts#L660-L851](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/models/eval.ts#L660-L851)

The `buildFilterWhereSql` method constructs SQL WHERE clauses for filtering:

```typescript
private buildFilterWhereSql(opts: {
  filterMode?: EvalResultsFilterMode;
  searchQuery?: string;
  filters?: string[];
}): string {
  const conditions = [`eval_id = '${this.id}'`];
  
  // Status filter
  if (mode === 'errors') {
    conditions.push(`failure_reason = ${ResultFailureReason.ERROR}`);
  } else if (mode === 'failures') {
    conditions.push(`success = 0 AND failure_reason != ${ResultFailureReason.ERROR}`);
  }
  
  // Search filter (across multiple fields)
  if (opts.searchQuery) {
    const searchConditions = [
      `response LIKE '%${sanitizedSearch}%'`,
      `json_extract(grading_result, '$.reason') LIKE '%${sanitizedSearch}%'`,
      `json_extract(test_case, '$.vars') LIKE '%${sanitizedSearch}%'`,
      // ...
    ];
    conditions.push(`(${searchConditions.join(' OR ')})`);
  }
  
  // Custom filters (metric, metadata, plugin, strategy, severity)
  if (opts.filters) {
    // Parse and apply each filter...
  }
  
  return conditions.join(' AND ');
}
```

**Why a Single Source of Truth?**

The same WHERE clause is used for:
1. Pagination (getting result IDs)
2. Metrics (calculating filtered stats)
3. Export (getting all matching results)

Having one method ensures they're always consistent.

### Pagination and Performance

Large evaluations need efficient pagination:

```typescript
async getTablePage(opts: {
  offset?: number;
  limit?: number;
  filterMode?: EvalResultsFilterMode;
  // ...
}) {
  // 1. Query for test indices (which tests match the filter?)
  const { testIndices, filteredCount } = await this.queryTestIndices(opts);
  
  // 2. Fetch all results for those test indices
  const allResults = await EvalResult.findManyByEvalIdAndTestIndices(
    this.id, 
    testIndices
  );
  
  // 3. Group by test index and build table rows
  const resultsByTestIdx = new Map<number, EvalResult[]>();
  for (const result of allResults) {
    // ...
  }
  
  return { body, totalCount, filteredCount, ... };
}
```

**The Two-Phase Approach**:

1. **Phase 1**: Query only test indices (small data)
2. **Phase 2**: Fetch full results for those indices

This is much faster than fetching all results and filtering in memory.

---

## Serialization and Export

### EvaluateSummary Versions

[📂 src/types/index.ts#L387-L401](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/types/index.ts#L387-L401)

```typescript
// Legacy format (pre-normalized)
interface EvaluateSummaryV2 {
  version: number;       // 2
  timestamp: string;
  results: EvaluateResult[];
  table: EvaluateTable;  // Pre-computed
  stats: EvaluateStats;
}

// Current format
interface EvaluateSummaryV3 {
  version: 3;
  timestamp: string;
  results: EvaluateResult[];
  prompts: CompletedPrompt[];  // Table computed on-demand
  stats: EvaluateStats;
}
```

### ResultsFile: The Complete Export

When you export or share an evaluation, `ResultsFile` contains everything:

```typescript
interface ResultsFile {
  version: number;
  createdAt: string;
  results: EvaluateSummaryV2 | EvaluateSummaryV3;
  config: Partial<UnifiedConfig>;
  author: string | null;
  prompts: CompletedPrompt[];
  datasetId: string | null;
  traces?: TraceData[];  // OpenTelemetry traces if available
}
```

### Privacy-Aware Stripping

When sharing results, you might want to hide sensitive data:

[📂 src/models/evalResult.ts#L334-L384](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/models/evalResult.ts#L334-L384)

```typescript
toEvaluateResult(): EvaluateResult {
  const shouldStripPromptText = getEnvBool('PROMPTFOO_STRIP_PROMPT_TEXT');
  const shouldStripResponseOutput = getEnvBool('PROMPTFOO_STRIP_RESPONSE_OUTPUT');
  const shouldStripTestVars = getEnvBool('PROMPTFOO_STRIP_TEST_VARS');
  
  const prompt = shouldStripPromptText
    ? { ...this.prompt, raw: '[prompt stripped]' }
    : this.prompt;
  
  const response = shouldStripResponseOutput
    ? { ...this.response, output: '[output stripped]' }
    : this.response;
  
  // ...
}
```

**Use Cases**:
- Sharing evaluations publicly without exposing prompts
- Complying with data retention policies
- Reducing export file size

---

## Practical Examples

### Example 1: Simple Pass

```typescript
// Input
const testCase = {
  vars: { query: "What is 2+2?" },
  assert: [
    { type: "contains", value: "4" }
  ]
};

// LLM Response
const providerResponse = {
  output: "The answer is 4.",
  latencyMs: 450,
  tokenUsage: { prompt: 10, completion: 5 }
};

// Assertion Result
const gradingResult = {
  pass: true,
  score: 1.0,
  reason: "Assertion passed",
  componentResults: [
    { pass: true, score: 1.0, reason: "Output contains '4'" }
  ]
};

// Final EvaluateResult
const result = {
  testIdx: 0,
  promptIdx: 0,
  success: true,
  score: 1.0,
  failureReason: ResultFailureReason.NONE,
  response: providerResponse,
  gradingResult: gradingResult,
  latencyMs: 450,
  namedScores: {},
};
```

### Example 2: Assertion Failure

```typescript
// Input
const testCase = {
  vars: { query: "What is the capital of France?" },
  assert: [
    { type: "contains", value: "Paris" },
    { type: "llm-rubric", value: "accurate and complete" }
  ]
};

// LLM Response (incorrect)
const providerResponse = {
  output: "The capital of France is Lyon.",
  latencyMs: 520,
};

// Assertion Results
const gradingResult = {
  pass: false,
  score: 0.25,
  reason: "Expected output to contain 'Paris'",
  componentResults: [
    { 
      pass: false, 
      score: 0.0, 
      reason: "Expected output to contain 'Paris'",
      assertion: { type: "contains", value: "Paris" }
    },
    { 
      pass: false, 
      score: 0.5, 
      reason: "Score: 5/10 - Response is incorrect...",
      assertion: { type: "llm-rubric", value: "accurate and complete" }
    }
  ]
};

// Final EvaluateResult
const result = {
  success: false,
  score: 0.25,  // (0.0 + 0.5) / 2
  failureReason: ResultFailureReason.ASSERT,
  error: "Expected output to contain 'Paris'",
  gradingResult: gradingResult,
};
```

### Example 3: API Error

```typescript
// LLM Response (error)
const providerResponse = {
  error: "Rate limit exceeded. Please retry after 60 seconds.",
  output: null,
};

// No grading (nothing to grade)
const gradingResult = null;

// Final EvaluateResult
const result = {
  success: false,
  score: 0,
  failureReason: ResultFailureReason.ERROR,
  error: "Rate limit exceeded. Please retry after 60 seconds.",
  response: providerResponse,
  gradingResult: null,
};
```

### Example 4: Multiple Assertions with Named Scores

```typescript
// Input
const testCase = {
  assert: [
    { type: "llm-rubric", value: "factually accurate", metric: "accuracy" },
    { type: "llm-rubric", value: "clear and helpful", metric: "helpfulness" },
    { type: "latency", threshold: 3000 }
  ]
};

// Assertion Results
const gradingResult = {
  pass: true,
  score: 0.83,  // Weighted average
  reason: "All assertions passed",
  namedScores: {
    accuracy: 0.9,
    helpfulness: 0.8,
  },
  tokensUsed: {
    total: 450,
    prompt: 350,
    completion: 100,
    numRequests: 2,  // Two llm-rubric calls
  },
  componentResults: [
    { pass: true, score: 0.9, reason: "Score: 9/10", assertion: {...} },
    { pass: true, score: 0.8, reason: "Score: 8/10", assertion: {...} },
    { pass: true, score: 1.0, reason: "Latency 450ms < 3000ms", assertion: {...} }
  ]
};

// Final EvaluateResult
const result = {
  success: true,
  score: 0.83,
  namedScores: { accuracy: 0.9, helpfulness: 0.8 },
  latencyMs: 450,
  tokenUsage: {
    prompt: 50,
    completion: 120,
    assertions: {
      total: 450,
      prompt: 350,
      completion: 100,
      numRequests: 2,
    }
  }
};
```

---

## File Reference

| File | Purpose |
|------|---------|
| [src/types/index.ts](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/types/index.ts) | Core type definitions: EvaluateResult, GradingResult, EvaluateStats |
| [src/types/providers.ts](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/types/providers.ts) | ProviderResponse, TokenUsage, ApiProvider |
| [src/types/shared.ts](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/types/shared.ts) | TokenUsage schema, shared utilities |
| [src/models/eval.ts](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/models/eval.ts) | Eval persistence model, result management |
| [src/models/evalResult.ts](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/models/evalResult.ts) | EvalResult persistence model, serialization |
| [src/assertions/assertionsResult.ts](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/assertions/assertionsResult.ts) | Assertion aggregation, scoring logic |
| [src/database/tables.ts](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/database/tables.ts) | Database schema for eval_results table |
| [src/evaluator.ts](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/evaluator.ts) | Result construction during evaluation |

---

## Key Design Decisions

| Decision | Rationale | Trade-off | Alternative Considered |
|----------|-----------|-----------|------------------------|
| Separate `pass` and `score` | Different use cases (binary gate vs continuous metric) | Slight conceptual overhead | Single field with threshold (rejected: loses nuance) |
| `EvaluateResult` vs `EvalResult` | Separate concerns: in-memory vs persisted | Two types to understand | Single type (rejected: serialization complexity) |
| Lazy result loading | Memory efficiency for large evals | Extra database calls | Eager loading (rejected: memory issues) |
| Batched iteration | Handle very large evaluations | More complex API | Stream all (rejected: memory issues) |
| Three failure reasons | Different remediation paths | More states to handle | Binary pass/fail (rejected: loses diagnostic info) |
| Component results in GradingResult | Debugging, analysis | Nested data structure | Flat list (rejected: loses grouping) |
| Separate table format | Optimized for display | Transformation required | Flat results in UI (rejected: poor UX) |
| Named scores | Custom metrics | Extra fields | Single score (rejected: loses dimensionality) |
| Version numbering | Backward compatibility | Migration complexity | Breaking changes (rejected: data loss) |
| Privacy stripping | Sharing without exposure | Data loss (intentional) | No stripping (rejected: privacy concerns) |
