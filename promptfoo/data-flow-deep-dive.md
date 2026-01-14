# Promptfoo Data Flow Deep Dive

## Table of Contents

1. [The Central Question](#the-central-question)
2. [Architecture Overview: The Complete Journey](#architecture-overview-the-complete-journey)
3. [Phase 1: Prompt Rendering](#phase-1-prompt-rendering)
   - [The Rendering Pipeline](#the-rendering-pipeline)
   - [Variable Resolution](#variable-resolution)
   - [File-Based Variables](#file-based-variables)
4. [Phase 2: Provider Call Protocol](#phase-2-provider-call-protocol)
   - [The callApi Contract](#the-callapi-contract)
   - [CallApiContextParams: The Rich Context Object](#callapicontextparams-the-rich-context-object)
   - [Why Such a Rich Context?](#why-such-a-rich-context)
5. [Phase 3: ProviderResponse—The Core Data Format](#phase-3-providerresponsethe-core-data-format)
   - [The Response Interface](#the-response-interface)
   - [Concrete Example: OpenAI Chat Response](#concrete-example-openai-chat-response)
   - [Design Rationale: Why This Structure?](#design-rationale-why-this-structure)
6. [Phase 4: Output Transformation](#phase-4-output-transformation)
   - [The Two-Phase Transform Pipeline](#the-two-phase-transform-pipeline)
   - [Transform Function Types](#transform-function-types)
   - [Why Two Phases?](#why-two-phases)
7. [Phase 5: Binary Data Extraction](#phase-5-binary-data-extraction)
   - [The Problem: Token Bloat](#the-problem-token-bloat)
   - [The Solution: Blob Externalization](#the-solution-blob-externalization)
8. [Phase 6: Assertion Evaluation](#phase-6-assertion-evaluation)
   - [The runAssertions Entry Point](#the-runassertions-entry-point)
   - [AssertionParams: Everything an Assertion Needs](#assertionparams-everything-an-assertion-needs)
   - [The coerceString Function](#the-coercestring-function)
9. [Phase 7: GradingResult—The Assertion Output](#phase-7-gradingresultthe-assertion-output)
   - [The GradingResult Interface](#the-gradingresult-interface)
   - [Concrete Examples](#concrete-examples)
10. [Phase 8: Score Aggregation](#phase-8-score-aggregation)
    - [The AssertionsResult Class](#the-assertionsresult-class)
    - [Weighted Average Scoring](#weighted-average-scoring)
    - [Threshold-Based Override](#threshold-based-override)
11. [Phase 9: EvaluateResult Construction](#phase-9-evaluateresult-construction)
    - [The EvaluateResult Interface](#the-evaluateresult-interface)
    - [How It All Comes Together](#how-it-all-comes-together)
12. [Phase 10: Persistence](#phase-10-persistence)
    - [The Transition from Runtime to Storage](#the-transition-from-runtime-to-storage)
    - [What Gets Stored?](#what-gets-stored)
13. [Complete Data Flow Diagram](#complete-data-flow-diagram)
14. [Concrete End-to-End Example](#concrete-end-to-end-example)
15. [Key Design Decisions](#key-design-decisions)
16. [File Reference](#file-reference)

## The Central Question

When you run a promptfoo evaluation, data flows through many transformations before becoming a stored result. Understanding this flow is essential for:

1. **Debugging**: When something goes wrong, you need to know where in the pipeline the issue occurred
2. **Customization**: To write custom transforms, assertions, or providers, you must understand what data you receive and what you must return
3. **Performance Optimization**: Knowing where data transformations happen helps identify bottlenecks
4. **Extension Development**: Building plugins or integrations requires understanding the data contracts

This document traces the complete journey of data from the moment a prompt template is rendered until the final result is persisted to the database.

## Architecture Overview: The Complete Journey

Before diving into details, here's the 30,000-foot view of the data flow:

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                           PROMPTFOO DATA FLOW                                   │
│                                                                                 │
│  ┌──────────────┐      ┌──────────────┐      ┌──────────────┐                   │
│  │   PROMPT     │      │   PROVIDER   │      │   RESPONSE   │                   │
│  │   TEMPLATE   │ ──▶  │    CALL      │ ──▶  │   RECEIVED   │                   │
│  │              │      │              │      │              │                   │
│  │  "Hello      │      │  OpenAI      │      │  { output:   │                   │
│  │   {{name}}"  │      │  API call    │      │    "Hi!" }   │                   │
│  └──────────────┘      └──────────────┘      └──────────────┘                   │
│         │                                            │                          │
│         │ Phase 1: Render                            │ Phase 4: Transform       │
│         ▼                                            ▼                          │
│  ┌──────────────┐                            ┌──────────────┐                   │
│  │   RENDERED   │                            │ TRANSFORMED  │                   │
│  │    PROMPT    │                            │   OUTPUT     │                   │
│  │              │                            │              │                   │
│  │  "Hello      │                            │  Extracted/  │                   │
│  │   Alice"     │                            │  Normalized  │                   │
│  └──────────────┘                            └──────────────┘                   │
│                                                      │                          │
│                                                      │ Phase 5: Blob Extraction │
│                                                      ▼                          │
│  ┌──────────────┐      ┌──────────────┐      ┌──────────────┐                   │
│  │   GRADING    │ ◀──  │  ASSERTION   │ ◀──  │   CLEANED    │                   │
│  │   RESULT     │      │  EVALUATION  │      │   RESPONSE   │                   │
│  │              │      │              │      │              │                   │
│  │  { pass: T,  │      │  Handler     │      │  Binary data │                   │
│  │    score: 1} │      │  execution   │      │  externalized│                   │
│  └──────────────┘      └──────────────┘      └──────────────┘                   │
│         │                                                                       │
│         │ Phase 8: Aggregate                                                    │
│         ▼                                                                       │
│  ┌──────────────┐      ┌──────────────┐      ┌──────────────┐                   │
│  │  AGGREGATED  │ ──▶  │  EVALUATE    │ ──▶  │   PERSISTED  │                   │
│  │   GRADING    │      │   RESULT     │      │     EVAL     │                   │
│  │              │      │              │      │    RESULT    │                   │
│  │  Weighted    │      │  Complete    │      │              │                   │
│  │  average     │      │  record      │      │  SQLite row  │                   │
│  └──────────────┘      └──────────────┘      └──────────────┘                   │
└─────────────────────────────────────────────────────────────────────────────────┘
```

Each box represents a distinct data format. The arrows represent transformations. Let's explore each one in detail.

## Phase 1: Prompt Rendering

### The Rendering Pipeline

The journey begins with a **prompt template**—a string containing placeholders that will be filled with test variables. The rendering process converts this template into a concrete string that can be sent to an LLM.

**Source**: [src/evaluatorHelpers.ts#L220-L340](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/evaluatorHelpers.ts#L220-L340)

```typescript
export async function renderPrompt(
  prompt: Prompt,
  vars: Record<string, VarValue>,
  nunjucksFilters?: NunjucksFilterMap,
  provider?: ApiProvider,
  skipRenderVars?: string[],
): Promise<string>
```

**The `Prompt` object** contains:

| Field | Type | Purpose |
|-------|------|---------|
| `raw` | `string` | The template text with `{{placeholders}}` |
| `label` | `string` | Human-readable identifier |
| `id` | `string?` | Unique hash for caching |
| `config` | `any?` | Provider config overrides |
| `function` | `PromptFunction?` | Dynamic prompt generator |

**Source**: [src/types/prompts.ts#L46-L55](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/types/prompts.ts#L46-L55)

### Variable Resolution

Variables (`vars`) are the test-specific data that gets substituted into the template. They come from your `promptfooconfig.yaml` test cases:

```yaml
tests:
  - vars:
      name: "Alice"
      age: 30
      preferences:
        color: "blue"
```

The `vars` type is `Record<string, VarValue>`, where `VarValue` can be:

- `string` - Simple text
- `number` - Numeric values
- `boolean` - True/false
- `object` - Nested structures (serialized to JSON)
- `unknown[]` - Arrays

**Source**: [src/types/shared.ts#L47-L49](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/types/shared.ts#L47-L49)

### File-Based Variables

A powerful feature is loading variables from files. When a variable value starts with `file://`, the rendering system loads the file contents:

**Source**: [src/evaluatorHelpers.ts#L232-L336](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/evaluatorHelpers.ts#L232-L336)

```typescript
if (typeof value === 'string' && value.startsWith('file://')) {
  const filePath = path.resolve(process.cwd(), basePath, value.slice('file://'.length));
  const fileExtension = filePath.split('.').pop();
  
  // Different handling based on file type:
  // - .js/.ts: Execute and get output
  // - .py: Run Python script
  // - .yaml/.yml: Parse as YAML, serialize to JSON
  // - .pdf: Extract text
  // - Images/Audio/Video: Base64 encode
  // - Other: Read as text
}
```

This is a **design decision with important implications**:

1. **JavaScript files** are executed as functions, receiving context:
   ```javascript
   module.exports = async function(varName, prompt, vars, provider) {
     return { output: `Computed value for ${varName}` };
   };
   ```

2. **Python files** call a `get_var` function:
   ```python
   def get_var(var_name, prompt, vars):
       return {"output": f"Python computed value for {var_name}"}
   ```

3. **Image files** become data URLs:
   ```
   data:image/png;base64,iVBORw0KGgo...
   ```

**Concrete Example**:

```yaml
# promptfooconfig.yaml
prompts:
  - "Describe what you see: {{image}}"
tests:
  - vars:
      image: "file://./test-image.png"
```

After rendering, the prompt becomes:
```
Describe what you see: data:image/png;base64,iVBORw0KGgoAAAANSUhEUg...
```

## Phase 2: Provider Call Protocol

### The callApi Contract

Once the prompt is rendered, it's sent to a **provider**—an abstraction over an LLM API. Every provider must implement the `ApiProvider` interface:

**Source**: [src/types/providers.ts#L94-L112](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/types/providers.ts#L94-L112)

```typescript
export interface ApiProvider {
  id: () => string;              // Unique identifier
  callApi: CallApiFunction;      // The main API call method
  config?: any;                  // Provider configuration
  delay?: number;                // Rate limiting delay (ms)
  label?: ProviderLabel;         // Display name
  transform?: string;            // Output transformation code
  cleanup?: () => void | Promise<void>;  // Resource cleanup
  // ... other optional methods
}
```

The core method is `callApi`:

**Source**: [src/types/providers.ts#L228-L235](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/types/providers.ts#L228-L235)

```typescript
export type CallApiFunction = {
  (
    prompt: string,                    // The rendered prompt text
    context?: CallApiContextParams,    // Rich context object
    options?: CallApiOptionsParams,    // Options like abort signal
  ): Promise<ProviderResponse>;
  label?: string;
};
```

### CallApiContextParams: The Rich Context Object

When a provider is called, it receives not just the prompt string, but a rich context object containing everything it might need:

**Source**: [src/types/providers.ts#L53-L84](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/types/providers.ts#L53-L84)

```typescript
export interface CallApiContextParams {
  // Template variables (the resolved vars after file loading)
  vars: Record<string, VarValue>;
  
  // The original prompt object (not just the rendered string)
  prompt: Prompt;
  
  // Custom Nunjucks filters for rendering
  filters?: NunjucksFilterMap;
  
  // Reference to the provider itself
  originalProvider?: ApiProvider;
  
  // The full test case being evaluated
  test?: AtomicTestCase;
  
  // Winston logger instance
  logger?: winston.Logger;
  
  // Cache access function
  getCache?: any;
  
  // Which repetition (when repeat > 1)
  repeatIndex?: number;
  
  // Force cache bypass
  bustCache?: boolean;
  
  // W3C Trace Context for distributed tracing
  traceparent?: string;  // Format: version-trace-id-parent-id-trace-flags
  
  // Evaluation identifiers for correlation
  evaluationId?: string;
  testCaseId?: string;
  testIdx?: number;
  promptIdx?: number;
}
```

**Concrete Example**:

When this configuration runs:

```yaml
providers:
  - openai:gpt-4o
tests:
  - vars:
      question: "What is 2+2?"
```

The provider receives:

```javascript
// prompt (first argument)
"What is 2+2?"

// context (second argument)
{
  vars: { question: "What is 2+2?" },
  prompt: { raw: "{{question}}", label: "math-question" },
  originalProvider: /* OpenAI provider instance */,
  test: {
    vars: { question: "What is 2+2?" },
    assert: [{ type: "equals", value: "4" }],
    // ...
  },
  repeatIndex: 0,
  traceparent: "00-abc123...-def456...-01"
}
```

### Why Such a Rich Context?

You might wonder why providers need all this information. The reasons include:

1. **Multi-turn conversations**: Providers need access to `vars._conversation` to maintain state
2. **Conditional logic**: Providers may behave differently based on test configuration
3. **Tracing**: Distributed tracing requires passing trace context through the call
4. **Caching**: Providers need cache access to avoid redundant calls
5. **Model-graded assertions**: Some providers (like graders) need to know about the original provider

This is a **design trade-off**: providing more context increases coupling but enables more sophisticated provider behaviors.

## Phase 3: ProviderResponse—The Core Data Format

### The Response Interface

`ProviderResponse` is the central data format in promptfoo. Every LLM call returns this structure:

**Source**: [src/types/providers.ts#L137-L199](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/types/providers.ts#L137-L199)

```typescript
export interface ProviderResponse {
  // ═══════════════════════════════════════════════════════════════
  // CORE OUTPUT - What the model generated
  // ═══════════════════════════════════════════════════════════════
  output?: string | any;        // The generated content (string or structured)
  raw?: string | any;           // Unparsed API response (for debugging)
  
  // ═══════════════════════════════════════════════════════════════
  // PERFORMANCE METRICS - How well did it perform?
  // ═══════════════════════════════════════════════════════════════
  latencyMs?: number;           // Time taken for the call
  cost?: number;                // Estimated cost in dollars
  cached?: boolean;             // Was this a cache hit?
  tokenUsage?: TokenUsage;      // Token breakdown (prompt, completion, total)
  
  // ═══════════════════════════════════════════════════════════════
  // STATUS - Did it succeed?
  // ═══════════════════════════════════════════════════════════════
  error?: string;               // Error message if failed
  finishReason?: string;        // "stop", "length", "content_filter"
  isRefusal?: boolean;          // Did the model refuse to answer?
  
  // ═══════════════════════════════════════════════════════════════
  // GUARDRAILS - Safety information
  // ═══════════════════════════════════════════════════════════════
  guardrails?: {
    flagged?: boolean;
    flaggedInput?: boolean;
    flaggedOutput?: boolean;
    reason?: string;
  };
  
  // ═══════════════════════════════════════════════════════════════
  // METADATA - Additional context
  // ═══════════════════════════════════════════════════════════════
  metadata?: {
    http?: {                    // HTTP details for debugging
      status: number;
      statusText: string;
      headers: Record<string, string>;
    };
    [key: string]: any;         // Provider-specific metadata
  };
  
  // ═══════════════════════════════════════════════════════════════
  // SPECIAL OUTPUTS - Non-text content
  // ═══════════════════════════════════════════════════════════════
  logProbs?: number[];          // Log probabilities for each token
  
  audio?: {
    data?: string;              // Base64 encoded audio
    transcript?: string;        // Text transcription
    format?: string;            // "wav", "mp3", etc.
    duration?: number;          // Duration in seconds
    blobRef?: BlobRef;          // Reference to externalized blob
  };
  
  video?: {
    url?: string;               // Video URL or blob reference
    format?: string;            // "mp4"
    duration?: number;          // Duration in seconds
    blobRef?: BlobRef;          // Reference to externalized blob
  };
  
  // ═══════════════════════════════════════════════════════════════
  // TRANSFORMATION TRACKING
  // ═══════════════════════════════════════════════════════════════
  isBase64?: boolean;           // Is output base64 encoded?
  format?: string;              // Format hint (e.g., "json")
  providerTransformedOutput?: string | any;  // Output after provider transform
}
```

### Concrete Example: OpenAI Chat Response

Here's how the OpenAI provider constructs a `ProviderResponse`:

**Source**: [src/providers/openai/chat.ts#L718-L734](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/providers/openai/chat.ts#L718-L734)

```typescript
return {
  output,                                    // "The answer is 4"
  tokenUsage: getTokenUsage(data, cached),   // { prompt: 10, completion: 5, total: 15 }
  cached,                                    // false
  latencyMs,                                 // 450
  logProbs,                                  // [-0.5, -0.3, ...]
  ...(finishReason && { finishReason }),     // "stop"
  cost: calculateOpenAICost(                 // 0.0001
    this.modelName,
    config,
    data.usage?.prompt_tokens,
    data.usage?.completion_tokens,
  ),
  guardrails: { flagged: contentFiltered },  // { flagged: false }
};
```

**Different response scenarios**:

```javascript
// SUCCESS: Normal response
{
  output: "Paris is the capital of France.",
  tokenUsage: { prompt: 15, completion: 8, total: 23 },
  cached: false,
  latencyMs: 523,
  finishReason: "stop",
  cost: 0.000115
}

// ERROR: API failure
{
  error: "Rate limit exceeded. Please retry after 60 seconds.",
  // No output field when there's an error
}

// REFUSAL: Model declined to answer
{
  output: "I cannot provide instructions for illegal activities.",
  tokenUsage: { prompt: 20, completion: 12, total: 32 },
  isRefusal: true,
  guardrails: { flagged: true },
  finishReason: "stop"
}

// CONTENT FILTER: Triggered safety filter
{
  output: "Content filtered by provider",
  isRefusal: true,
  finishReason: "content_filter",
  guardrails: { flagged: true }
}

// AUDIO OUTPUT: Speech generation
{
  output: "",  // Empty because content is in audio
  audio: {
    data: "UklGRiQAAABXQVZFZ...",  // Base64 WAV
    transcript: "Hello, how are you?",
    format: "wav",
    duration: 2.5
  },
  tokenUsage: { prompt: 10, completion: 0, total: 10 }
}
```

### Design Rationale: Why This Structure?

The `ProviderResponse` structure reflects several design decisions:

1. **Optional `output`**: Sometimes there's no output (errors) or the output is in a special field (audio/video). Making it optional handles all cases.

2. **Separate `error` field**: Rather than throwing exceptions, providers return error information. This allows the evaluation to continue with other test cases even if one fails.

3. **`raw` for debugging**: Preserving the raw API response helps debug issues without modifying production code.

4. **`cached` flag**: Knowing whether a response was cached affects rate limiting behavior—we don't delay after cache hits.

5. **`providerTransformedOutput`**: Tracks the output after provider-level transforms, separate from test-level transforms. This enables assertions to operate on provider-normalized output.

## Phase 4: Output Transformation

### The Two-Phase Transform Pipeline

Once the provider returns a response, the output may undergo transformation before assertion evaluation:

**Source**: [src/evaluator.ts#L512-L534](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/evaluator.ts#L512-L534)

```typescript
// Phase A: Provider-level transform
if (provider.transform) {
  processedResponse.output = await transform(provider.transform, processedResponse.output, {
    vars,
    prompt,
  });
}

// Store the provider-transformed output for assertions
const providerTransformedOutput = processedResponse.output;

// Phase B: Test-level transform
const testTransform = test.options?.transform || test.options?.postprocess;
if (testTransform) {
  processedResponse.output = await transform(testTransform, processedResponse.output, {
    vars,
    prompt,
    ...(response && response.metadata && { metadata: response.metadata }),
  });
}
```

### Transform Function Types

The `transform` function supports multiple formats:

**Source**: [src/util/transform.ts#L169-L190](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/util/transform.ts#L169-L190)

**1. Inline JavaScript (single expression)**:
```yaml
transform: "output.trim().toLowerCase()"
```

**2. Inline JavaScript (multi-line)**:
```yaml
transform: |
  const parsed = JSON.parse(output);
  return parsed.answer;
```

**3. External JavaScript file**:
```yaml
transform: "file://./transforms/extract-answer.js"
```

The JavaScript file exports a function:
```javascript
// transforms/extract-answer.js
module.exports = function(output, context) {
  const { vars, prompt } = context;
  const parsed = JSON.parse(output);
  return parsed.answer;
};
```

**4. External Python file**:
```yaml
transform: "file://./transforms/process.py:extract_answer"
```

The Python file defines the named function:
```python
# transforms/process.py
def extract_answer(output, context):
    import json
    parsed = json.loads(output)
    return parsed['answer']
```

### Why Two Phases?

The two-phase transform design solves a real problem:

**Problem**: You want provider normalization to happen consistently, but also want test-specific transformations.

**Example**:
- **Provider transform**: Your Claude provider always wraps responses in `<response>` tags. The provider transform strips these tags.
- **Test transform**: Some tests need to extract JSON from the response; others need the raw text.

```yaml
providers:
  - id: anthropic:claude-3-opus
    transform: "output.replace(/<\/?response>/g, '')"  # Always runs

tests:
  - vars: { query: "List three colors" }
    options:
      transform: "JSON.parse(output)"  # Only for this test
    assert:
      - type: javascript
        value: "Array.isArray(output) && output.length === 3"
```

The `providerTransformedOutput` field in `ProviderResponse` tracks the output after provider transform, allowing `contextTransform` assertions to operate on consistently normalized output.

## Phase 5: Binary Data Extraction

### The Problem: Token Bloat

When providers return binary data (audio, images, video), storing this data inline causes two problems:

1. **Database bloat**: A single audio response might be 1MB of base64 data
2. **Token explosion**: Model-graded assertions would need to include this data in their prompts

### The Solution: Blob Externalization

Before assertion evaluation, promptfoo extracts binary data and stores it separately:

**Source**: [src/evaluator.ts#L538-L546](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/evaluator.ts#L538-L546)

```typescript
// Externalize large blobs before grading to avoid token bloat
const blobbedResponse = await extractAndStoreBinaryData(processedResponse, {
  evalId,
  testIdx,
  promptIdx,
});
if (blobbedResponse) {
  processedResponse = blobbedResponse;
}
```

The extraction function identifies and externalizes binary data:

**Source**: [src/blobs/extractor.ts#L206-L236](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/blobs/extractor.ts#L206-L236)

```typescript
export async function extractAndStoreBinaryData(
  response: ProviderResponse | null | undefined,
  context?: BlobContext,
): Promise<ProviderResponse | null | undefined> {
  if (!response) {
    return response;
  }

  let mutated = false;
  const next: ProviderResponse = { ...response };

  // Audio at top level
  if (response.audio?.data && typeof response.audio.data === 'string') {
    const stored = await maybeStore(
      response.audio.data,
      normalizeAudioMimeType(response.audio.format),
      context,
      'response.audio.data',
      'audio',
    );
    if (stored) {
      next.audio = {
        ...response.audio,
        data: undefined,           // Remove the inline data
        blobRef: stored,           // Add reference to stored blob
      };
      mutated = true;
    }
  }
  // ... similar handling for video and base64 output
}
```

**Before externalization**:
```javascript
{
  output: "",
  audio: {
    data: "UklGRiQAAABXQVZFZm10IBAAAAABAAEA...",  // 1MB of base64
    transcript: "Hello world",
    format: "wav"
  }
}
```

**After externalization**:
```javascript
{
  output: "",
  audio: {
    data: undefined,  // Removed!
    blobRef: {
      hash: "sha256:abc123...",
      uri: "promptfoo://blob/sha256:abc123..."
    },
    transcript: "Hello world",
    format: "wav"
  }
}
```

The blob data is stored in a separate table (`blob_assets`) and can be retrieved when needed for display or playback.

## Phase 6: Assertion Evaluation

### The runAssertions Entry Point

After transformation and blob extraction, the response is evaluated against assertions:

**Source**: [src/evaluator.ts#L559-L571](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/evaluator.ts#L559-L571)

```typescript
const checkResult = await runAssertions({
  prompt: renderedPrompt,
  provider,
  providerResponse: {
    ...processedResponse,
    providerTransformedOutput,  // Add for contextTransform
  },
  test,
  latencyMs: response.latencyMs ?? latencyMs,
  assertScoringFunction: test.assertScoringFunction as ScoringFunction,
  traceId,
});
```

The `runAssertions` function orchestrates all assertion evaluation:

**Source**: [src/assertions/index.ts#L508-L608](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/assertions/index.ts#L508-L608)

```typescript
export async function runAssertions({
  assertScoringFunction,
  latencyMs,
  prompt,
  provider,
  providerResponse,
  test,
  traceId,
}: {
  assertScoringFunction?: ScoringFunction;
  latencyMs?: number;
  prompt?: string;
  provider?: ApiProvider;
  providerResponse: ProviderResponse;
  test: AtomicTestCase;
  traceId?: string;
}): Promise<GradingResult> {
  // If no assertions, test passes automatically
  if (!test.assert || test.assert.length < 1) {
    return AssertionsResult.noAssertsResult();
  }

  // Create result aggregator
  const mainAssertResult = new AssertionsResult({
    threshold: test.threshold,
  });

  // Run assertions concurrently (up to ASSERTIONS_MAX_CONCURRENCY)
  await async.forEachOfLimit(
    asserts,
    ASSERTIONS_MAX_CONCURRENCY,
    async ({ assertion, assertResult, index }) => {
      const result = await runAssertion({ /* ... */ });
      assertResult.addResult({
        index,
        result,
        metric: renderMetricName(assertion.metric, test.vars || {}),
        weight: assertion.weight,
      });
    },
  );

  // Aggregate and return final result
  return mainAssertResult.testResult(assertScoringFunction);
}
```

### AssertionParams: Everything an Assertion Needs

Each assertion handler receives a comprehensive `AssertionParams` object:

**Source**: [src/assertions/index.ts#L455-L472](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/assertions/index.ts#L455-L472)

```typescript
const assertionParams: AssertionParams = {
  assertion,                // The assertion configuration
  baseType,                 // Normalized type (e.g., "contains" from "not-contains")
  
  // The actual output to test
  output,                   // Raw output (string or object)
  outputString,             // Coerced to string for text operations
  
  // The expected value (after rendering)
  renderedValue,            // Template-rendered assertion value
  
  // Assertion modifiers
  inverse,                  // true for "not-X" assertions
  
  // Context
  test,                     // The full test case
  provider,                 // The provider used
  providerResponse,         // Full response (for metadata access)
  prompt,                   // The rendered prompt
  
  // Performance data
  latencyMs,                // For latency assertions
  cost,                     // For cost assertions
  logProbs,                 // For perplexity assertions
  
  // For model-graded assertions
  providerCallContext,      // Context to pass to grader calls
  assertionValueContext,    // Context for assertion value functions
  
  // Script output (if value was from a script)
  valueFromScript,          // Result from JavaScript/Python value
};
```

**Concrete Example**:

For this test:
```yaml
tests:
  - vars:
      city: "Paris"
    assert:
      - type: contains
        value: "{{city}}"
```

The `AssertionParams` would be:
```javascript
{
  assertion: { type: "contains", value: "{{city}}" },
  baseType: "contains",
  output: "Paris is the capital of France.",
  outputString: "Paris is the capital of France.",
  renderedValue: "Paris",  // "{{city}}" rendered with vars
  inverse: false,
  test: { vars: { city: "Paris" }, assert: [...] },
  latencyMs: 450,
  // ...
}
```

### The coerceString Function

Many assertions need string output, but `output` can be an object. The `coerceString` utility handles this:

**Source**: [src/assertions/utils.ts#L73-L78](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/assertions/utils.ts#L73-L78)

```typescript
export function coerceString(value: string | object): string {
  if (typeof value === 'string') {
    return value;
  }
  return JSON.stringify(value);
}
```

This allows assertions like `contains` to work with both:
- String output: `"Hello world"` → `"Hello world"`
- Object output: `{ message: "Hello" }` → `'{"message":"Hello"}'`

## Phase 7: GradingResult—The Assertion Output

### The GradingResult Interface

Every assertion must return a `GradingResult`:

**Source**: [src/types/index.ts#L421-L465](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/types/index.ts#L421-L465)

```typescript
export interface GradingResult {
  // ═══════════════════════════════════════════════════════════════
  // CORE VERDICT - Did it pass?
  // ═══════════════════════════════════════════════════════════════
  pass: boolean;              // Binary pass/fail
  score: number;              // Quality score (usually 0-1)
  reason: string;             // Human-readable explanation
  
  // ═══════════════════════════════════════════════════════════════
  // METRICS - Named scores for tracking
  // ═══════════════════════════════════════════════════════════════
  namedScores?: Record<string, number>;  // Custom metrics
  
  // ═══════════════════════════════════════════════════════════════
  // RESOURCE USAGE - For model-graded assertions
  // ═══════════════════════════════════════════════════════════════
  tokensUsed?: TokenUsage;    // Tokens consumed by grading LLM
  
  // ═══════════════════════════════════════════════════════════════
  // COMPOSITION - For aggregate assertions
  // ═══════════════════════════════════════════════════════════════
  componentResults?: GradingResult[];  // Sub-assertion results
  assertion?: Assertion;               // The assertion evaluated
  
  // ═══════════════════════════════════════════════════════════════
  // USER FEEDBACK
  // ═══════════════════════════════════════════════════════════════
  comment?: string;           // User-facing comment
  suggestions?: ResultSuggestion[];  // Improvement suggestions
  
  // ═══════════════════════════════════════════════════════════════
  // ADDITIONAL CONTEXT
  // ═══════════════════════════════════════════════════════════════
  metadata?: {
    pluginId?: string;        // Red team plugin identifier
    strategyId?: string;      // Red team strategy identifier
    context?: string;         // RAG context used
    renderedAssertionValue?: string;  // What the value resolved to
    renderedGradingPrompt?: string;   // LLM-as-judge prompt
    [key: string]: any;
  };
}
```

### Concrete Examples

**Simple binary assertion (contains)**:
```javascript
{
  pass: true,
  score: 1,
  reason: "Output contains 'Paris'"
}
```

**Failing assertion**:
```javascript
{
  pass: false,
  score: 0,
  reason: "Expected output to contain 'London' but got 'Paris is the capital...'"
}
```

**Continuous scoring (similarity)**:
```javascript
{
  pass: true,
  score: 0.92,
  reason: "Similarity: 0.92 (threshold: 0.8)"
}
```

**Model-graded assertion (llm-rubric)**:
```javascript
{
  pass: true,
  score: 0.85,
  reason: "The response accurately describes the capital with relevant details",
  tokensUsed: {
    prompt: 150,
    completion: 50,
    total: 200
  },
  metadata: {
    renderedGradingPrompt: "Evaluate the following response..."
  }
}
```

**Composite assertion (assert-set)**:
```javascript
{
  pass: true,
  score: 0.9,
  reason: "All assertions passed",
  componentResults: [
    { pass: true, score: 1, reason: "Contains 'Paris'" },
    { pass: true, score: 0.8, reason: "Similarity: 0.8" }
  ]
}
```

## Phase 8: Score Aggregation

### The AssertionsResult Class

The `AssertionsResult` class aggregates individual assertion results into a final verdict:

**Source**: [src/assertions/assertionsResult.ts#L21-L186](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/assertions/assertionsResult.ts#L21-L186)

```typescript
export class AssertionsResult {
  private tokensUsed = { /* ... */ };
  private threshold: number | undefined;
  private totalScore: number = 0;
  private totalWeight: number = 0;
  private failedReason: string | undefined;
  private componentResults: GradingResult[] = [];
  private namedScores: Record<string, number> = {};
  
  addResult({
    index,
    result,
    metric,
    weight = 1,
  }: {
    index: number;
    result: GradingResult;
    metric?: string;
    weight?: number;
  }) {
    // Weighted score accumulation
    this.totalScore += result.score * weight;
    this.totalWeight += weight;
    this.componentResults[index] = result;
    
    // Named score accumulation
    if (metric) {
      this.namedScores[metric] = (this.namedScores[metric] || 0) + result.score;
    }
    
    // Token usage accumulation
    if (result.tokensUsed) {
      this.tokensUsed.total += result.tokensUsed.total || 0;
      // ...
    }
    
    // Track failure reason
    if (!result.pass) {
      this.failedReason = result.reason;
    }
  }
}
```

### Weighted Average Scoring

The final score is a weighted average:

```typescript
const score = this.totalWeight > 0 ? this.totalScore / this.totalWeight : 0;
```

**Example**:

```yaml
assert:
  - type: contains
    value: "Paris"
    weight: 2          # 2x weight
  - type: similar
    value: "France"
    threshold: 0.7
    weight: 1          # 1x weight
```

If `contains` passes (score=1) and `similar` scores 0.8:
```
finalScore = (1×2 + 0.8×1) / (2+1) = 2.8/3 = 0.933
```

### Threshold-Based Override

When a test has a `threshold`, the pass/fail status is based on aggregate score, not individual assertions:

**Source**: [src/assertions/assertionsResult.ts#L122-L131](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/assertions/assertionsResult.ts#L122-L131)

```typescript
if (this.threshold) {
  // Existence of a test threshold overrides individual assertions
  pass = score >= this.threshold;

  if (pass) {
    reason = `Aggregate score ${score.toFixed(2)} ≥ ${this.threshold} threshold`;
  } else {
    reason = `Aggregate score ${score.toFixed(2)} < ${this.threshold} threshold`;
  }
}
```

**Why this design?**

Thresholds enable "partial credit" evaluation. A test with 5 assertions might pass if 4/5 succeed (threshold: 0.8), even though one assertion failed. This is useful for:
- Evaluating complex, multi-faceted responses
- Allowing minor imperfections in generated content
- Red team testing where "mostly safe" is acceptable

## Phase 9: EvaluateResult Construction

### The EvaluateResult Interface

After assertions complete, everything is bundled into an `EvaluateResult`:

**Source**: [src/types/index.ts#L296-L317](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/types/index.ts#L296-L317)

```typescript
export interface EvaluateResult {
  // ═══════════════════════════════════════════════════════════════
  // POSITION - Where is this result in the matrix?
  // ═══════════════════════════════════════════════════════════════
  id?: string;                // Unique identifier
  promptIdx: number;          // Column in results table
  testIdx: number;            // Row in results table
  promptId: string;           // Hash of the prompt
  
  // ═══════════════════════════════════════════════════════════════
  // INPUTS - What went in?
  // ═══════════════════════════════════════════════════════════════
  prompt: Prompt;             // The prompt used
  testCase: AtomicTestCase;   // The test case
  vars: Vars;                 // Resolved variables
  provider: Pick<ProviderOptions, 'id' | 'label'>;  // Provider info
  
  // ═══════════════════════════════════════════════════════════════
  // OUTPUTS - What came out?
  // ═══════════════════════════════════════════════════════════════
  response?: ProviderResponse;  // Full LLM response
  error?: string | null;        // Error if any
  
  // ═══════════════════════════════════════════════════════════════
  // VERDICT - Did it pass?
  // ═══════════════════════════════════════════════════════════════
  success: boolean;             // Binary pass/fail
  score: number;                // Aggregate score
  failureReason: ResultFailureReason;  // 0=NONE, 1=ASSERT, 2=ERROR
  gradingResult?: GradingResult | null;  // Full grading details
  namedScores: Record<string, number>;   // Named metrics
  
  // ═══════════════════════════════════════════════════════════════
  // PERFORMANCE - How did it perform?
  // ═══════════════════════════════════════════════════════════════
  latencyMs: number;
  cost?: number;
  tokenUsage?: Required<TokenUsage>;
  
  // ═══════════════════════════════════════════════════════════════
  // METADATA
  // ═══════════════════════════════════════════════════════════════
  description?: string;
  metadata?: Record<string, any>;
}
```

### How It All Comes Together

Here's how the evaluator constructs the `EvaluateResult`:

**Source**: [src/evaluator.ts#L461-L591](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/evaluator.ts#L461-L591)

```typescript
const ret: EvaluateResult = {
  ...setup,                   // provider, prompt, vars
  response,                   // ProviderResponse
  success: false,             // Will be updated
  failureReason: ResultFailureReason.NONE,
  score: 0,
  namedScores: {},
  latencyMs: response.latencyMs ?? latencyMs,
  cost: response.cost,
  metadata: {
    ...test.metadata,
    ...response.metadata,
  },
  promptIdx,
  testIdx,
  testCase: test,
  promptId: prompt.id || '',
  tokenUsage: createEmptyTokenUsage(),
};

// Handle error responses
if (response.error) {
  ret.error = response.error;
  ret.failureReason = ResultFailureReason.ERROR;
  ret.success = false;
} else if (response.output === null || response.output === undefined) {
  ret.success = false;
  ret.score = 0;
  ret.error = 'No output';
} else {
  // Run assertions and update result
  const checkResult = await runAssertions({ /* ... */ });
  
  if (!checkResult.pass) {
    ret.error = checkResult.reason;
    ret.failureReason = ResultFailureReason.ASSERT;
  }
  ret.success = checkResult.pass;
  ret.score = checkResult.score;
  ret.namedScores = checkResult.namedScores || {};
  ret.gradingResult = checkResult;
}

return [ret];
```

## Phase 10: Persistence

### The Transition from Runtime to Storage

The final step converts `EvaluateResult` (runtime representation) to `EvalResult` (database representation):

**Source**: [src/evaluator.ts#L1140-L1145](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/evaluator.ts#L1140-L1145)

```typescript
// Add result to eval record (triggers persistence)
await this.evalRecord.addResult(result);
```

The `Eval.addResult` method delegates to `EvalResult.createFromEvaluateResult`:

**Source**: [src/models/eval.ts#L152-L163](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/models/eval.ts#L152-L163)

```typescript
async addResult(result: EvaluateResult) {
  const newResult = await EvalResult.createFromEvaluateResult(this.id, result, {
    persist: this.persisted,
  });
  if (!this.persisted) {
    this.results.push(newResult);  // Keep in memory
  }
  if (this.persisted) {
    updateSignalFile(this.id);     // Notify UI
  }
}
```

### What Gets Stored?

The `EvalResult.createFromEvaluateResult` method prepares data for storage:

**Source**: [src/models/evalResult.ts#L81-L140](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/models/evalResult.ts#L81-L140)

```typescript
static async createFromEvaluateResult(
  evalId: string,
  result: EvaluateResult,
  opts?: { persist: boolean },
) {
  // 1. Sanitize provider (remove circular refs, serialize methods)
  const preSanitizeTestCase = {
    ...testCase,
    ...(testCase.provider && {
      provider: sanitizeProvider(testCase.provider),
    }),
  };

  // 2. Extract binary data to blob storage
  const processedResponse = await extractAndStoreBinaryData(result.response, {
    evalId,
    testIdx: result.testIdx,
    promptIdx: result.promptIdx,
  });

  // 3. Prepare database row
  const args = {
    id: crypto.randomUUID(),
    evalId,
    testCase: preSanitizeTestCase,
    promptIdx: result.promptIdx,
    testIdx: result.testIdx,
    prompt,
    promptId: hashPrompt(prompt),
    error: error?.toString(),
    success,
    score: score == null ? 0 : score,
    response: processedResponse || null,
    gradingResult: gradingResult || null,
    namedScores,
    provider: sanitizeProvider(provider),
    latencyMs,
    cost,
    metadata,
    failureReason,
  };
  
  // 4. Insert into database
  if (persist) {
    const db = getDb();
    const dbResult = await db.insert(evalResultsTable).values(args).returning();
    return new EvalResult({ ...dbResult[0], persisted: true });
  }
  return new EvalResult(args);
}
```

**Database Schema** (simplified):

**Source**: [src/database/tables.ts#L75-L130](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/database/tables.ts#L75-L130)

```sql
CREATE TABLE eval_results (
  id TEXT PRIMARY KEY,
  created_at INTEGER NOT NULL,
  eval_id TEXT NOT NULL REFERENCES evals(id),
  
  -- Position
  prompt_idx INTEGER NOT NULL,
  test_idx INTEGER NOT NULL,
  
  -- Inputs (JSON columns)
  test_case TEXT NOT NULL,      -- JSON: AtomicTestCase
  prompt TEXT NOT NULL,         -- JSON: Prompt
  prompt_id TEXT,
  provider TEXT NOT NULL,       -- JSON: ProviderOptions
  
  -- Outputs
  response TEXT,                -- JSON: ProviderResponse
  error TEXT,
  
  -- Verdict
  success INTEGER NOT NULL,     -- 0 or 1
  score REAL NOT NULL,
  failure_reason INTEGER NOT NULL,
  grading_result TEXT,          -- JSON: GradingResult
  named_scores TEXT,            -- JSON: Record<string, number>
  
  -- Performance
  latency_ms INTEGER,
  cost REAL,
  
  -- Metadata
  metadata TEXT                 -- JSON: Record<string, any>
);
```

## Complete Data Flow Diagram

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                        PROMPTFOO EVALUATION DATA FLOW                           │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                 │
│  CONFIG INPUT                                                                   │
│  ┌──────────────────────────────────────────────────────────────────────────┐   │
│  │ promptfooconfig.yaml                                                     │   │
│  │ ┌────────────────┐ ┌────────────────┐ ┌────────────────────────────────┐ │   │
│  │ │ prompts:       │ │ providers:     │ │ tests:                         │ │   │
│  │ │  - "Hello      │ │  - openai:     │ │  - vars: {name: "Alice"}       │ │   │
│  │ │    {{name}}"   │ │    gpt-4o      │ │    assert:                     │ │   │
│  │ │                │ │                │ │      - type: contains          │ │   │
│  │ │                │ │                │ │        value: "hello"          │ │   │
│  │ └────────────────┘ └────────────────┘ └────────────────────────────────┘ │   │
│  └──────────────────────────────────────────────────────────────────────────┘   │
│                                         │                                       │
│                                         ▼                                       │
│  PHASE 1: RENDER ────────────────────────────────────────────────────────────── │
│  ┌──────────────────────────────────────────────────────────────────────────┐   │
│  │ renderPrompt(prompt, vars) → string                                      │   │
│  │                                                                          │   │
│  │   Input:  "Hello {{name}}" + { name: "Alice" }                           │   │
│  │   Output: "Hello Alice"                                                  │   │
│  │                                                                          │   │
│  │   Also handles:                                                          │   │
│  │   • file:// → load file contents                                         │   │
│  │   • Images → base64 data URLs                                            │   │
│  │   • JS/Python → execute and use output                                   │   │
│  └──────────────────────────────────────────────────────────────────────────┘   │
│                                         │                                       │
│                                         ▼                                       │
│  PHASE 2: CALL PROVIDER ─────────────────────────────────────────────────────── │
│  ┌──────────────────────────────────────────────────────────────────────────┐   │
│  │ provider.callApi(prompt, context, options) → ProviderResponse            │   │
│  │                                                                          │   │
│  │   prompt:  "Hello Alice"                                                 │   │
│  │   context: { vars, test, originalProvider, traceparent, ... }            │   │
│  │   options: { abortSignal }                                               │   │
│  │                                                                          │   │
│  │   ┌─────────────────────────────────────────────────────────┐            │   │
│  │   │              OPENAI API                                  │           │   │
│  │   │  POST /v1/chat/completions                              │            │   │
│  │   │  { messages: [{role:"user", content:"Hello Alice"}] }   │            │   │
│  │   └─────────────────────────────────────────────────────────┘            │   │
│  └──────────────────────────────────────────────────────────────────────────┘   │
│                                         │                                       │
│                                         ▼                                       │
│  PHASE 3: RECEIVE RESPONSE ──────────────────────────────────────────────────── │
│  ┌──────────────────────────────────────────────────────────────────────────┐   │
│  │ ProviderResponse                                                         │   │
│  │ {                                                                        │   │
│  │   output: "Hello! It's nice to meet you, Alice!",                        │   │
│  │   tokenUsage: { prompt: 8, completion: 12, total: 20 },                  │   │
│  │   latencyMs: 342,                                                        │   │
│  │   cost: 0.00012,                                                         │   │
│  │   cached: false,                                                         │   │
│  │   finishReason: "stop"                                                   │   │
│  │ }                                                                        │   │
│  └──────────────────────────────────────────────────────────────────────────┘   │
│                                         │                                       │
│                                         ▼                                       │
│  PHASE 4: TRANSFORM ─────────────────────────────────────────────────────────── │
│  ┌──────────────────────────────────────────────────────────────────────────┐   │
│  │ transform(code, output, context) → transformedOutput                     │   │
│  │                                                                          │   │
│  │   Provider transform:   output.toLowerCase()                             │   │
│  │   Test transform:       output.split('!')[0]                             │   │
│  │                                                                          │   │
│  │   "Hello! It's nice..." → "hello! it's nice..." → "hello"                │   │
│  └──────────────────────────────────────────────────────────────────────────┘   │
│                                         │                                       │
│                                         ▼                                       │
│  PHASE 5: BLOB EXTRACTION ───────────────────────────────────────────────────── │
│  ┌──────────────────────────────────────────────────────────────────────────┐   │
│  │ extractAndStoreBinaryData(response) → cleanedResponse                    │   │
│  │                                                                          │   │
│  │   Audio/Video/Image data → stored in blob_assets table                   │   │
│  │   Replaced with: { blobRef: { hash: "sha256:...", uri: "..." } }         │   │
│  └──────────────────────────────────────────────────────────────────────────┘   │
│                                         │                                       │
│                                         ▼                                       │
│  PHASE 6: ASSERTION EVALUATION ──────────────────────────────────────────────── │
│  ┌──────────────────────────────────────────────────────────────────────────┐   │
│  │ runAssertions({ providerResponse, test, ... }) → GradingResult           │   │
│  │                                                                          │   │
│  │   For each assertion in test.assert:                                     │   │
│  │   ┌─────────────────────────────────────────────────────────────────┐    │   │
│  │   │ AssertionParams                                                 │    │   │
│  │   │ {                                                               │    │   │
│  │   │   assertion: { type: "contains", value: "hello" },              │    │   │
│  │   │   output: "hello",                                              │    │   │
│  │   │   outputString: "hello",                                        │    │   │
│  │   │   renderedValue: "hello",                                       │    │   │
│  │   │   inverse: false,                                               │    │   │
│  │   │   ...                                                           │    │   │
│  │   │ }                                                               │    │   │
│  │   └─────────────────────────────────────────────────────────────────┘    │   │
│  │                              │                                           │   │
│  │                              ▼                                           │   │
│  │   ┌─────────────────────────────────────────────────────────────────┐    │   │
│  │   │ ASSERTION_HANDLERS["contains"](params) → GradingResult          │    │   │
│  │   │ { pass: true, score: 1, reason: "Output contains 'hello'" }     │    │   │
│  │   └─────────────────────────────────────────────────────────────────┘    │   │
│  └──────────────────────────────────────────────────────────────────────────┘   │
│                                         │                                       │
│                                         ▼                                       │
│  PHASE 7: SCORE AGGREGATION ─────────────────────────────────────────────────── │
│  ┌──────────────────────────────────────────────────────────────────────────┐   │
│  │ AssertionsResult.testResult() → aggregated GradingResult                 │   │
│  │                                                                          │   │
│  │   Weighted average: (1×1 + 0.8×2) / 3 = 0.867                            │   │
│  │   Threshold check:  0.867 >= 0.7 → pass                                  │   │
│  │                                                                          │   │
│  │   {                                                                      │   │
│  │     pass: true,                                                          │   │
│  │     score: 0.867,                                                        │   │
│  │     reason: "All assertions passed",                                     │   │
│  │     componentResults: [ {...}, {...} ],                                  │   │
│  │     namedScores: { accuracy: 1, similarity: 0.8 }                        │   │
│  │   }                                                                      │   │
│  └──────────────────────────────────────────────────────────────────────────┘   │
│                                         │                                       │
│                                         ▼                                       │
│  PHASE 8: RESULT CONSTRUCTION ───────────────────────────────────────────────── │
│  ┌──────────────────────────────────────────────────────────────────────────┐   │
│  │ EvaluateResult                                                           │   │
│  │ {                                                                        │   │
│  │   // Position                                                            │   │
│  │   promptIdx: 0, testIdx: 0, promptId: "abc123",                          │   │
│  │                                                                          │   │
│  │   // Inputs                                                              │   │
│  │   prompt: { raw: "Hello {{name}}", label: "greeting" },                  │   │
│  │   testCase: { vars: { name: "Alice" }, assert: [...] },                  │   │
│  │   vars: { name: "Alice" },                                               │   │
│  │   provider: { id: "openai:gpt-4o", label: "GPT-4o" },                    │   │
│  │                                                                          │   │
│  │   // Outputs                                                             │   │
│  │   response: { output: "hello", tokenUsage: {...}, ... },                 │   │
│  │                                                                          │   │
│  │   // Verdict                                                             │   │
│  │   success: true,                                                         │   │
│  │   score: 0.867,                                                          │   │
│  │   failureReason: 0,  // NONE                                             │   │
│  │   gradingResult: { pass: true, score: 0.867, componentResults: [...] },  │   │
│  │   namedScores: { accuracy: 1, similarity: 0.8 },                         │   │
│  │                                                                          │   │
│  │   // Performance                                                         │   │
│  │   latencyMs: 342,                                                        │   │
│  │   cost: 0.00012,                                                         │   │
│  │   tokenUsage: { prompt: 8, completion: 12, total: 20, assertions: {...} }│   │
│  │ }                                                                        │   │
│  └──────────────────────────────────────────────────────────────────────────┘   │
│                                         │                                       │
│                                         ▼                                       │
│  PHASE 9: PERSISTENCE ───────────────────────────────────────────────────────── │
│  ┌──────────────────────────────────────────────────────────────────────────┐   │
│  │ EvalResult.createFromEvaluateResult(evalId, result) → database row       │   │
│  │                                                                          │   │
│  │   1. sanitizeProvider() - Remove circular refs, serialize methods        │   │
│  │   2. extractAndStoreBinaryData() - Store blobs separately                │   │
│  │   3. db.insert(evalResultsTable).values(args)                            │   │
│  │   4. updateSignalFile() - Notify UI of new data                          │   │
│  │                                                                          │   │
│  │   ┌─────────────────────────────────────────────────────────────────┐    │   │
│  │   │ SQLite: eval_results table                                      │    │   │
│  │   │ ┌─────────┬────────────┬───────────┬───────────┬─────────────┐  │    │   │
│  │   │ │ id      │ eval_id    │ success   │ score     │ response    │  │    │   │
│  │   │ ├─────────┼────────────┼───────────┼───────────┼─────────────┤  │    │   │
│  │   │ │ uuid... │ eval-123   │ 1         │ 0.867     │ {...json}   │  │    │   │
│  │   │ └─────────┴────────────┴───────────┴───────────┴─────────────┘  │    │   │
│  │   └─────────────────────────────────────────────────────────────────┘    │   │
│  └──────────────────────────────────────────────────────────────────────────┘   │
│                                                                                 │
└─────────────────────────────────────────────────────────────────────────────────┘
```

## Concrete End-to-End Example

Let's trace a complete example from configuration to stored result:

### Configuration

```yaml
# promptfooconfig.yaml
prompts:
  - "What is the capital of {{country}}?"

providers:
  - id: openai:gpt-4o
    transform: "output.trim()"

tests:
  - description: "France capital test"
    vars:
      country: "France"
    options:
      transform: "output.toLowerCase()"
    assert:
      - type: contains
        value: "paris"
        weight: 2
      - type: llm-rubric
        value: "The response should be accurate and concise"
        weight: 1
    threshold: 0.7
```

### Step-by-Step Trace

**Step 1: Prompt Rendering**
```
Template: "What is the capital of {{country}}?"
Vars: { country: "France" }
→ Rendered: "What is the capital of France?"
```

**Step 2: Provider Call**
```javascript
provider.callApi(
  "What is the capital of France?",
  {
    vars: { country: "France" },
    prompt: { raw: "What is the capital of {{country}}?", label: "..." },
    test: { vars: {...}, assert: [...] },
    // ...
  }
)
```

**Step 3: Provider Response**
```javascript
{
  output: "  The capital of France is Paris.  ",
  tokenUsage: { prompt: 12, completion: 8, total: 20 },
  latencyMs: 523,
  cost: 0.00015,
  cached: false,
  finishReason: "stop"
}
```

**Step 4: Provider Transform**
```javascript
// provider.transform = "output.trim()"
"  The capital of France is Paris.  " → "The capital of France is Paris."
```

**Step 5: Test Transform**
```javascript
// test.options.transform = "output.toLowerCase()"
"The capital of France is Paris." → "the capital of france is paris."
```

**Step 6: Assertion Evaluation**

*Assertion 1: contains "paris" (weight: 2)*
```javascript
// AssertionParams
{
  output: "the capital of france is paris.",
  outputString: "the capital of france is paris.",
  renderedValue: "paris",
  inverse: false
}

// Handler result
{
  pass: true,
  score: 1,
  reason: "Output contains 'paris'"
}
```

*Assertion 2: llm-rubric (weight: 1)*
```javascript
// Calls grading LLM with rubric prompt
{
  pass: true,
  score: 0.9,
  reason: "The response is accurate and reasonably concise",
  tokensUsed: { prompt: 150, completion: 50, total: 200 }
}
```

**Step 7: Score Aggregation**
```javascript
// Weighted average
totalScore = (1 × 2) + (0.9 × 1) = 2.9
totalWeight = 2 + 1 = 3
score = 2.9 / 3 = 0.967

// Threshold check
0.967 >= 0.7 → pass = true

// Final GradingResult
{
  pass: true,
  score: 0.967,
  reason: "All assertions passed",
  componentResults: [
    { pass: true, score: 1, reason: "Output contains 'paris'" },
    { pass: true, score: 0.9, reason: "The response is accurate..." }
  ],
  namedScores: {},
  tokensUsed: { total: 200, prompt: 150, completion: 50 }
}
```

**Step 8: EvaluateResult Construction**
```javascript
{
  promptIdx: 0,
  testIdx: 0,
  promptId: "hash-abc123",
  
  prompt: { raw: "What is the capital of {{country}}?", label: "..." },
  testCase: { vars: { country: "France" }, assert: [...] },
  vars: { country: "France" },
  provider: { id: "openai:gpt-4o", label: "GPT-4o" },
  
  response: {
    output: "the capital of france is paris.",
    tokenUsage: { prompt: 12, completion: 8, total: 20 },
    latencyMs: 523,
    cost: 0.00015,
    cached: false
  },
  
  success: true,
  score: 0.967,
  failureReason: 0,
  gradingResult: { /* as above */ },
  namedScores: {},
  
  latencyMs: 523,
  cost: 0.00015,
  tokenUsage: {
    prompt: 12,
    completion: 8,
    total: 20,
    assertions: { total: 200, prompt: 150, completion: 50, numRequests: 1 }
  },
  
  description: "France capital test",
  metadata: {}
}
```

**Step 9: Persistence**
```sql
INSERT INTO eval_results (
  id, eval_id, prompt_idx, test_idx, success, score, failure_reason,
  prompt, test_case, provider, response, grading_result, named_scores,
  latency_ms, cost, metadata
) VALUES (
  'uuid-...',
  'eval-123',
  0, 0, 1, 0.967, 0,
  '{"raw":"What is the capital of {{country}}?","label":"..."}',
  '{"vars":{"country":"France"},"assert":[...]}',
  '{"id":"openai:gpt-4o","label":"GPT-4o"}',
  '{"output":"the capital of france is paris.","tokenUsage":{...}}',
  '{"pass":true,"score":0.967,"componentResults":[...]}',
  '{}',
  523, 0.00015,
  '{}'
);
```

## Key Design Decisions

| Decision | Rationale | Trade-off |
|----------|-----------|-----------|
| **Optional `output` in ProviderResponse** | Allows representing errors and special outputs (audio/video) | Requires null checks throughout the codebase |
| **Separate `error` field instead of throwing** | Evaluation continues even if one test fails | Error handling is less explicit |
| **Rich CallApiContextParams** | Enables sophisticated provider behaviors | Increases coupling, harder to test providers in isolation |
| **Two-phase transforms** | Separates provider normalization from test-specific processing | More complexity, potential for confusion |
| **Blob externalization** | Prevents database/token bloat | Adds indirection, requires blob management |
| **Weighted average scoring** | Allows prioritizing certain assertions | Non-intuitive for users expecting simple pass/fail |
| **Threshold override** | Enables "partial credit" evaluation | Can hide individual assertion failures |
| **JSON columns in SQLite** | Flexible schema for complex nested data | Harder to query, less type safety |
| **GradingResult with componentResults** | Enables composite assertions and detailed debugging | Deep nesting can be hard to navigate |

## File Reference

| Phase | Key Files |
|-------|-----------|
| **Prompt Rendering** | [src/evaluatorHelpers.ts#L220-L340](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/evaluatorHelpers.ts#L220-L340) |
| **Provider Interface** | [src/types/providers.ts#L94-L112](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/types/providers.ts#L94-L112) |
| **CallApiContextParams** | [src/types/providers.ts#L53-L84](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/types/providers.ts#L53-L84) |
| **ProviderResponse** | [src/types/providers.ts#L137-L199](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/types/providers.ts#L137-L199) |
| **Transform Utilities** | [src/util/transform.ts#L169-L190](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/util/transform.ts#L169-L190) |
| **Blob Extraction** | [src/blobs/extractor.ts#L206-L236](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/blobs/extractor.ts#L206-L236) |
| **runAssertions** | [src/assertions/index.ts#L508-L608](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/assertions/index.ts#L508-L608) |
| **AssertionParams** | [src/assertions/index.ts#L455-L472](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/assertions/index.ts#L455-L472) |
| **GradingResult** | [src/types/index.ts#L421-L465](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/types/index.ts#L421-L465) |
| **AssertionsResult** | [src/assertions/assertionsResult.ts#L21-L186](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/assertions/assertionsResult.ts#L21-L186) |
| **EvaluateResult** | [src/types/index.ts#L296-L317](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/types/index.ts#L296-L317) |
| **Result Construction** | [src/evaluator.ts#L461-L591](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/evaluator.ts#L461-L591) |
| **EvalResult Persistence** | [src/models/evalResult.ts#L81-L140](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/models/evalResult.ts#L81-L140) |
| **Database Schema** | [src/database/tables.ts#L75-L130](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/database/tables.ts#L75-L130) |
| **OpenAI Provider Example** | [src/providers/openai/chat.ts#L718-L734](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/providers/openai/chat.ts#L718-L734) |
| **TokenUsage Schema** | [src/types/shared.ts#L15-L39](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/types/shared.ts#L15-L39) |
| **Prompt Type** | [src/types/prompts.ts#L46-L55](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/types/prompts.ts#L46-L55) |
