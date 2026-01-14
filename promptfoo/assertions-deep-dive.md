# Promptfoo Assertions Deep Dive

## Table of Contents

1. [The Central Challenge: What Makes a Good LLM Response?](#the-central-challenge-what-makes-a-good-llm-response)
   - [Why Traditional Testing Falls Short](#why-traditional-testing-falls-short)
   - [The Spectrum of Certainty](#the-spectrum-of-certainty)
   - [The Multi-Criteria Reality](#the-multi-criteria-reality)
2. [Architecture Overview](#architecture-overview)
   - [The Assertion Lifecycle](#the-assertion-lifecycle)
   - [Why This Architecture?](#why-this-architecture)
3. [The Assertion Handler Pattern](#the-assertion-handler-pattern)
   - [Why a Handler Map?](#why-a-handler-map)
   - [The AssertionParams Contract](#the-assertionparams-contract)
   - [The GradingResult Contract](#the-gradingresult-contract)
   - [The Inverse Pattern: Elegant Negation](#the-inverse-pattern-elegant-negation)
4. [Assertion Categories](#assertion-categories)
   - [Category 1: Deterministic Text Matching](#category-1-deterministic-text-matching)
   - [Category 2: Structural Validation](#category-2-structural-validation)
   - [Category 3: Semantic Similarity](#category-3-semantic-similarity)
   - [Category 4: Model-Graded Assertions](#category-4-model-graded-assertions)
   - [Category 5: Custom Code Assertions](#category-5-custom-code-assertions)
   - [Category 6: Performance Assertions](#category-6-performance-assertions)
   - [Category 7: NLP Metrics](#category-7-nlp-metrics)
   - [Category 8: Trace Assertions](#category-8-trace-assertions)
5. [The Scoring System](#the-scoring-system)
   - [Why Two Concepts: Pass and Score?](#why-two-concepts-pass-and-score)
   - [Weighted Average Scoring](#weighted-average-scoring)
   - [Threshold-Based Pass/Fail](#threshold-based-passfail)
   - [Named Scores: Custom Metrics](#named-scores-custom-metrics)
   - [Custom Scoring Functions](#custom-scoring-functions)
6. [Assertion Sets: Grouping with Logic](#assertion-sets-grouping-with-logic)
   - [The Problem: Boolean Logic in Evaluation](#the-problem-boolean-logic-in-evaluation)
   - [How Assert-Sets Work](#how-assert-sets-work)
   - [Practical Patterns](#practical-patterns)
7. [Comparison Assertions](#comparison-assertions)
   - [The Unique Challenge of Relative Evaluation](#the-unique-challenge-of-relative-evaluation)
   - [`select-best`: LLM-Based Comparison](#select-best-llm-based-comparison)
   - [`max-score`: Automated Winner](#max-score-automated-winner)
8. [Value Rendering and Transformation](#value-rendering-and-transformation)
   - [Template Variables in Assertions](#template-variables-in-assertions)
   - [Output Transformation](#output-transformation)
   - [File-Based Assertion Values](#file-based-assertion-values)
   - [Script-Based Assertion Values](#script-based-assertion-values)
9. [Concurrency and Performance](#concurrency-and-performance)
   - [Assertion Concurrency](#assertion-concurrency)
   - [Short-Circuit on Failure](#short-circuit-on-failure)
   - [Caching Considerations](#caching-considerations)
10. [Validation](#validation)
    - [Why Validate Assertions?](#why-validate-assertions)
    - [Common Validation Errors](#common-validation-errors)
    - [Limits](#limits)
11. [Model-Graded Assertions Deep Dive](#model-graded-assertions-deep-dive)
    - [The Philosophy of LLM-as-Judge](#the-philosophy-of-llm-as-judge)
    - [The Grading Prompt Architecture](#the-grading-prompt-architecture)
    - [Choosing the Right Grader Model](#choosing-the-right-grader-model)
    - [Factuality: A Specialized Pattern](#factuality-a-specialized-pattern)
    - [G-Eval: Multi-Step Evaluation](#g-eval-multi-step-evaluation)
    - [Reliability and Limitations](#reliability-and-limitations)
12. [Integration Points](#integration-points)
    - [Red Team Assertions](#red-team-assertions)
    - [Guardrails](#guardrails)
13. [Practical Guidance: Choosing Assertions](#practical-guidance-choosing-assertions)
    - [Decision Framework](#decision-framework)
    - [Common Mistakes](#common-mistakes)
14. [File Reference](#file-reference)
15. [Key Design Decisions](#key-design-decisions)

## The Central Challenge: What Makes a Good LLM Response?

This is the hardest problem in LLM evaluation, and understanding it deeply is essential to using assertions effectively.

### Why Traditional Testing Falls Short

In traditional software testing, the paradigm is simple: given input X, expect output Y. You call a function with arguments and compare the result to a known correct answer. This works because the function is **deterministic**—it produces the same output every time for the same input.

```
Traditional test:
  calculateTax(100000, "CA") → 9300
  assert result == 9300  ✓
```

LLMs break this paradigm in three fundamental ways:

**1. Non-determinism**: Ask GPT-4 "What's the capital of France?" ten times, and you'll get slightly different responses each time. "Paris is the capital of France." "The capital of France is Paris." "France's capital city is Paris." All correct, all different.

**2. Semantic equivalence without textual identity**: "Yes, I can help you with that" and "Absolutely! I'd be happy to assist" convey the same meaning but share almost no words. A simple string comparison fails, yet both are equally valid responses.

**3. Subjective quality**: What makes a customer service response "helpful"? What makes an essay "well-written"? These are judgment calls that humans disagree on. There's no single correct answer—only better and worse answers.

### The Spectrum of Certainty

Promptfoo's insight is that evaluation exists on a spectrum:

```
Certainty Spectrum:

High Certainty                                              Low Certainty
(Objective)                                                 (Subjective)
     │                                                           │
     ▼                                                           ▼
┌─────────┬───────────┬──────────────┬────────────────┬──────────────┐
│ Exact   │ Pattern   │ Structural   │ Semantic       │ Quality/     │
│ Match   │ Match     │ Validation   │ Similarity     │ Judgment     │
├─────────┼───────────┼──────────────┼────────────────┼──────────────┤
│"equals" │"contains" │ "is-json"    │ "similar"      │ "llm-rubric" │
│         │"regex"    │ "is-sql"     │ embedding-     │ "factuality" │
│         │           │              │ based          │ human-like   │
└─────────┴───────────┴──────────────┴────────────────┴──────────────┘
     │                                                               │
     │ Cheap, fast, deterministic          Expensive, slow, may vary │
     │ Good for hard requirements          Good for soft quality     │
```

The key insight: **use the simplest assertion that captures your requirement**. If you need the response to contain "error code: 404", use `contains`. Don't use `llm-rubric` with "should mention error code 404"—it's slower, costs money, and might occasionally fail even when the output is correct.

### The Multi-Criteria Reality

Real-world evaluation rarely involves a single criterion. A good chatbot response might need to be:
- Accurate (doesn't make things up)
- Complete (addresses all parts of the question)
- Concise (doesn't ramble)
- On-topic (doesn't go off on tangents)
- Professional (appropriate tone)
- Safe (doesn't give harmful advice)

Each criterion might need a different evaluation approach. Accuracy might use `factuality`. Completeness might use `contains-all` for key points. Safety might use `not-contains` for dangerous patterns plus `llm-rubric` for nuanced cases.

This is why promptfoo supports multiple assertions per test—and why understanding how they combine matters.

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           Test Case Configuration                           │
│                                                                             │
│   Each test case can define multiple assertions. Assertions can be:         │
│   - Inherited from defaultTest (applied to all tests)                       │
│   - Defined inline on the specific test                                     │
│   - Grouped into assertion sets with custom thresholds                      │
│                                                                             │
│   The final list is the union: inherited + inline, with inline overriding.  │
└──────────────────────────────────┬──────────────────────────────────────────┘
                                   │
                                   ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                           runAssertions()                                   │
│                                                                             │
│   This is the orchestrator for a single test case's assertions:             │
│                                                                             │
│   1. Pre-processing: Transform outputs if needed, render template variables │
│   2. Dispatch: Send each assertion to its handler (up to 3 concurrent)      │
│   3. Collection: Gather all GradingResults                                  │
│   4. Aggregation: Compute weighted score and overall pass/fail              │
│                                                                             │
│   Note: Assertion sets are recursively processed—they're assertions that    │
│   contain other assertions, with their own threshold logic.                 │
└──────────────────────────────────┬──────────────────────────────────────────┘
                                   │
              ┌────────────────────┼────────────────────┐
              │                    │                    │
              ▼                    ▼                    ▼
     ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐
     │  runAssertion() │  │  runAssertion() │  │  runAssertion() │
     │                 │  │                 │  │                 │
     │  Single assert  │  │  Executes in    │  │  Each returns   │
     │  execution:     │  │  parallel with  │  │  GradingResult  │
     │  1. Transform   │  │  concurrency    │  │  with pass,     │
     │  2. Render vars │  │  limits         │  │  score, reason  │
     │  3. Look up     │  │                 │  │                 │
     │     handler     │  │                 │  │                 │
     │  4. Execute     │  │                 │  │                 │
     └─────────────────┘  └─────────────────┘  └─────────────────┘
                                   │
                                   ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                           AssertionsResult                                  │
│                                                                             │
│   Final aggregation:                                                        │
│   - pass: Overall pass/fail (based on threshold or all assertions passing)  │
│   - score: Weighted average of all assertion scores                         │
│   - namedScores: Individual metrics (if assertions specify `metric`)        │
│   - tokensUsed: Aggregate token consumption from model-graded assertions    │
│   - componentResults: Array of individual GradingResults (for debugging)    │
└─────────────────────────────────────────────────────────────────────────────┘
```

[📂 src/assertions/index.ts](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/assertions/index.ts)

### The Assertion Lifecycle

Let's trace a single assertion through the system to understand the data flow:

```
1. Configuration Parse
   ┌────────────────────────────────────┐
   │ yaml:                              │
   │   assert:                          │
   │     - type: contains               │
   │       value: "{{expected_word}}"   │
   └────────────────────────────────────┘
                    │
                    ▼
2. Variable Rendering (Nunjucks)
   ┌────────────────────────────────────┐
   │ vars: { expected_word: "hello" }   │
   │ → renderedValue = "hello"          │
   └────────────────────────────────────┘
                    │
                    ▼
3. Output Transformation (if specified)
   ┌────────────────────────────────────┐
   │ If assertion has `transform`:      │
   │   output = eval(transform)         │
   │ Otherwise: output unchanged        │
   └────────────────────────────────────┘
                    │
                    ▼
4. Handler Lookup
   ┌────────────────────────────────────┐
   │ ASSERTION_HANDLERS["contains"]     │
   │ → handleContains function          │
   └────────────────────────────────────┘
                    │
                    ▼
5. Handler Execution
   ┌────────────────────────────────────┐
   │ handleContains({                   │
   │   outputString: "Hello world!",    │
   │   renderedValue: "hello",          │
   │   inverse: false,                  │
   │   ...                              │
   │ })                                 │
   └────────────────────────────────────┘
                    │
                    ▼
6. Result
   ┌────────────────────────────────────┐
   │ GradingResult {                    │
   │   pass: false,  // Case mismatch   │
   │   score: 0,                        │
   │   reason: "Expected to contain..." │
   │ }                                  │
   └────────────────────────────────────┘
```

### Why This Architecture?

**Separation of Concerns**: The orchestration layer (`runAssertions`) handles common tasks like variable rendering, transformation, and aggregation. Individual handlers focus purely on their specific evaluation logic. This means a new assertion type only needs to implement the actual check—all the infrastructure comes for free.

**Uniform Interface**: Every handler receives `AssertionParams` and returns `GradingResult`. This uniformity enables powerful features like:
- Generic weighting and aggregation
- Consistent error handling
- Easy addition of new assertion types
- Composition (assertion sets contain assertions)

**Lazy Complexity**: Simple assertions like `contains` are just string operations. Complex assertions like `similar` load embedding libraries. The architecture allows this—heavy dependencies are only loaded when their handlers are invoked.

## The Assertion Handler Pattern

### Why a Handler Map?

Promptfoo supports 50+ assertion types. How do you manage that complexity without a massive switch statement or deeply nested conditionals?

The answer is a **handler map**—a dictionary that maps assertion type strings to handler functions:

[📂 src/assertions/index.ts#L116](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/assertions/index.ts#L116)

```
ASSERTION_HANDLERS = {
  "contains":     handleContains,
  "equals":       handleEquals,
  "is-json":      handleIsJson,
  "similar":      handleSimilar,
  "llm-rubric":   handleLlmRubric,
  ...
}

Dispatch:
  type = assertion.type.replace("not-", "")  // Strip negation prefix
  handler = ASSERTION_HANDLERS[type]
  result = handler(params)
  if (wasNegated) invertResult(result)
```

**Why This Pattern Is Powerful**:

1. **Open/Closed Principle**: The system is open for extension (add new handlers) but closed for modification (core dispatch logic never changes). Adding `handle-my-custom-thing` doesn't touch `handleContains`.

2. **Isolation**: Each handler is a standalone function. Bugs don't propagate. You can understand `handleSimilar` without knowing anything about `handleJson`.

3. **Testability**: Mock `AssertionParams`, call handler directly, check `GradingResult`. No need to set up the entire evaluation infrastructure.

4. **Lazy Loading**: Some handlers have heavy dependencies. The `meteor` NLP metric requires loading a large library. By using a handler map with functions, we can dynamically import dependencies only when needed:

   ```typescript
   "meteor": async (params) => {
     const { computeMeteor } = await import('./nlp/meteor');  // Only loaded if meteor is used
     return computeMeteor(params);
   }
   ```

**Alternative Considered: Class Hierarchy**

One alternative would be an object-oriented approach with a base `Assertion` class and subclasses like `ContainsAssertion`, `SimilarAssertion`, etc. This was rejected because:

- JavaScript/TypeScript functions are simpler and more idiomatic
- No benefit from polymorphism—each handler has completely different logic
- Configuration comes from YAML, not from instantiating classes
- Testing pure functions is simpler than testing objects

### The AssertionParams Contract

Every handler receives a standard set of parameters. Not all handlers use all parameters, but the contract is consistent:

| Parameter | Purpose | Who Uses It |
|-----------|---------|-------------|
| `assertion` | The full assertion configuration object | All handlers—they may need threshold, weight, custom config |
| `output` | Raw LLM output (can be string, object, array, null) | Handlers that need to work with structured outputs |
| `outputString` | Output coerced to string (for text comparisons) | Most text-based handlers (contains, regex, similar) |
| `renderedValue` | The assertion's `value` after template substitution | Most handlers—this is what they compare against |
| `inverse` | True if this is a `not-` assertion | Handled at dispatch level, but available to handlers |
| `test` | The full test case configuration | Handlers that need test context (vars, options) |
| `provider` | The LLM provider that generated the output | Model-graded assertions (for chain-of-thought grading) |
| `providerResponse` | Full response with metadata, tokens, latency | Performance assertions (latency, cost, token counting) |
| `latencyMs` | How long the API call took | `latency` assertion |
| `cost` | Estimated cost of the API call | `cost` assertion |
| `providerCallContext` | Context for making grader API calls | Model-graded assertions (to call the grader LLM) |

**Why Pass Everything?**: It's simpler than having each handler request what it needs. The overhead is negligible (it's just passing references), and it means handlers have access to anything they might need without modifying the dispatch logic.

### The GradingResult Contract

Every handler must return a `GradingResult`:

```typescript
interface GradingResult {
  pass: boolean;          // Binary: did this assertion pass?
  score: number;          // Continuous: how well did it pass? (typically 0-1)
  reason: string;         // Human-readable explanation
  assertion?: Assertion;  // The assertion that was evaluated (for debugging)
  tokensUsed?: TokenUsage; // Tokens consumed (for model-graded assertions)
  namedScores?: Record<string, number>; // Additional named metrics
  componentResults?: GradingResult[]; // For assertion sets—results of inner assertions
}
```

This structure is carefully designed:

**`pass` vs `score`**: These serve fundamentally different purposes. `pass` is for automation—"should this test fail CI?" `score` is for analysis—"how much better is prompt A than prompt B?" Consider:

- An assertion might pass (score ≥ threshold) but you still want to track that it scored 0.75, not 1.0
- When comparing providers, you want to see score distributions, not just pass rates
- Weight-0 assertions intentionally don't affect `pass` but still contribute `score` for metrics

**`reason`**: Crucial for debugging. When an assertion fails, you need to understand why. Good handlers provide informative reasons:
- Bad: "Assertion failed"
- Good: "Expected output to contain 'hello', but output was 'Hi there! How can I help?'"

**`tokensUsed`**: Model-graded assertions consume tokens. Tracking this separately from generation tokens helps users understand cost breakdown—are you spending more on generation or evaluation?

**`componentResults`**: For assertion sets and custom scoring. Allows drilling down from "test failed" to "which specific assertion failed and why?"

### The Inverse Pattern: Elegant Negation

Rather than implementing 50+ assertion types AND 50+ negated versions, promptfoo uses a single `inverse` flag:

```
User writes: type: not-contains

Dispatch logic:
  1. Detect "not-" prefix
  2. Strip it: type = "contains"
  3. Set inverse = true
  4. Call handleContains(params) with inverse = true
  5. Handler knows to flip its logic
```

Most handlers implement this as:

```typescript
const pass = outputString.includes(value) !== inverse;
//                                        ^^^^^^^^^^
// XOR: if inverse is true, flip the result
```

This pattern:
- Halves the number of handlers needed
- Ensures positive and negative assertions are always consistent
- Makes it trivial to add negation to new assertion types

**Edge Cases**: Some assertions don't support negation logically (what would `not-latency` mean?). The dispatcher validates this and rejects invalid negated types at configuration time.

## Assertion Categories

Understanding when to use each category is as important as understanding how they work.

### Category 1: Deterministic Text Matching

**When to Use**: You have specific text that must (or must not) appear in the output. These are your first line of defense—fast, free, and deterministic.

[📂 src/assertions/contains.ts](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/assertions/contains.ts)

| Assertion | Logic | Example Use Case |
|-----------|-------|------------------|
| `contains` | Output includes substring | Response mentions required term ("invoice", "error code") |
| `not-contains` | Output excludes substring | Response doesn't leak PII, competitor names, profanity |
| `contains-all` | Output includes ALL listed substrings | Response covers multiple required topics |
| `contains-any` | Output includes AT LEAST ONE listed substring | Response uses one of several valid phrasings |
| `icontains` | Case-insensitive contains | Flexible matching where case doesn't matter |
| `equals` | Exact match (trimmed) | Fixed responses like "yes" or "no" |
| `starts-with` | Output begins with string | Format validation ("Dear Customer,") |
| `regex` | Regular expression match | Complex patterns (email format, phone numbers) |

**Design Decisions**:

Why `contains` instead of `includes` or `has`? Naming follows common testing conventions. `contains` is immediately understood.

Why separate `icontains` instead of a `case_insensitive` option? Simplicity. One assertion type = one behavior. No ambiguity.

Why `contains-all` and `contains-any` instead of just `contains` with array logic? Explicit is better than implicit. Reading `type: contains-any` immediately tells you the logic.

**How It Works Internally**:

```typescript
export const handleContains = ({
  renderedValue,
  outputString,
  inverse,
}: AssertionParams): GradingResult => {
  // Simple substring check
  const found = outputString.includes(String(renderedValue));
  const pass = found !== inverse;  // XOR for negation
  
  return {
    pass,
    score: pass ? 1 : 0,  // Binary: all-or-nothing for text matching
    reason: pass
      ? 'Assertion passed'
      : `Expected output to ${inverse ? 'not ' : ''}contain "${renderedValue}"`,
    assertion,
  };
};
```

**Why Binary Scoring?** For text matching, partial credit doesn't make sense. Either "error code 404" appears or it doesn't. Some tools try fuzzy matching with partial scores, but this adds complexity without benefit—use `similar` if you want similarity.

**Practical Considerations**:

- **Substring gotchas**: `contains: "no"` matches "I don't know" and "This is the north pole". Be specific.
- **Case sensitivity matters**: "Error" ≠ "error" for `contains`. Use `icontains` when case varies.
- **Regex power**: When patterns get complex (`\d{3}-\d{4}` for phone numbers), `regex` is cleaner than multiple `contains`.

### Category 2: Structural Validation

**When to Use**: You need the output to conform to a specific format—JSON, SQL, HTML, or custom structure. Critical for LLM applications that feed outputs into downstream systems.

[📂 src/assertions/json.ts](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/assertions/json.ts)

| Assertion | What It Validates |
|-----------|-------------------|
| `is-json` | Valid JSON syntax |
| `is-json` (with value) | JSON conforming to JSON Schema |
| `contains-json` | Valid JSON somewhere in the output (even with surrounding text) |
| `is-sql` | Valid SQL syntax |
| `is-html` | Valid HTML structure |
| `is-xml` | Valid XML structure |
| `is-valid-openai-tools-call` | OpenAI function calling format |
| `is-valid-openai-function-call` | Legacy function call format |

**Why Structural Validation Matters**:

LLMs are increasingly used to generate structured outputs—JSON for APIs, SQL for databases, code for execution. A response that's semantically correct but syntactically broken is useless:

```
Good LLM output:
  {"name": "John", "age": 30}
  
Bad LLM output (common failure mode):
  Sure! Here's the JSON:
  {"name": "John", "age": 30}
```

The second output is valid JSON... if you extract it. But `JSON.parse()` fails because of the preamble. Use `contains-json` for this case.

**JSON Schema Validation Deep Dive**:

When you provide a `value` to `is-json`, it's interpreted as a JSON Schema:

```yaml
assert:
  - type: is-json
    value:
      type: object
      required: [name, age]
      properties:
        name: 
          type: string
          minLength: 1
        age: 
          type: number
          minimum: 0
          maximum: 150
```

This validates:
1. Output is valid JSON
2. It's an object (not array, string, etc.)
3. It has `name` and `age` properties
4. `name` is a non-empty string
5. `age` is a number between 0 and 150

**Why JSON Schema?** It's a well-documented standard. Developers know it. Tools exist. Promptfoo doesn't invent its own format.

**Under the Hood**:

```typescript
export function handleIsJson({ outputString, renderedValue }): GradingResult {
  // Step 1: Can we parse it?
  let parsedJson;
  try {
    parsedJson = JSON.parse(outputString);
  } catch (e) {
    return {
      pass: false,
      score: 0,
      reason: `Invalid JSON: ${e.message}`,
    };
  }

  // Step 2: If schema provided, validate against it
  if (renderedValue) {
    const ajv = getAjv();  // Singleton Ajv instance
    const validate = ajv.compile(renderedValue);
    const valid = validate(parsedJson);
    
    if (!valid) {
      const errors = ajv.errorsText(validate.errors);
      return {
        pass: false,
        score: 0,
        reason: `JSON doesn't match schema: ${errors}`,
      };
    }
  }

  return { pass: true, score: 1, reason: 'Assertion passed' };
}
```

**contains-json: The Forgiving Variant**:

LLMs love to add explanatory text around structured output:

```
Here's the user data you requested:
{"name": "John", "age": 30}
Hope this helps!
```

`is-json` fails on this. `contains-json` succeeds by:
1. Scanning the output for `{` or `[`
2. Attempting to parse from each position
3. Passing if any valid JSON is found

**Trade-off**: `contains-json` is more forgiving but less precise. If you specifically need clean JSON output (for programmatic consumption), use `is-json` and fix your prompts if they add preamble.

### Category 3: Semantic Similarity

**When to Use**: You want to check if the output conveys the same meaning as a reference, even if the words differ. Essential for open-ended responses where many phrasings are valid.

[📂 src/assertions/similar.ts](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/assertions/similar.ts)

| Assertion | Metric | Characteristics |
|-----------|--------|-----------------|
| `similar` | Cosine (default) | Direction-based, normalized, most common |
| `similar:cosine` | Cosine | Explicit variant |
| `similar:dot` | Dot product | Magnitude-sensitive, favors longer texts |
| `similar:euclidean` | Euclidean distance | Distance-based, sensitive to all dimensions |

**How Semantic Similarity Works**:

The process involves three steps:

```
1. EMBED: Convert text to vectors
   ┌─────────────────────┐       ┌─────────────────────┐
   │ Expected:           │       │ Actual:             │
   │ "I love dogs"       │       │ "Dogs are great     │
   │                     │       │  pets"              │
   └──────────┬──────────┘       └──────────┬──────────┘
              │                             │
              ▼                             ▼
   ┌─────────────────────┐       ┌─────────────────────┐
   │ Embedding Model     │       │ Embedding Model     │
   │ (text-embedding-    │       │ (same model)        │
   │  3-small)           │       │                     │
   └──────────┬──────────┘       └──────────┬──────────┘
              │                             │
              ▼                             ▼
   [0.23, -0.45, 0.67, ...]    [0.19, -0.42, 0.71, ...]
   
2. COMPARE: Compute similarity metric
   
   Cosine: angle between vectors
   ┌───────────────────────────────────────────────┐
   │ cos(θ) = (A·B) / (||A|| × ||B||)              │
   │                                               │
   │ Result: 0.89 (high similarity)                │
   └───────────────────────────────────────────────┘
   
3. THRESHOLD: Determine pass/fail
   
   threshold: 0.75 (default)
   0.89 ≥ 0.75 → pass: true, score: 0.89
```

**Understanding the Metrics**:

**Cosine Similarity**: Measures the angle between vectors, ignoring magnitude. Two vectors pointing in the same direction have similarity 1.0, regardless of length. This means:
- "I love dogs" and "I absolutely positively love dogs very much" might score similarly
- Good for semantic comparison where verbosity shouldn't matter

**Dot Product**: Considers both angle AND magnitude. Longer texts produce larger vectors (more words = more signal), which affects the score:
- "I love dogs" vs "I love dogs and cats and birds and fish" → magnitude matters
- Use when length is semantically meaningful

**Euclidean Distance**: Absolute distance in vector space. Converted to similarity via `1 / (1 + distance)`:
- More sensitive to differences in individual dimensions
- Use when precision matters more than general direction

**Why the Default Threshold is 0.75**:

This was chosen empirically. Some observations:
- 0.9+ : Near-paraphrases, very similar meaning
- 0.8-0.9: Same topic, similar sentiment, minor variations
- 0.7-0.8: Related content, might have different emphasis
- Below 0.7: Often different topics or opposite sentiments

0.75 is a reasonable default that catches most semantic matches without being overly strict. Adjust based on your use case:
- Strict factual matching: raise to 0.85+
- Flexible topic matching: lower to 0.65

**Cost Considerations**:

Unlike text matching, similarity assertions make API calls to embedding models. Each comparison requires embedding both the expected text and the actual output. With thousands of test cases, this adds up.

**Optimization**: If you're comparing the same expected text against many outputs, promptfoo caches the expected embedding. Only the outputs need fresh embedding calls.

**Choosing an Embedding Model**:

The default is `text-embedding-3-small` from OpenAI. It's:
- Fast: Low latency, high throughput
- Cheap: Lowest cost per token among OpenAI embeddings
- Good enough: For most similarity comparisons, you don't need the large model

Override if needed:
```yaml
defaultTest:
  options:
    provider:
      embedding:
        id: openai:embedding:text-embedding-3-large
```

### Category 4: Model-Graded Assertions

**When to Use**: The quality you're measuring is subjective or complex—helpfulness, accuracy, professionalism, completeness. When you can describe good quality in English but can't reduce it to a simple check.

[📂 src/assertions/llmRubric.ts](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/assertions/llmRubric.ts)

| Assertion | Specialized For |
|-----------|-----------------|
| `llm-rubric` | General purpose—your criteria, your words |
| `factuality` | Does the response match ground truth facts? |
| `answer-relevance` | Does the response address the question asked? |
| `context-faithfulness` | Is the response grounded in provided context (RAG)? |
| `context-recall` | Does the response use all relevant context? |
| `context-relevance` | Is the retrieved context relevant to the query? |
| `model-graded-closedqa` | Is the answer correct for this question? |
| `g-eval` | Research-backed multi-step evaluation |

**The LLM-as-Judge Paradigm**:

Human evaluation doesn't scale. You can't manually review 10,000 responses. But humans are excellent at articulating quality criteria. The insight: let humans write rubrics, let LLMs apply them at scale.

```
Human expertise:                    LLM execution:
┌────────────────────────┐         ┌────────────────────────┐
│ "A good customer       │         │ For each response:     │
│ service response is    │   →     │ - Read the rubric      │
│ empathetic, addresses  │         │ - Evaluate the output  │
│ the concern, and       │         │ - Score 0-10           │
│ offers next steps"     │         │ - Explain the score    │
└────────────────────────┘         └────────────────────────┘
```

**How llm-rubric Works Internally**:

```
1. User provides rubric:
   type: llm-rubric
   value: "The response should be helpful, accurate, and professional"

2. Promptfoo constructs grading prompt:
   ┌────────────────────────────────────────────────────────────┐
   │ You are an expert evaluator assessing AI responses.        │
   │                                                            │
   │ ORIGINAL PROMPT:                                           │
   │ [the prompt that was sent to the LLM being tested]         │
   │                                                            │
   │ AI RESPONSE:                                               │
   │ [the output being evaluated]                               │
   │                                                            │
   │ EVALUATION CRITERIA:                                       │
   │ The response should be helpful, accurate, and professional │
   │                                                            │
   │ Does the response meet the criteria?                       │
   │ Respond with a score from 1-10 and explanation.            │
   └────────────────────────────────────────────────────────────┘

3. Grader LLM (default: gpt-4o-mini) evaluates:
   ┌────────────────────────────────────────────────────────────┐
   │ Score: 8                                                   │
   │                                                            │
   │ Explanation: The response addresses the user's question    │
   │ directly and provides accurate information. It maintains   │
   │ a professional tone throughout. However, it could be more  │
   │ concise—the third paragraph adds little value.             │
   └────────────────────────────────────────────────────────────┘

4. Parse response → GradingResult:
   { pass: true, score: 0.8, reason: "Score: 8..." }
```

**Writing Effective Rubrics**:

Bad rubric: "Be good"
- Too vague. What does "good" mean?

Better rubric: "The response should be helpful"
- Slightly better, but still subjective

Good rubric: "The response should (1) directly answer the user's question, (2) provide at least one actionable step, and (3) maintain a professional tone"
- Specific criteria
- Enumerated points for the grader to check
- Clear, observable requirements

**Why gpt-4o-mini is the Default Grader**:

Model choice involves trade-offs:

| Model | Speed | Cost | Quality |
|-------|-------|------|---------|
| gpt-4o-mini | Fast | $0.15/M input | Good enough for most rubrics |
| gpt-4o | Medium | $2.50/M input | Better nuance, catches edge cases |
| gpt-4 | Slow | $30/M input | Overkill for most evaluation |
| claude-3-haiku | Fast | $0.25/M input | Good alternative |

For evaluations with thousands of test cases, grading costs can exceed generation costs. gpt-4o-mini provides good quality at minimal cost.

**When to Use a More Sophisticated Grader**:

- Complex rubrics with multiple interacting criteria
- Nuanced quality distinctions (is this response "good" or "great"?)
- High-stakes evaluations where accuracy matters more than cost
- Rubrics involving domain expertise (medical, legal)

Configure globally:
```yaml
defaultTest:
  options:
    provider: openai:gpt-4o
```

Or per-assertion:
```yaml
assert:
  - type: llm-rubric
    value: "Complex medical criteria..."
    provider: openai:gpt-4o  # Use better model for this one
```

### Category 5: Custom Code Assertions

**When to Use**: Built-in assertions don't capture your requirement. You need to query a database, call an API, run complex calculations, or implement custom logic.

[📂 src/assertions/javascript.ts](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/assertions/javascript.ts)

| Assertion | Language/Method |
|-----------|-----------------|
| `javascript` | JavaScript code (inline or file) |
| `python` | Python script (file-based) |
| `ruby` | Ruby script (file-based) |
| `webhook` | HTTP POST to your endpoint |

**Inline JavaScript**:

For simple checks, embed JavaScript directly:

```yaml
assert:
  - type: javascript
    value: |
      // Check response length is reasonable
      output.length > 50 && output.length < 500
```

**How It Works**:

The JavaScript assertion is surprisingly sophisticated:

```typescript
export const handleJavascript = async (params: AssertionParams): Promise<GradingResult> => {
  const { renderedValue, output, assertionValueContext } = params;
  
  // For single-line code, wrap in return statement
  // "output.length > 50" becomes "return output.length > 50"
  const functionBody = buildFunctionBody(renderedValue);
  
  // Create a sandboxed function with access to output and context
  const customFunction = new Function(
    'output',    // The LLM's response
    'context',   // Rich context object
    'process',   // Limited process shim (env vars)
    functionBody
  );
  
  // Execute the custom code
  const result = await customFunction(output, assertionValueContext, processShim);
  
  // Interpret the result flexibly
  if (typeof result === 'boolean') {
    return { pass: result, score: result ? 1 : 0, reason: '...' };
  } else if (typeof result === 'number') {
    // Numbers are treated as scores
    // Positive = pass, score = the number (normalized to 0-1 if > 1)
    return { pass: result > 0, score: normalizeScore(result), reason: '...' };
  } else if (typeof result === 'object') {
    // Full GradingResult object
    return result;
  }
};
```

**The Context Object**:

Custom code receives rich context:

```javascript
context = {
  prompt: "The original prompt sent to the LLM",
  vars: {
    user_question: "How do I reset my password?",
    user_tier: "premium",
    // ... all test variables
  },
  test: {
    description: "Password reset for premium user",
    // ... full test case
  },
  provider: {
    id: "openai:gpt-4o",
    // ... provider info
  },
  providerResponse: {
    output: "To reset your password...",
    tokenUsage: { input: 50, output: 120 },
    latencyMs: 450,
    // ... full response metadata
  }
}
```

This enables powerful patterns:

```javascript
// Check response against user tier
if (context.vars.user_tier === 'premium') {
  // Premium users should get more detailed responses
  return output.length > 200;
} else {
  return output.length > 50;
}
```

**File-Based Custom Assertions**:

For complex logic, use external files:

```yaml
assert:
  - type: javascript
    value: file://assertions/validate_response.js
  
  - type: python
    value: file://assertions/check_database.py
```

Python assertions receive data via stdin (JSON) and return results via stdout.

**Webhook Assertions**:

For integration with external systems:

```yaml
assert:
  - type: webhook
    value: https://my-validation-service.com/check
```

Promptfoo POSTs:
```json
{
  "output": "LLM response",
  "prompt": "Original prompt",
  "vars": { ... },
  "test": { ... }
}
```

Your endpoint returns:
```json
{
  "pass": true,
  "score": 0.85,
  "reason": "All checks passed"
}
```

**Use Cases for Custom Assertions**:

1. **Database validation**: Check that extracted entities match your database
2. **Business logic**: Validate calculations, rule application
3. **External APIs**: Verify against ground truth from other systems
4. **Complex parsing**: Extract and validate structured data
5. **Integration testing**: Ensure LLM output works with downstream systems

### Category 6: Performance Assertions

**When to Use**: Quality isn't just about content—it's about latency, cost, and resource usage. Production systems have SLAs and budgets.

[📂 src/assertions/latency.ts](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/assertions/latency.ts)

| Assertion | What It Checks |
|-----------|----------------|
| `latency` | Response time in milliseconds |
| `cost` | Estimated cost per response |
| `perplexity` | Model confidence (requires logprobs) |
| `perplexity-score` | Normalized perplexity |

**Why Performance Assertions Matter**:

A response that's perfect but takes 30 seconds is useless for a chatbot. A response that's accurate but costs $2 per query will bankrupt you at scale.

```yaml
assert:
  - type: latency
    threshold: 3000  # Must respond in under 3 seconds
  
  - type: cost
    threshold: 0.01  # Must cost less than $0.01 per response
```

**How Cost Is Calculated**:

Promptfoo estimates cost from token usage and known pricing:

```
cost = (input_tokens × input_price) + (output_tokens × output_price)

For gpt-4o:
  - Input: $2.50 per million tokens
  - Output: $10.00 per million tokens

100 input tokens + 200 output tokens:
  = (100 × $0.0000025) + (200 × $0.00001)
  = $0.00025 + $0.002
  = $0.00225 per request
```

**Perplexity: Model Confidence**:

Perplexity measures how "surprised" the model is by its own output. Lower perplexity = more confident.

```yaml
assert:
  - type: perplexity
    threshold: 10  # Perplexity must be under 10
```

Use cases:
- Detecting when the model is guessing vs. confident
- Identifying hallucination-prone responses (often high perplexity)
- Quality filtering in batch processing

**Caveat**: Perplexity requires `logprobs` to be enabled in the API request. Not all providers support this.

### Category 7: NLP Metrics

**When to Use**: Classic machine translation and summarization metrics. Useful when you have reference outputs and want reproducible, well-understood scores.

| Assertion | What It Measures |
|-----------|------------------|
| `bleu` | Bilingual Evaluation Understudy (translation quality) |
| `rouge-n` | Recall-Oriented Understudy for Gisting Evaluation |
| `levenshtein` | Edit distance (character-level similarity) |
| `meteor` | Metric for Evaluation of Translation with Explicit Ordering |

**When NLP Metrics Work Well**:

1. **Translation**: Comparing output to professional reference translations
2. **Summarization**: Measuring overlap with reference summaries
3. **Fixed outputs**: Tasks with known correct answers
4. **Regression testing**: Detecting drift from baseline behavior

**When NLP Metrics Fail**:

These metrics compare surface-level text features. They fail when:
- Many valid phrasings exist ("yes" vs "absolutely" vs "correct")
- Semantic equivalence matters more than word overlap
- The task is creative or open-ended
- Style variation is acceptable

**BLEU Score Explained**:

BLEU compares n-grams (sequences of n words) between output and reference:

```
Reference: "The quick brown fox jumps over the lazy dog"
Output:    "A fast brown fox leaps over the sleepy dog"

1-grams in common: brown, fox, over, the, dog (5 matches)
2-grams in common: brown fox, over the (2 matches)
3-grams in common: none
4-grams in common: none

BLEU score combines these with brevity penalty → ~0.4
```

**Trade-offs**: BLEU is well-understood and reproducible, but it misses synonyms ("quick" vs "fast") and penalizes valid paraphrases.

### Category 8: Trace Assertions

**When to Use**: Your LLM application is instrumented with OpenTelemetry. You want to validate not just the output, but the execution path—how many retrievals happened, how long embedding took, were there errors?

[📂 src/assertions/traceSpanCount.ts](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/assertions/traceSpanCount.ts)

| Assertion | What It Validates |
|-----------|-------------------|
| `trace-span-count` | Number of spans with given name |
| `trace-span-duration` | Duration of specific spans |
| `trace-error-spans` | Presence/absence of error spans |

**Use Case: RAG Pipeline Validation**:

```yaml
tests:
  - vars:
      query: "What is the company's refund policy?"
    assert:
      # Content checks
      - type: contains
        value: "30 days"
      
      # Trace checks
      - type: trace-span-count
        value:
          name: "retrieval"
          min: 1  # At least one retrieval happened
          max: 3  # Not too many (efficiency)
      
      - type: trace-span-duration
        value:
          name: "embedding"
          maxDuration: 200  # Embedding should be fast
      
      - type: trace-error-spans
        value:
          maxCount: 0  # No errors in the pipeline
```

**Why Trace Assertions?**

LLM applications are increasingly complex pipelines—retrieval, reranking, generation, postprocessing. The output might be correct, but:
- Did retrieval actually happen, or did the model hallucinate?
- Was the embedding step unreasonably slow?
- Did a retry succeed after an initial failure?

Trace assertions let you validate internal behavior, not just external output.

## The Scoring System

The scoring system is how promptfoo aggregates multiple assertions into a single test result. Understanding it deeply is crucial for designing effective test suites.

### Why Two Concepts: Pass and Score?

This is a common point of confusion. Why does `GradingResult` have both `pass` (boolean) and `score` (number)?

**They serve fundamentally different purposes**:

**`pass` is for automation**: "Should this test fail CI?" It's binary—either the test passes or it doesn't. This is what CI systems, gates, and go/no-go decisions use.

**`score` is for analysis**: "How good was this response?" It's continuous—you want to track whether quality is improving over time, compare prompts, identify the best-performing providers.

**The Disconnect**:

Consider a similarity assertion with threshold 0.80:

```
Response A: score 0.85 → pass: true
Response B: score 0.95 → pass: true
```

Both pass, but Response B is clearly better. If you only tracked `pass`, you'd miss this signal.

Consider a `llm-rubric` assertion:

```
Response C: score 0.75 → pass: false (below implicit threshold)
Response D: score 0.72 → pass: false
```

Both fail, but Response C was closer. When debugging, this matters.

**Practical Implication**: Always examine score distributions, not just pass rates. A suite with 90% pass rate but average score 0.65 is different from one with 90% pass rate and average score 0.92.

### Weighted Average Scoring

When a test has multiple assertions, the aggregate score is a weighted average:

[📂 src/assertions/assertionsResult.ts](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/assertions/assertionsResult.ts)

```
Formula:
  aggregate_score = Σ(score_i × weight_i) / Σ(weight_i)

Example:
  - contains "hello": score=1.0, weight=1
  - llm-rubric "helpful": score=0.8, weight=2
  - similar: score=0.6, weight=1
  
  aggregate = (1.0×1 + 0.8×2 + 0.6×1) / (1+2+1)
            = (1.0 + 1.6 + 0.6) / 4
            = 0.8
```

**Why Weighted Average?**

Not all assertions are equally important. A hard requirement ("must mention refund policy") should outweigh a soft preference ("should be concise"). Weights express this priority.

**Default Weight Is 1**: Every assertion starts with equal weight. Explicitly set weight when you want to adjust:

```yaml
assert:
  - type: contains
    value: "refund policy"
    weight: 3  # Critical requirement
  
  - type: llm-rubric
    value: "professional tone"
    weight: 1  # Nice to have
```

**Weight 0: Metrics Without Consequences**:

A special case—assertions with `weight: 0` are evaluated and recorded but don't affect the aggregate score or pass/fail:

```yaml
assert:
  - type: latency
    threshold: 5000
    weight: 0  # Track latency, but don't fail the test if it's slow
```

Use cases:
- Gathering metrics for analysis without affecting test outcomes
- Soft constraints that shouldn't block deployment
- Experimental assertions you're not confident in yet

### Threshold-Based Pass/Fail

There are two different thresholds—they serve different purposes and are often confused:

**Test-Level Threshold**: Applied to the aggregate score

```yaml
tests:
  - vars: { ... }
    threshold: 0.75  # Test passes if aggregate score ≥ 0.75
    assert:
      - type: llm-rubric
        value: "helpful"
      - type: llm-rubric
        value: "accurate"
```

This means: "I don't need every assertion to pass. If the aggregate quality is above 0.75, that's good enough."

Use case: Soft quality requirements where perfect isn't necessary.

**Assertion-Level Threshold**: Applied to specific assertions

```yaml
assert:
  - type: similar
    value: "Expected response text"
    threshold: 0.85  # This specific check needs 85% similarity
```

This means: "For this similarity check, I need at least 85% match."

**Default Pass/Fail Behavior** (when no test-level threshold):

Without an explicit threshold, the test passes only if all assertions pass:

```
Assertion 1: pass=true
Assertion 2: pass=true
Assertion 3: pass=false
→ Test: pass=false
```

This is the "strict" mode—any failure fails the test.

**With Threshold** (lenient mode):

```
threshold: 0.7
Assertion 1: score=1.0
Assertion 2: score=0.8
Assertion 3: score=0.5
→ aggregate = 0.77
→ 0.77 ≥ 0.7 → pass=true
```

Even though assertion 3 had a low score, the aggregate exceeds the threshold.

### Named Scores: Custom Metrics

Sometimes the aggregate score isn't enough. You want to track specific quality dimensions:

```yaml
assert:
  - type: llm-rubric
    value: "factually accurate"
    metric: accuracy
  
  - type: llm-rubric
    value: "clear and easy to understand"
    metric: clarity
  
  - type: llm-rubric
    value: "appropriate length—not too long or short"
    metric: conciseness
  
  - type: latency
    threshold: 3000
    metric: performance
```

Results include:

```json
{
  "score": 0.82,
  "namedScores": {
    "accuracy": 0.95,
    "clarity": 0.80,
    "conciseness": 0.70,
    "performance": 1.0
  }
}
```

**Why Named Scores Matter**:

1. **Diagnosis**: Overall score dropped. Which dimension? Named scores tell you.
2. **Comparison**: Prompt A is better at accuracy, Prompt B at clarity. Named scores surface this.
3. **Optimization**: You can't improve what you don't measure. Named scores enable targeted optimization.

### Custom Scoring Functions

When weighted average doesn't fit your needs, you can define custom scoring logic:

```yaml
tests:
  - vars: { ... }
    assertScoringFunction: file://scorer.js:customScore
    assert:
      - type: contains
        value: "answer"
      - type: llm-rubric
        value: "complete"
```

```javascript
// scorer.js
module.exports.customScore = async (namedScores, context) => {
  const { componentResults, threshold, traceId, test, provider, providerResponse } = context;
  
  // Example: ALL assertions must pass (no weighted average)
  const allPassed = componentResults.every(result => result.pass);
  
  // Example: Apply different logic based on test metadata
  if (test.metadata?.strict) {
    return {
      pass: allPassed,
      score: allPassed ? 1.0 : 0.0,
      reason: allPassed ? 'All checks passed' : 'At least one check failed',
    };
  }
  
  // Example: Geometric mean instead of arithmetic mean
  const scores = componentResults.map(r => r.score);
  const geometricMean = Math.pow(scores.reduce((a, b) => a * b, 1), 1 / scores.length);
  
  return {
    pass: geometricMean >= (threshold || 0.5),
    score: geometricMean,
    reason: `Geometric mean: ${geometricMean.toFixed(3)}`,
  };
};
```

**When to Use Custom Scoring**:

- **"All must pass" logic**: Weighted average allows one assertion to compensate for another. Sometimes that's wrong.
- **Non-linear combinations**: Geometric mean, max, min, custom formulas
- **Conditional logic**: Different scoring based on test properties
- **External validation**: Call external systems during scoring

## Assertion Sets: Grouping with Logic

### The Problem: Boolean Logic in Evaluation

Individual assertions are implicitly AND-ed together: all must pass (or aggregate score must exceed threshold). But sometimes you need OR logic:

"The response should contain 'yes' OR 'affirmative' OR 'correct'"

Any of these is acceptable. How do you express this?

### How Assert-Sets Work

An assertion set is an assertion that contains other assertions, with its own threshold logic:

```yaml
assert:
  - type: assert-set
    threshold: 0.33  # At least 1 of 3 must pass (33%)
    assert:
      - type: contains
        value: "yes"
      - type: contains
        value: "affirmative"
      - type: contains
        value: "correct"
```

**The threshold interpretation**:

- `threshold: 1.0` = ALL inner assertions must pass (AND logic)
- `threshold: 0.5` = AT LEAST HALF must pass
- `threshold: 0.33` = AT LEAST ONE-THIRD must pass
- For "at least one": `threshold = 1 / count`

**Nesting for Complex Logic**:

Assert-sets can be nested to express complex boolean combinations:

```yaml
# Expression: (A OR B) AND (C OR D)
assert:
  - type: assert-set
    threshold: 1.0  # All inner sets must pass (AND)
    assert:
      - type: assert-set
        threshold: 0.5  # At least one must pass (OR)
        assert:
          - type: contains
            value: "hello"   # A
          - type: contains
            value: "hi"      # B
      - type: assert-set
        threshold: 0.5  # At least one must pass (OR)
        assert:
          - type: contains
            value: "goodbye"  # C
          - type: contains
            value: "bye"      # D
```

### Practical Patterns

**Multiple valid response formats**:

```yaml
- type: assert-set
  threshold: 0.5
  assert:
    - type: is-json
    - type: is-xml
```

"Output can be JSON or XML—either is acceptable."

**Graceful degradation**:

```yaml
- type: assert-set
  threshold: 0.66  # 2 of 3 must pass
  assert:
    - type: llm-rubric
      value: "accurate"
    - type: llm-rubric
      value: "helpful"
    - type: llm-rubric
      value: "concise"
```

"We prefer all three qualities, but if one is weak, that's okay."

**Hard requirement + flexible options**:

```yaml
assert:
  - type: contains
    value: "safety warning"  # This MUST be present
  
  - type: assert-set
    threshold: 0.5  # At least one of these
    assert:
      - type: contains
        value: "doctor"
      - type: contains
        value: "medical professional"
      - type: contains
        value: "healthcare provider"
```

"Must mention safety warning AND must mention a medical professional (using any of these terms)."

## Comparison Assertions

### The Unique Challenge of Relative Evaluation

Most assertions evaluate each response independently: "Is this response good?" Comparison assertions ask a different question: "Which response is best?"

This is fundamentally different:
- **Absolute evaluation**: Response gets score based on fixed criteria
- **Relative evaluation**: Responses are ranked against each other

[📂 src/assertions/index.ts#L610](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/assertions/index.ts#L610)

### `select-best`: LLM-Based Comparison

```yaml
providers:
  - openai:gpt-4o
  - anthropic:claude-3-sonnet
  - openai:gpt-4o-mini

tests:
  - vars:
      question: "Explain quantum entanglement"
    assert:
      - type: select-best
        value: "Which response best explains the concept clearly and accurately?"
```

**How It Works**:

```
1. All providers generate responses to the same prompt
   
   GPT-4o:           "Quantum entanglement is a phenomenon..."
   Claude Sonnet:    "Entanglement occurs when particles..."
   GPT-4o-mini:      "It's like having two coins..."

2. Promptfoo collects all responses

3. A grader LLM (typically gpt-4o-mini) sees all responses together:
   ┌─────────────────────────────────────────────────────────────┐
   │ You are comparing these AI responses to the question:       │
   │ "Explain quantum entanglement"                              │
   │                                                             │
   │ Response A: "Quantum entanglement is a phenomenon..."       │
   │ Response B: "Entanglement occurs when particles..."         │
   │ Response C: "It's like having two coins..."                 │
   │                                                             │
   │ Criteria: Which response best explains the concept clearly  │
   │ and accurately?                                             │
   │                                                             │
   │ Select the best response and explain why.                   │
   └─────────────────────────────────────────────────────────────┘

4. Grader responds: "Response A is best because..."

5. Results:
   GPT-4o:         pass: true, score: 1.0 (winner)
   Claude Sonnet:  pass: false, score: 0.0
   GPT-4o-mini:    pass: false, score: 0.0
```

**Use Cases**:

- **Model selection**: Which model performs best on your tasks?
- **Prompt comparison**: Same model, different prompts—which wins?
- **A/B testing**: Evaluating prompt changes head-to-head
- **Human-like judgment**: When you want holistic comparison, not metric aggregation

**Limitations**:

- **Expensive**: Requires collecting all responses before grading
- **Position bias**: LLMs tend to favor responses presented first or last
- **Not deterministic**: Different runs might pick different winners
- **Ties are hard**: Three equally good responses? The grader picks one anyway.

### `max-score`: Automated Winner

```yaml
providers:
  - openai:gpt-4o
  - anthropic:claude-3-sonnet

assert:
  - type: llm-rubric
    value: "Clear and accurate explanation"
  - type: latency
    threshold: 5000
  - type: max-score  # Highest combined score wins
```

**How It Works**:

1. Each response is scored independently using other assertions
2. The response with the highest aggregate score is marked as passing
3. Others are marked as failing

**Difference from `select-best`**:

| Aspect | `select-best` | `max-score` |
|--------|---------------|-------------|
| Comparison method | LLM judges holistically | Use existing assertion scores |
| Requires | All responses at once | Each scored independently |
| Best when | You want holistic judgment | You have good numeric metrics |
| Cost | Additional grader call | No extra cost |

## Value Rendering and Transformation

### Template Variables in Assertions

Assertion values can include template variables, rendered with Nunjucks:

```yaml
tests:
  - vars:
      expected_name: "John Smith"
      expected_department: "Engineering"
    assert:
      - type: contains
        value: "{{expected_name}}"
      - type: contains
        value: "{{expected_department}}"
```

**Why This Matters**:

Without template variables, you'd need to copy expected values between test data and assertions—error-prone and hard to maintain. With templates, assertions reference the same source of truth as your test inputs.

**Computed Expected Values**:

```yaml
tests:
  - vars:
      first_name: "John"
      last_name: "Smith"
    assert:
      - type: contains
        value: "{{first_name}} {{last_name}}"  # "John Smith"
```

**Data-Driven Testing**:

```yaml
tests: file://test_cases.csv

# test_cases.csv:
# input,expected_output
# "What is 2+2?","4"
# "What is 3*3?","9"

defaultTest:
  assert:
    - type: contains
      value: "{{expected_output}}"
```

Each row provides both input and expected output. Assertions are defined once, applied to all.

### Output Transformation

Before assertion evaluation, you can transform the LLM output:

```yaml
assert:
  - type: equals
    value: "success"
    transform: "JSON.parse(output).status"
```

**How It Works**:

```
Original output: '{"status": "success", "data": {...}}'

Transform applied: JSON.parse(output).status

Transformed output: "success"

Assertion compares: "success" equals "success" → pass
```

**Use Cases**:

1. **Extract from JSON**: `JSON.parse(output).field`
2. **Normalize text**: `output.toLowerCase().trim()`
3. **Parse structured data**: Custom extraction logic
4. **Clean up**: `output.replace(/\s+/g, ' ')` (collapse whitespace)

**Global vs Per-Assertion Transform**:

```yaml
# Global: applies to all assertions
defaultTest:
  options:
    transformOutput: "output.toLowerCase()"

# Per-assertion: only affects this one
assert:
  - type: contains
    value: "yes"
    transform: "output.split('\\n')[0]"  # Check only first line
```

### File-Based Assertion Values

For long expected texts or complex schemas:

```yaml
assert:
  - type: similar
    value: file://expected_responses/greeting.txt
  
  - type: is-json
    value: file://schemas/user_response.json
```

**Benefits**:

- Keep assertions readable (no 50-line inline strings)
- Share expected values across multiple tests
- Version control expected values separately
- Use your favorite editor for complex schemas

### Script-Based Assertion Values

For dynamic expected values that depend on context:

```yaml
assert:
  - type: contains
    value: file://assertions/get_expected.py:compute_expected
```

```python
# assertions/get_expected.py

def compute_expected(output, context):
    # Context includes vars, test, prompt, provider, etc.
    user_tier = context['vars']['user_tier']
    
    if user_tier == 'premium':
        return "exclusive premium content"
    else:
        return "standard content"
```

**Use Cases**:

- Look up expected values from database
- Compute expected values based on input
- Conditional expected values based on test context
- Integration with external systems

## Concurrency and Performance

### Assertion Concurrency

Assertions within a single test run concurrently, but with limits:

```typescript
const ASSERTIONS_MAX_CONCURRENCY = getEnvInt('PROMPTFOO_ASSERTIONS_MAX_CONCURRENCY', 3);
```

**Why Limit Concurrency?**

1. **Rate Limits**: Model-graded assertions call LLM APIs. Ten assertions hitting GPT-4o simultaneously might trigger rate limits.

2. **Resource Usage**: Each assertion might:
   - Make API calls (embedding, grading)
   - Load files
   - Run external scripts
   - Open network connections

3. **Debugging**: When things fail, lower concurrency means more predictable logs. "Assertion 1 ran, then 2, then 3" is easier to debug than "1, 2, 3 ran simultaneously and all failed."

**Tuning the Limit**:

```bash
# More concurrency for simple assertions
PROMPTFOO_ASSERTIONS_MAX_CONCURRENCY=10 promptfoo eval

# Less concurrency if hitting rate limits
PROMPTFOO_ASSERTIONS_MAX_CONCURRENCY=1 promptfoo eval
```

### Short-Circuit on Failure

For faster feedback during development:

```bash
PROMPTFOO_SHORT_CIRCUIT_TEST_FAILURES=true promptfoo eval
```

When enabled:
- First failing assertion stops evaluation
- Remaining assertions aren't run
- You get faster feedback on obvious failures

**Trade-off**:

| With Short-Circuit | Without Short-Circuit |
|--------------------|-----------------------|
| Faster feedback | Complete picture |
| Lower cost (fewer API calls) | Higher cost |
| See only first failure | See all failures |
| Good for development | Good for comprehensive testing |

### Caching Considerations

Promptfoo caches results to avoid redundant work:

1. **Provider response caching**: If you run the same prompt twice, the LLM response is cached (optional)
2. **Embedding caching**: Same text is only embedded once
3. **Grading isn't cached by default**: Model-graded assertions are re-evaluated each run

**Why Not Cache Grading?**

Grading models are updated. The same rubric might be evaluated differently by an updated model. Caching grading results could hide improvements or regressions in grading quality.

## Validation

### Why Validate Assertions?

A typo in an assertion type (`"containss"` instead of `"contains"`) would fail at runtime—potentially after many expensive API calls. Promptfoo validates assertions upfront.

[📂 src/assertions/validateAssertions.ts](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/assertions/validateAssertions.ts)

### Common Validation Errors

**Missing `type`**:

```yaml
# ❌ Wrong - missing type
assert:
  - value: "hello"

# ✅ Correct
assert:
  - type: contains
    value: "hello"
```

**YAML Formatting Errors** (very common):

```yaml
# ❌ Wrong - value is a separate list item
assert:
  - type: python
  - value: file://script.py

# ✅ Correct - value is property of same item
assert:
  - type: python
    value: file://script.py
```

In YAML, indentation matters. The dash (`-`) starts a new list item. Properties of the same item must be indented under the same item.

**Invalid Assertion Type**:

```yaml
# ❌ Typo
assert:
  - type: containz
    value: "hello"

# Error: Invalid assertion type "containz". Did you mean "contains"?
```

**Invalid Negation**:

```yaml
# ❌ Can't negate this (what would not-latency mean?)
assert:
  - type: not-latency
    threshold: 5000

# Error: Assertion type "latency" does not support negation
```

### Limits

To prevent denial-of-service via malicious configurations:

```typescript
const MAX_ASSERTIONS_PER_TEST = 10000;
```

If you legitimately need more than 10,000 assertions per test, you probably need to restructure your evaluation approach.

## Model-Graded Assertions Deep Dive

### The Philosophy of LLM-as-Judge

Using an LLM to evaluate another LLM's output seems circular. Why does this work?

**The insight**: Judging is easier than generating.

Consider translation. A non-native speaker might struggle to write a perfect French sentence, but they can often tell which of two translations is better—especially if one has obvious errors.

Similarly, a model might struggle to generate a perfect response, but it can often reliably identify when a response fails to meet stated criteria. The judging task is more constrained: you're given criteria and asked to check against them.

**When LLM-as-Judge Works Well**:

- Clear, specific criteria (not "be good" but "address all three sub-questions")
- Factual checks (did the response mention X?)
- Obvious failures (response is off-topic, rude, or factually wrong)
- Comparative judgment (which response is better?)

**When LLM-as-Judge Struggles**:

- Subtle quality distinctions (is this 7/10 or 8/10?)
- Domain expertise (is this legal advice correct?)
- Subjective preferences (is this writing style appropriate?)
- Evaluating itself (self-evaluation bias)

### The Grading Prompt Architecture

[📂 src/matchers.ts](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/matchers.ts)

Each model-graded assertion type has a specialized prompt template:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           Prompt Template Library                           │
│                                                                             │
│   llm-rubric:                                                               │
│     "You are evaluating an AI response. [prompt] [response] [rubric]"       │
│                                                                             │
│   factuality:                                                               │
│     "Given these facts... does the response contradict any?"                │
│                                                                             │
│   answer-relevance:                                                         │
│     "Does this response directly address the question asked?"               │
│                                                                             │
│   context-faithfulness:                                                     │
│     "Is this response grounded in the provided context?"                    │
│                                                                             │
│   Each template is carefully tuned for its specific evaluation task.        │
└─────────────────────────────────────────────────────────────────────────────┘
                                   │
                                   ▼ Fill template with test data
                                   │
┌─────────────────────────────────────────────────────────────────────────────┐
│                           Completed Grading Prompt                          │
│                                                                             │
│   You are an expert evaluator assessing AI responses.                       │
│                                                                             │
│   ORIGINAL PROMPT TO AI:                                                    │
│   "What is the capital of France?"                                          │
│                                                                             │
│   AI RESPONSE:                                                              │
│   "The capital of France is Paris, located along the Seine River."          │
│                                                                             │
│   EVALUATION CRITERIA:                                                      │
│   The response should be accurate and directly answer the question.         │
│                                                                             │
│   Does the response meet the criteria? Score from 1-10 and explain.         │
└─────────────────────────────────────────────────────────────────────────────┘
                                   │
                                   ▼ Send to grader LLM
                                   │
┌─────────────────────────────────────────────────────────────────────────────┐
│                           Grader Response                                   │
│                                                                             │
│   Score: 9                                                                  │
│                                                                             │
│   The response correctly identifies Paris as the capital of France,         │
│   directly answering the question. The additional detail about the          │
│   Seine River is accurate but not strictly necessary. A perfect score       │
│   would require more concise direct answer, but this is an excellent        │
│   response overall.                                                         │
└─────────────────────────────────────────────────────────────────────────────┘
                                   │
                                   ▼ Parse response
                                   │
                           GradingResult {
                             pass: true,
                             score: 0.9,
                             reason: "Score: 9. The response correctly..."
                           }
```

### Choosing the Right Grader Model

| Model | Latency | Cost (per 1M input) | Quality | Best For |
|-------|---------|---------------------|---------|----------|
| gpt-4o-mini | ~500ms | $0.15 | Good | Most rubrics, high volume |
| gpt-4o | ~1.5s | $2.50 | Better | Complex criteria, nuanced judgment |
| gpt-4 | ~3s | $30 | Best | Critical evaluations, highest quality |
| claude-3-haiku | ~500ms | $0.25 | Good | Alternative vendor, cost-effective |
| claude-3-sonnet | ~1s | $3.00 | Better | Complex reasoning |

**Default is gpt-4o-mini** because:
- Fast: low latency enables high-volume evaluation
- Cheap: evaluation costs can exceed generation costs
- Sufficient: most rubrics don't need GPT-4-level reasoning

**Upgrade grader when**:
- Rubric requires complex reasoning or multi-step logic
- Subtle distinctions matter (7/10 vs 8/10)
- High stakes (production evaluation, model selection)
- Current grader is inconsistent or clearly wrong

### Factuality: A Specialized Pattern

[📂 src/assertions/factuality.ts](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/assertions/factuality.ts)

Factuality is the most specialized model-graded assertion. It's designed for a specific task: checking whether an LLM response contradicts known facts.

```yaml
assert:
  - type: factuality
    value: |
      - Paris is the capital of France
      - The Eiffel Tower is 330 meters tall
      - The Louvre is the world's most visited museum
```

**How It Works**:

The grading prompt is structured specifically for fact-checking:

```
Given these reference facts:
- Paris is the capital of France
- The Eiffel Tower is 330 meters tall
- The Louvre is the world's most visited museum

Does the AI response contradict any of these facts?
AI Response: "Paris is the capital of France. The Eiffel Tower 
stands at about 1,000 feet (roughly 305 meters) and attracts 
millions of visitors. The Louvre Museum is very popular."

Analyze for factual contradictions and report.
```

**The grading focuses on contradictions**, not completeness. The response doesn't need to mention all facts—it just shouldn't contradict them.

**Use Cases**:
- RAG applications: responses should match retrieved context
- Knowledge-grounded assistants: responses should align with known facts
- Fact-checking: detecting hallucinations

### G-Eval: Multi-Step Evaluation

[📂 src/assertions/geval.ts](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/assertions/geval.ts)

G-Eval is based on research showing that multi-step evaluation produces more reliable scores.

```yaml
assert:
  - type: g-eval
    value: "coherence"
    threshold: 0.7
```

**How G-Eval Works**:

```
1. Generate evaluation steps:
   ┌─────────────────────────────────────────────────────────────┐
   │ For the criterion "coherence":                              │
   │                                                             │
   │ Step 1: Read the entire response to understand the topic    │
   │ Step 2: Check if ideas flow logically from one to another   │
   │ Step 3: Identify any abrupt topic changes                   │
   │ Step 4: Assess if the conclusion follows from the content   │
   │ Step 5: Rate overall coherence on 1-10 scale                │
   └─────────────────────────────────────────────────────────────┘

2. Execute each step (chain-of-thought)

3. Aggregate into final score
```

**Built-in Criteria**:
- `coherence`: Logical flow and organization
- `fluency`: Grammar, readability, naturalness
- `relevance`: Addresses the prompt/question
- `consistency`: No internal contradictions

**Custom Criteria**:

```yaml
assert:
  - type: g-eval
    value: "technical accuracy for software engineering topics"
```

G-Eval generates evaluation steps for your custom criterion.

### Reliability and Limitations

Model-graded assertions have inherent limitations:

**Inter-rater reliability**: Run the same evaluation twice, you might get different scores. LLMs aren't perfectly consistent.

**Mitigation**: Use lower thresholds to account for noise. A score of 0.8 might fluctuate to 0.75 or 0.85.

**Position bias**: In comparisons, LLMs tend to favor responses presented first or last.

**Mitigation**: For `select-best`, responses are sometimes shuffled and evaluated multiple times.

**Self-evaluation bias**: A model evaluating its own output might be biased.

**Mitigation**: Use a different model for grading than generation when this matters.

**Prompt sensitivity**: Subtle changes to the grading prompt can change results.

**Mitigation**: Test your rubrics on known good/bad examples before using at scale.

## Integration Points

### Red Team Assertions

Red team assertions evaluate security and safety properties:

```yaml
assert:
  - type: promptfoo:redteam:harmful
  - type: promptfoo:redteam:jailbreak
  - type: promptfoo:redteam:pii-leak
```

These are namespaced (`promptfoo:redteam:`) and handled by specialized graders trained to detect:
- Harmful content generation
- Jailbreak attempts and successes
- PII exposure
- Prompt injection vulnerabilities

**How They Work**:

Red team assertions don't use the generic `llm-rubric` flow. They have custom prompts and detection logic tuned for adversarial scenarios.

### Guardrails

The `guardrails` assertion integrates with content moderation systems:

```yaml
assert:
  - type: guardrails
    config:
      purpose: "content-moderation"
```

**Behavior**:
- If the guardrail **allows** content: Response is evaluated normally
- If the guardrail **blocks** content: Assertion passes (system correctly refused)

This inverts the usual pass/fail logic—a refusal is a success for safety assertions.

## Practical Guidance: Choosing Assertions

### Decision Framework

When choosing assertions for a test case, work through this decision tree:

```
START: What are you testing?
        │
        ├─► Specific text must appear?
        │   └─► Use: contains, contains-all, contains-any
        │
        ├─► Specific text must NOT appear?
        │   └─► Use: not-contains (PII, profanity, competitors)
        │
        ├─► Output must be valid format?
        │   └─► Use: is-json, is-sql, is-html
        │
        ├─► Output should match reference semantically?
        │   └─► Use: similar (with appropriate threshold)
        │
        ├─► Subjective quality judgment?
        │   └─► Use: llm-rubric (write clear criteria)
        │
        ├─► Checking against known facts?
        │   └─► Use: factuality
        │
        ├─► Performance constraints?
        │   └─► Use: latency, cost
        │
        ├─► Need custom logic?
        │   └─► Use: javascript, python, webhook
        │
        └─► Comparing multiple responses?
            └─► Use: select-best, max-score
```

### Common Mistakes

**Mistake 1: Using llm-rubric when contains would work**

```yaml
# ❌ Over-engineered
assert:
  - type: llm-rubric
    value: "The response should mention the refund policy"

# ✅ Simpler and cheaper
assert:
  - type: contains
    value: "refund"
```

**Mistake 2: Vague rubrics**

```yaml
# ❌ Too vague
assert:
  - type: llm-rubric
    value: "The response should be good"

# ✅ Specific and actionable
assert:
  - type: llm-rubric
    value: |
      The response should:
      1. Directly answer the user's question
      2. Provide at least one concrete example
      3. Be under 200 words
      4. Not include any caveats or disclaimers
```

**Mistake 3: Ignoring threshold tuning**

```yaml
# ❌ Default threshold might be wrong for your use case
assert:
  - type: similar
    value: "Expected response"

# ✅ Tuned threshold based on testing
assert:
  - type: similar
    value: "Expected response"
    threshold: 0.85  # Tested on known good/bad examples
```

**Mistake 4: Too many assertions**

Having 20 assertions per test makes debugging hard. If many are related, consider:
- Using assertion sets to group related checks
- Creating a single `llm-rubric` with multi-point criteria
- Splitting into multiple focused tests

**Mistake 5: Not using named scores**

```yaml
# ❌ Can't analyze quality dimensions
assert:
  - type: llm-rubric
    value: "accurate, helpful, and concise"

# ✅ Track each dimension
assert:
  - type: llm-rubric
    value: "factually accurate"
    metric: accuracy
  - type: llm-rubric
    value: "helpful and actionable"
    metric: helpfulness
  - type: llm-rubric
    value: "concise, under 100 words"
    metric: conciseness
```

## File Reference

| File | Purpose |
|------|---------|
| [src/assertions/index.ts](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/assertions/index.ts) | Main dispatcher, handler registry, runAssertions orchestration |
| [src/assertions/assertionsResult.ts](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/assertions/assertionsResult.ts) | Score aggregation, weighted average, named scores |
| [src/assertions/validateAssertions.ts](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/assertions/validateAssertions.ts) | Upfront validation, error messages, limits |
| [src/assertions/contains.ts](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/assertions/contains.ts) | Text matching handlers |
| [src/assertions/similar.ts](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/assertions/similar.ts) | Similarity assertions, embedding metrics |
| [src/assertions/llmRubric.ts](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/assertions/llmRubric.ts) | LLM-as-judge implementation |
| [src/assertions/javascript.ts](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/assertions/javascript.ts) | Custom JavaScript assertions |
| [src/assertions/json.ts](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/assertions/json.ts) | JSON validation, schema checking |
| [src/assertions/factuality.ts](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/assertions/factuality.ts) | Fact-checking implementation |
| [src/assertions/geval.ts](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/assertions/geval.ts) | G-Eval multi-step evaluation |
| [src/matchers.ts](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/matchers.ts) | Core matching logic for model-graded assertions |

## Key Design Decisions

| Decision | Rationale | Trade-off | Alternative Considered |
|----------|-----------|-----------|------------------------|
| Handler map pattern | Extensibility, isolation, testability | Indirection, more files | Class hierarchy (rejected: functions simpler for this use case) |
| `inverse` flag for negation | DRY—one implementation handles both positive and negative | Handler must consider inverse | Separate handlers for each (rejected: doubles code, inconsistency risk) |
| Weighted average scoring | Intuitive, familiar aggregation method | Might not fit all use cases | Custom scoring always (rejected: too complex for simple cases) |
| Default grader `gpt-4o-mini` | Fast, cheap, good enough for most rubrics | Less sophisticated than GPT-4 | Default to GPT-4 (rejected: cost prohibitive for high volume) |
| Concurrent assertion execution | Speed—run multiple checks in parallel | Rate limits, resource usage | Sequential always (rejected: too slow for many assertions) |
| Zod validation upfront | Catch errors before expensive operations | Startup time overhead | Runtime-only validation (rejected: wastes API calls on typos) |
| JSON Schema for structural validation | Standards-based, tools exist | Requires learning JSON Schema | Custom DSL (rejected: reinventing the wheel) |
| Two-level scoring (pass + score) | Binary for automation, continuous for analysis | Conceptual complexity | Just boolean (rejected: loses nuance) |
| Template variables in assertions | Data-driven testing, single source of truth | Learning curve (Nunjucks) | Static values only (rejected: requires duplication) |
