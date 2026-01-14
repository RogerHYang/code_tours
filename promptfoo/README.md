# Promptfoo Architectural Overview: Evaluations

## Table of Contents

1. [Executive Summary](#executive-summary)
2. [The Mental Model: What Is an Evaluation?](#the-mental-model-what-is-an-evaluation)
3. [High-Level Architecture](#high-level-architecture)
4. [Part 1: The Entry Point](#part-1-the-entry-point)
   - [Why a Separate Entry Point?](#why-a-separate-entry-point)
   - [Configuration Resolution: Why It's Complicated](#configuration-resolution-why-its-complicated)
5. [Part 2: The Evaluator](#part-2-the-evaluator)
   - [Why an Evaluator Class?](#why-an-evaluator-class)
   - [The Core Loop: What Happens During Evaluation](#the-core-loop-what-happens-during-evaluation)
   - [Variable Expansion: The Cartesian Product](#variable-expansion-the-cartesian-product)
   - [Concurrency: The Balancing Act](#concurrency-the-balancing-act)
6. [Part 3: The Provider System](#part-3-the-provider-system)
   - [Why Abstract Providers?](#why-abstract-providers)
   - [The Provider Interface](#the-provider-interface)
   - [The Provider Registry: How IDs Become Objects](#the-provider-registry-how-ids-become-objects)
   - [Provider Categories: A Taxonomy](#provider-categories-a-taxonomy)
   - [Why So Many Provider Options?](#why-so-many-provider-options)
7. [Part 4: The Assertion System](#part-4-the-assertion-system)
   - [The Fundamental Challenge: What Makes a Good Response?](#the-fundamental-challenge-what-makes-a-good-response)
   - [Assertion Types: A Spectrum of Sophistication](#assertion-types-a-spectrum-of-sophistication)
   - [Why So Many Assertion Types?](#why-so-many-assertion-types)
   - [The Assertion Handler Architecture](#the-assertion-handler-architecture)
   - [Model-Graded Assertions: LLM-as-Judge](#model-graded-assertions-llm-as-judge)
   - [Assertion Combination: AND vs OR](#assertion-combination-and-vs-or)
   - [Scoring: From Multiple Assertions to One Number](#scoring-from-multiple-assertions-to-one-number)
8. [Part 5: Prompt Templates](#part-5-prompt-templates)
   - [Why Templates?](#why-templates)
   - [Variable Substitution](#variable-substitution)
   - [Special Variables](#special-variables)
   - [Prompt Sources: More Than Just Strings](#prompt-sources-more-than-just-strings)
9. [Part 6: Test Cases](#part-6-test-cases)
   - [What Is a Test Case?](#what-is-a-test-case)
   - [Test-Specific Assertions](#test-specific-assertions)
   - [Test-Specific Providers](#test-specific-providers)
   - [providerOutput: Skip the API Call](#provideroutput-skip-the-api-call)
   - [Metadata: Tagging and Filtering](#metadata-tagging-and-filtering)
10. [Part 7: The Result Model](#part-7-the-result-model)
    - [What Gets Stored?](#what-gets-stored)
    - [Why So Much Data?](#why-so-much-data)
11. [Part 8: Comparison Assertions](#part-8-comparison-assertions)
    - [The Comparison Problem](#the-comparison-problem)
    - [How Comparison Works](#how-comparison-works)
    - [`select-best`: LLM-Based Comparison](#select-best-llm-based-comparison)
    - [`max-score`: Automated Winner Selection](#max-score-automated-winner-selection)
12. [Part 9: Concurrency Deep Dive](#part-9-concurrency-deep-dive)
    - [Why Concurrency Is Tricky](#why-concurrency-is-tricky)
    - [Promptfoo's Concurrency Strategy](#promptfoos-concurrency-strategy)
    - [Rate Limit Handling](#rate-limit-handling)
13. [Part 10: The Data Flow](#part-10-the-data-flow)
    - [Phase 1: Configuration Loading](#phase-1-configuration-loading)
    - [Phase 2: Expansion](#phase-2-expansion)
    - [Phase 3: Execution](#phase-3-execution)
    - [Phase 4: Comparison (if applicable)](#phase-4-comparison-if-applicable)
    - [Phase 5: Output](#phase-5-output)
14. [Part 11: Advanced Features](#part-11-advanced-features)
    - [Red Team Testing](#red-team-testing)
    - [Tracing](#tracing)
    - [Extension Hooks](#extension-hooks)
    - [Caching](#caching)
15. [Part 12: Key Design Decisions](#part-12-key-design-decisions)
    - [Why No Real-Time Streaming to UI?](#why-no-real-time-streaming-to-ui)
    - [Why Drizzle ORM?](#why-drizzle-orm)
    - [Why SQLite?](#why-sqlite)
16. [Part 13: Common Patterns](#part-13-common-patterns)
    - [Pattern: Progressive Assertion Complexity](#pattern-progressive-assertion-complexity)
    - [Pattern: Provider Comparison](#pattern-provider-comparison)
    - [Pattern: Regression Testing](#pattern-regression-testing)
    - [Pattern: Cost Control](#pattern-cost-control)
17. [File Reference](#file-reference)

## Executive Summary

**Promptfoo** is an open-source framework for evaluating and testing LLM applications. This document provides a deep architectural understanding of how evaluations work—not just *what* happens, but *why* the system is designed this way.

**The Core Problem Promptfoo Solves**: Testing LLM outputs is fundamentally different from testing traditional software. There's no simple "expected output" to compare against. Responses vary, quality is subjective, and the same prompt might produce different results each time. Promptfoo provides a structured way to define what "good" looks like and measure whether your LLM application achieves it.

## The Mental Model: What Is an Evaluation?

Before diving into architecture, let's establish the conceptual model:

**An evaluation answers the question**: "How well does my LLM application perform across a variety of scenarios?"

To answer this, you need:

1. **Prompts**: The instructions or questions you send to the LLM. These might be templates with variables like "Translate '{{text}}' to {{language}}" that get filled in for each test.

2. **Providers**: The LLM APIs you're testing. This could be OpenAI's GPT-4, Anthropic's Claude, a local model, or multiple providers for comparison.

3. **Test Cases**: Specific scenarios to evaluate. Each test case provides values for the prompt variables and defines what a good response looks like.

4. **Assertions**: Criteria for judging responses. These range from simple checks ("does it contain this word?") to sophisticated evaluations ("does another LLM think this response is helpful?").

**The Cartesian Product Problem**: If you have 3 prompts, 2 providers, and 100 test cases, that's 600 individual evaluations to run. Promptfoo manages this combinatorial explosion efficiently.

## High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              CLI / API Entry                                │
│                                                                             │
│   The user runs "promptfoo eval" or calls the programmatic API.             │
│   This layer parses arguments and kicks off the process.                    │
└──────────────────────────────────┬──────────────────────────────────────────┘
                                   │
                                   ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                          Configuration Loader                               │
│                                                                             │
│   Reads YAML config files, resolves file references, merges settings.       │
│   Transforms user-friendly config into internal data structures.            │
└──────────────────────────────────┬──────────────────────────────────────────┘
                                   │
                                   ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                              EVALUATOR                                      │
│                                                                             │
│   The heart of the system. Manages the execution of all evaluations,        │
│   handles concurrency, collects results, and coordinates grading.           │
│                                                                             │
│  ┌─────────────┐  ┌──────────────┐  ┌─────────────┐  ┌─────────────────┐    │
│  │   Prompts   │  │  Providers   │  │ Test Cases  │  │   Assertions    │    │
│  │             │──│              │──│             │──│                 │    │
│  └─────────────┘  └──────────────┘  └─────────────┘  └─────────────────┘    │
└──────────────────────────────────┬──────────────────────────────────────────┘
                                   │
                                   ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                           Persistence Layer                                 │
│                                                                             │
│   Stores results in SQLite for browsing, comparison, and sharing.           │
│   Enables the web UI to display results in real-time.                       │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Part 1: The Entry Point

[📂 src/commands/eval.ts](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/commands/eval.ts#L80)

### Why a Separate Entry Point?

Promptfoo supports multiple ways to run evaluations:

1. **CLI Command**: `promptfoo eval -c config.yaml`
2. **Programmatic API**: Import and call from your own code
3. **CI/CD Integration**: Run as part of automated testing pipelines

The entry point (`doEval()`) serves as the translation layer between the user interface and the core evaluation engine. It handles concerns that are specific to how the evaluation was invoked:

- Parsing command-line arguments
- Reading configuration files from disk
- Displaying progress to the terminal
- Formatting output (JSON, HTML, table)
- Sharing results to the cloud

**Design Principle**: The core evaluator has no knowledge of command-line arguments or file I/O. This separation means the same evaluation logic works whether invoked via CLI, API, or test suite.

### Configuration Resolution: Why It's Complicated

Configuration can come from many sources, and they merge in specific ways:

1. **Default configuration** (baked into promptfoo)
2. **Configuration file** (`promptfooconfig.yaml`)
3. **Command-line arguments** (override everything)
4. **Environment variables** (API keys, feature flags)

**The Merging Challenge**: A user might define prompts in the config file but override providers on the command line. The resolution logic must handle partial overrides gracefully.

**Why YAML?** Configuration files use YAML because:
- It's human-readable and editable
- It supports multi-line strings (crucial for prompts)
- It allows comments (documenting why certain settings exist)
- JSON can be embedded when needed (for structured data)

## Part 2: The Evaluator

[📂 src/evaluator.ts](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/evaluator.ts#L785)

### Why an Evaluator Class?

The evaluator is implemented as a class rather than standalone functions for several reasons:

**State Management**: An evaluation accumulates state over time—results, statistics, error counts, token usage. A class provides a natural container for this state.

**Lifecycle Hooks**: The evaluator needs to run setup before evaluations, cleanup after, and potentially handle interruptions mid-flight. A class with methods like `initialize()`, `evaluate()`, and `cleanup()` expresses this lifecycle clearly.

**Testability**: Having a class makes it easier to mock dependencies and test the evaluator in isolation.

### The Core Loop: What Happens During Evaluation

At its heart, the evaluator executes a nested loop:

```
For each test case:
    For each prompt:
        For each provider:
            1. Render the prompt with test variables
            2. Call the provider API
            3. Run assertions on the response
            4. Record the result
```

**Why This Ordering?** Test cases are the outer loop because:
- It's more intuitive for users (they think in terms of test cases)
- It enables comparison assertions that compare providers on the same input
- It groups related results together in the UI

### Variable Expansion: The Cartesian Product

Consider this configuration:

```yaml
tests:
  - vars:
      language: ["English", "Spanish", "French"]
      topic: ["science", "history"]
```

This single test case definition expands into 6 test cases (3 languages × 2 topics). The evaluator performs this expansion before the main loop.

**Why Expansion at Runtime?** 
- It keeps configuration concise
- Users can programmatically generate test variations
- The same pattern works whether you have 2 combinations or 10,000

**The `repeat` Option**: For statistical significance, you can run each test multiple times. With `repeat: 5`, each of those 6 combinations runs 5 times, yielding 30 total evaluations. This helps account for LLM non-determinism.

### Concurrency: The Balancing Act

[📂 src/evaluator.ts](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/evaluator.ts#L1629)

Running evaluations sequentially would be painfully slow—a 1000-test evaluation calling GPT-4 might take hours. Concurrent execution is essential.

**But Concurrency Has Limits**:

1. **Rate Limits**: LLM providers throttle requests. Too much concurrency triggers errors.
2. **Cost Control**: Parallel requests rack up costs faster. Users need time to abort.
3. **Dependencies**: Some tests depend on previous results.
4. **Resource Exhaustion**: Memory, network connections, file handles.

**Promptfoo's Approach**: Configurable concurrency with smart defaults.

- Default: 4 concurrent evaluations
- Providers can specify their own limits (e.g., Anthropic has stricter rate limits)
- Automatic reduction when using features that require ordering

**Forced Sequential Execution**: Certain features require sequential execution:

1. **Multi-turn Conversations** (`_conversation` variable): Each turn depends on the previous response. You can't run turns concurrently.

2. **Value Passing** (`storeOutputAs`): When one test's output becomes another test's input, ordering matters.

3. **Rate Limiting** (`--delay`): Explicit delays between requests require sequencing.

**The Serial/Concurrent Split**: The evaluator partitions work into two buckets:
- Tests marked `runSerially: true` run one at a time
- Everything else runs with the configured concurrency

This allows mixing both patterns in a single evaluation.

## Part 3: The Provider System

[📂 src/providers/index.ts](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/providers/index.ts#L1)

### Why Abstract Providers?

The provider abstraction is one of promptfoo's most valuable design decisions. Without it, every evaluation would be tied to a specific LLM API.

**What the Abstraction Enables**:

1. **Comparison Testing**: Run the same tests against GPT-4, Claude, and a local model. The evaluator treats them identically.

2. **Easy Switching**: Change `openai:gpt-4` to `anthropic:claude-3-opus` and everything still works.

3. **Custom Providers**: Wrap your company's internal API in the provider interface and use all of promptfoo's tooling.

4. **Mock Testing**: Use a deterministic "test" provider for CI/CD without real API calls.

### The Provider Interface

[📂 src/types/providers.ts](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/types/providers.ts#L94)

Every provider must implement a small set of methods:

**`id()`**: Returns a unique identifier like `"openai:gpt-4o"`. This is used for caching, display, and configuration matching.

**`callApi(prompt, context)`**: The core method. Takes a prompt string, returns a response. The context provides additional information like:
- Variable values used
- Previous conversation turns
- The original test case

**Why Context?** Some providers need more than just the prompt:
- Chat providers need conversation history
- Tool-using providers need function definitions
- Some providers adjust behavior based on test metadata

**Optional Methods**:

- `callEmbeddingApi()`: For similarity comparisons
- `callClassificationApi()`: For classification tasks
- `cleanup()`: For resource management (closing connections, etc.)

### The Provider Registry: How IDs Become Objects

[📂 src/providers/registry.ts](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/providers/registry.ts#L1)

When you write `openai:gpt-4o` in configuration, how does promptfoo know what to do?

**The Registry Pattern**: A list of factory functions, each with a test function that examines the provider ID.

```
User specifies: "openai:gpt-4o"

Registry checks:
  - Does it start with "openai:"? → Use OpenAI factory
  - Does it start with "anthropic:"? → Use Anthropic factory
  - Does it start with "http://"? → Use HTTP endpoint factory
  - Does it end with ".py"? → Use Python script factory
  ...
```

**Why This Pattern?**

1. **Extensibility**: Adding a new provider type means adding one entry to the registry. No core code changes needed.

2. **Lazy Loading**: Provider implementations are loaded only when needed. The Anthropic SDK isn't loaded if you're only using OpenAI.

3. **Disambiguation**: Clear precedence when patterns might overlap.

### Provider Categories: A Taxonomy

| Category | Pattern | Example | Use Case |
|----------|---------|---------|----------|
| **Cloud APIs** | `provider:model` | `openai:gpt-4o` | Production LLMs |
| **Local Models** | `ollama:model` | `ollama:llama2` | Privacy, cost savings |
| **HTTP Endpoints** | `http://...` | `http://localhost:8080` | Custom deployments |
| **Script-based** | `file://script.py` | `file://provider.py:call` | Custom logic |
| **Test/Mock** | `echo`, `test` | `echo:hello` | Development, CI/CD |

### Why So Many Provider Options?

**Real-World Complexity**: Enterprises don't just use one LLM. They might have:
- GPT-4 for quality-critical tasks
- Claude for certain compliance requirements
- A fine-tuned local model for domain-specific work
- A mock provider for testing

Promptfoo's broad provider support means one tool handles all these scenarios.

## Part 4: The Assertion System

[📂 src/assertions/index.ts](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/assertions/index.ts#L1)

### The Fundamental Challenge: What Makes a Good Response?

This is the hardest problem in LLM evaluation. Unlike traditional software where you can check `result === expectedValue`, LLM outputs are:

- **Non-deterministic**: The same prompt might yield different responses
- **Semantically equivalent but textually different**: "Yes" and "Absolutely" might mean the same thing
- **Context-dependent**: A good customer service response differs from a good code review

### Assertion Types: A Spectrum of Sophistication

[📂 src/types/index.ts](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/types/index.ts#L482)

Promptfoo provides assertions ranging from simple to complex:

**Exact Matching** (fast, deterministic):
- `contains`: The response includes this substring
- `equals`: The response exactly matches this string
- `regex`: The response matches this pattern

**Structural Validation** (medium complexity):
- `is-json`: The response is valid JSON
- `is-valid-openai-tools-call`: The response is a correctly formatted tool call
- `contains-json`: JSON is embedded somewhere in the response

**Semantic Similarity** (requires embeddings):
- `similar`: Cosine similarity between response and expected text
- Uses embedding models to compare meaning, not exact words

**Model-Graded** (most sophisticated):
- `llm-rubric`: Another LLM evaluates the response against criteria
- `factuality`: Checks if claims are factually accurate
- `answer-relevance`: Checks if the response addresses the question

### Why So Many Assertion Types?

Different evaluation scenarios need different tools:

**Scenario 1: Structured Output**
You want the LLM to return JSON with specific fields. Use `is-json` plus custom validation.

**Scenario 2: Translation Quality**
Exact matching doesn't work—many valid translations exist. Use `similar` for semantic comparison or `llm-rubric` for quality judgment.

**Scenario 3: Safety Testing**
Checking if the response contains harmful content. Use `llm-rubric` with a safety-focused rubric.

**Scenario 4: Performance Testing**
Checking if responses are fast enough. Use `latency` assertions.

### The Assertion Handler Architecture

[📂 src/assertions/index.ts](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/assertions/index.ts#L116)

Each assertion type has a handler function. The dispatcher looks up the appropriate handler:

```
Assertion type: "contains"
  → Handler: handleContains(params)
  → Checks if response includes the expected substring
  → Returns: { pass: true/false, score: 0-1, reason: "..." }
```

**Why a Handler Map?**

1. **Isolation**: Each assertion type is implemented independently. A bug in `llm-rubric` doesn't affect `contains`.

2. **Extensibility**: Adding a new assertion type means adding one handler function.

3. **Testing**: Individual handlers can be unit tested in isolation.

### Model-Graded Assertions: LLM-as-Judge

[📂 src/assertions/index.ts](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/assertions/index.ts#L104)

This is where things get interesting. For complex evaluation criteria, promptfoo uses another LLM to grade responses.

**How It Works**:

1. You define a rubric: "The response should be helpful, accurate, and concise"
2. Promptfoo constructs a grading prompt that includes:
   - The original prompt
   - The LLM's response
   - Your rubric
   - Instructions for scoring
3. A "grader" LLM evaluates and returns a structured judgment
4. Promptfoo parses the judgment into pass/fail and score

**Why Use an LLM to Grade?**

Human evaluation doesn't scale. You can't manually review 10,000 test responses. But humans are good at writing rubrics—descriptions of what good looks like. LLMs can then apply those rubrics at scale.

**The Grader Provider**: By default, promptfoo uses `gpt-4o-mini` for grading—it's fast and cheap. But you can configure a different grader if needed. Some users even use the same model for both generation and grading (though this can introduce bias).

**Token Accounting**: Grading uses tokens too. Promptfoo tracks "assertion tokens" separately from "generation tokens" so you understand where costs are going.

### Assertion Combination: AND vs OR

When a test has multiple assertions:

```yaml
assert:
  - type: contains
    value: "Hello"
  - type: not-contains
    value: "Goodbye"
  - type: llm-rubric
    value: "Should be friendly"
```

By default, **all assertions must pass** (AND logic). The test only passes if every assertion succeeds.

**But Sometimes You Want OR Logic**: "The response should contain either 'yes' or 'affirmative'". Promptfoo supports assertion sets for this:

```yaml
assert:
  - type: assert-set
    assert:
      - type: contains
        value: "yes"
      - type: contains
        value: "affirmative"
    threshold: 0.5  # At least 50% of these must pass
```

### Scoring: From Multiple Assertions to One Number

A test might have 5 assertions. How do we get a single score?

**Default Scoring: Weighted Average**

Each assertion has a weight (default 1). The final score is:

```
score = Σ(assertion_score × weight) / Σ(weights)
```

If all assertions pass with score 1.0, the test scores 1.0. If half fail, it scores 0.5.

**Custom Scoring Functions**

Sometimes the default isn't right. Maybe assertion A is 10x more important than B. Or maybe you want a binary pass/fail regardless of individual scores.

Promptfoo allows custom scoring functions:

```yaml
tests:
  - vars: { ... }
    assertScoringFunction: file://scorer.js:myCustomScorer
```

Your function receives all assertion results and can compute any score you want.

## Part 5: Prompt Templates

[📂 src/prompts/index.ts](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/prompts/index.ts#L1)

### Why Templates?

Static prompts are limited. Real applications need dynamic content:

- User names, dates, context
- Retrieved documents for RAG
- Few-shot examples that vary by scenario

Promptfoo uses **Nunjucks**, a templating language similar to Jinja2 (Python) or Liquid (Ruby).

### Variable Substitution

The simplest template feature:

```
Template: "Translate '{{text}}' to {{language}}"
Variables: { text: "Hello", language: "French" }
Result:   "Translate 'Hello' to French"
```

The `{{ }}` syntax marks variable placeholders.

### Special Variables

Some variables are provided by promptfoo rather than the test case:

**`_conversation`**: In multi-turn tests, this contains the conversation history—previous prompts and responses. It enables chatbot testing.

**`__expected`**: The expected output for the current test (if defined). Useful for few-shot prompts that include examples.

### Prompt Sources: More Than Just Strings

Prompts can come from various sources:

| Source | Syntax | Use Case |
|--------|--------|----------|
| Inline | `"Translate {{text}}"` | Simple prompts |
| File | `file://prompt.txt` | Long prompts, version control |
| Python | `file://prompts.py:gen` | Dynamic prompt generation |
| JSON | `file://chat.json` | Chat-format prompts |

**Why Python/JavaScript Prompts?**

Sometimes the prompt itself needs logic:

- Selecting examples based on the input
- Formatting complex data structures
- Calling external services for retrieval

Script-based prompts allow arbitrary code to generate the final prompt.

## Part 6: Test Cases

[📂 src/types/index.ts](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/types/index.ts#L737)

### What Is a Test Case?

A test case is a specific scenario to evaluate. At minimum, it provides variable values:

```yaml
tests:
  - vars:
      question: "What is the capital of France?"
```

But test cases can do much more.

### Test-Specific Assertions

Override or add assertions for specific tests:

```yaml
defaultTest:
  assert:
    - type: llm-rubric
      value: "Response should be helpful"

tests:
  - vars:
      question: "What is 2+2?"
    assert:
      - type: contains
        value: "4"  # This test also requires "4" in the response
```

### Test-Specific Providers

Sometimes you want to test a specific provider for certain cases:

```yaml
tests:
  - vars:
      task: "Write secure code"
    provider: openai:gpt-4o  # Only use GPT-4 for security-critical tests
```

### providerOutput: Skip the API Call

For testing assertions without calling an LLM:

```yaml
tests:
  - vars: { ... }
    providerOutput: "This is a mock response"
```

Useful for:
- Testing assertion logic
- Reproducing specific edge cases
- CI/CD without API costs

### Metadata: Tagging and Filtering

Tests can carry arbitrary metadata:

```yaml
tests:
  - vars: { ... }
    metadata:
      category: "safety"
      severity: "high"
      author: "security-team"
```

This metadata flows through to results, enabling:
- Filtering in the UI
- Custom reporting
- Integration with other systems

## Part 7: The Result Model

[📂 src/types/index.ts](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/types/index.ts#L296)

### What Gets Stored?

Each evaluation produces a result with comprehensive data:

**Identification**:
- `promptIdx`, `testIdx`: Position in the results matrix (which prompt × which test)
- `promptId`: Hash of the prompt for deduplication
- `provider`: Which LLM generated the response

**The Response**:
- `response`: The raw provider response (includes output, usage stats, etc.)
- `error`: If the call failed, what went wrong

**Grading**:
- `success`: Overall pass/fail
- `score`: Aggregate score (0-1)
- `namedScores`: Individual assertion scores (e.g., `{ "accuracy": 0.9, "relevance": 0.8 }`)
- `gradingResult`: Detailed breakdown of all assertions

**Performance**:
- `latencyMs`: How long the API call took
- `cost`: Estimated cost in dollars
- `tokenUsage`: Token counts for billing and analysis

### Why So Much Data?

**Debugging**: When a test fails, you need context. The stored data lets you reconstruct exactly what happened.

**Analysis**: Aggregate statistics like "average latency for GPT-4 on safety tests" require granular data.

**Comparison**: Comparing providers requires storing provider identity with each result.

**Reproducibility**: With stored prompts and configurations, you can re-run evaluations or investigate discrepancies.

## Part 8: Comparison Assertions

[📂 src/assertions/index.ts](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/assertions/index.ts#L610)

### The Comparison Problem

Most assertions evaluate each response independently. But sometimes you want to compare across providers:

- "Which provider gives the best response to this query?"
- "Is GPT-4 consistently better than GPT-3.5 for this use case?"

### How Comparison Works

Comparison assertions run **after** all providers have responded to a test:

```
Test case: "Explain quantum computing simply"

Responses:
  - GPT-4: [long explanation]
  - Claude: [different explanation]
  - Llama: [third explanation]

Comparison assertion (select-best):
  → Grader LLM compares all three
  → Picks GPT-4 as the best
  → GPT-4 passes, others fail
```

### `select-best`: LLM-Based Comparison

The grader LLM sees all responses and picks the winner based on your criteria:

```yaml
assert:
  - type: select-best
    value: "Which response is most accurate and easiest to understand?"
```

**Use Case**: When you can't define "good" precisely, but you can compare two responses and say which is better.

### `max-score`: Automated Winner Selection

For simpler comparisons, `max-score` picks whichever response scored highest on other assertions:

```yaml
assert:
  - type: accuracy  # All providers get accuracy scores
    ...
  - type: max-score  # Highest accuracy score wins
```

**Use Case**: When you have numeric metrics and want the mathematically highest one to "win."

## Part 9: Concurrency Deep Dive

### Why Concurrency Is Tricky

LLM API calls are slow—often 1-10 seconds each. Running 1000 tests sequentially would take hours. Concurrency is essential for practical use.

But several factors constrain concurrency:

**Rate Limits**: OpenAI limits requests per minute. Exceed it and requests fail. Different providers have different limits, and they change over time.

**Ordered Dependencies**: Some tests depend on previous results. Multi-turn conversations can't be parallelized.

**Resource Exhaustion**: Too many concurrent connections can exhaust file descriptors, memory, or network resources.

**Error Blast Radius**: If many concurrent requests fail, you might waste significant API credits before detecting the problem.

### Promptfoo's Concurrency Strategy

**Configurable with Smart Defaults**:
- Default concurrency: 4
- Override with `--max-concurrency` or `evaluateOptions.maxConcurrency`

**Provider-Specific Limits**: Providers can declare their own limits:
```yaml
providers:
  - id: anthropic:claude-3-opus
    config:
      maxConcurrency: 2  # Anthropic's stricter rate limits
```

**Automatic Reduction**: When using features that require ordering, concurrency drops to 1 automatically:
- `_conversation` variable (multi-turn)
- `storeOutputAs` (value passing)
- `--delay` flag (rate limiting)

**The Two-Bucket Approach**:
1. Partition tests into "serial required" and "can be concurrent"
2. Run serial tests first, one at a time
3. Run concurrent tests with the configured concurrency

This allows mixing both patterns efficiently.

### Rate Limit Handling

When a provider returns a rate limit error (HTTP 429):

1. Promptfoo logs a warning
2. The request is retried with exponential backoff
3. After several retries, it fails permanently

**Configuration**: You can set delays between requests:
```yaml
evaluateOptions:
  delay: 500  # Wait 500ms between requests
```

## Part 10: The Data Flow

Let's trace a complete evaluation from start to finish:

### Phase 1: Configuration Loading

```
User runs: promptfoo eval -c config.yaml

1. Parse command-line arguments
2. Read config.yaml
3. Merge with defaults and environment
4. Resolve file references (file://prompt.txt → actual content)
5. Validate configuration schema
6. Produce a TestSuite object
```

[📂 Configuration loading](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/util/config/load.ts#L1)

### Phase 2: Expansion

```
TestSuite:
  - 2 prompts
  - 3 providers  
  - 50 tests (some with array variables)
  - repeat: 2

Expansion:
  - Array variables expand: 50 → 120 test cases
  - Cartesian product: 2 × 3 × 120 × 2 = 1,440 evaluation tasks
  - Partition: 40 serial, 1,400 concurrent
```

[📂 Variable expansion](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/evaluator.ts#L735)

### Phase 3: Execution

```
Serial tasks (40):
  Run one at a time, maintain conversation state

Concurrent tasks (1,400):
  Run with concurrency limit (e.g., 8 at a time)

For each task:
  1. Render prompt template
  2. Check cache (skip API if cached)
  3. Call provider.callApi()
  4. Apply output transform (if configured)
  5. Run all assertions
  6. Compute aggregate score
  7. Store result
  8. Update progress display
```

[📂 Evaluation execution](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/evaluator.ts#L293)

### Phase 4: Comparison (if applicable)

```
If select-best or max-score assertions exist:
  1. Group results by test case
  2. For each test case, collect all provider responses
  3. Run comparison grading
  4. Update results with comparison outcomes
```

[📂 Comparison logic](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/evaluator.ts#L1715)

### Phase 5: Output

```
1. Calculate aggregate statistics
2. Generate output formats:
   - CLI table (always)
   - JSON file (if -o output.json)
   - HTML report (if -o output.html)
   - Web UI (if --view)
   - Cloud share (if --share)
3. Display summary
```

[📂 Output generation](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/table.ts#L1)

## Part 11: Advanced Features

### Red Team Testing

[📂 src/redteam/](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/redteam/index.ts#L1)

Red teaming goes beyond functional testing to adversarial testing—trying to make the LLM behave badly.

**Plugins** generate attack prompts:
- `harmful:hate`: Attempts to elicit hateful content
- `pii:direct`: Attempts to extract personal information
- `hijacking`: Attempts to redirect the conversation

**Strategies** modify attack delivery:
- `jailbreak`: Wraps attacks in jailbreak patterns
- `prompt-injection`: Embeds attacks in data that might be processed

**Graders** evaluate attack success:
- Did the model refuse appropriately?
- Did it leak sensitive information?
- Did it maintain its intended behavior?

**Why Red Teaming Matters**: Before deploying an LLM application, you need confidence it won't behave dangerously. Automated red teaming scales human intuition about attacks.

### Tracing

[📂 src/tracing/](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/tracing/evaluatorTracing.ts#L1)

Tracing captures detailed execution data for debugging and observability:

- When did each step start and end?
- What was the token count at each stage?
- Where did errors occur?

**Integration**: Traces can be exported to OpenTelemetry-compatible backends (Jaeger, Grafana, etc.) for visualization.

**Trace-Aware Assertions**:
- `trace-span-count`: Verify expected number of operations
- `trace-span-duration`: Check performance requirements
- `trace-error-spans`: Detect internal errors

### Extension Hooks

[📂 src/evaluatorHelpers.ts](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/evaluatorHelpers.ts#L1)

Lifecycle hooks for custom logic:

| Hook | When | Use Case |
|------|------|----------|
| `beforeAll` | Before any evaluations | Setup, data loading |
| `beforeEach` | Before each test | Per-test setup |
| `afterEach` | After each test | Logging, cleanup |
| `afterAll` | After all evaluations | Summary, notifications |

```yaml
extensions:
  - file://hooks.js:beforeAll
  - file://hooks.py:afterAll
```

**Why Hooks?** They enable integration with external systems:
- Report results to Slack
- Log to monitoring systems
- Trigger downstream processes

### Caching

[📂 src/cache.ts](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/cache.ts#L1)

LLM API calls are expensive and slow. Caching avoids redundant calls.

**Cache Key**: Based on (provider, prompt, configuration). Same inputs → same cache key.

**Cache Location**: `~/.cache/promptfoo`

**Invalidation**: Cache entries don't expire automatically. Use `--no-cache` to bypass.

**When Caching Helps**:
- Iterating on assertions (same prompts, different grading)
- Re-running evaluations after fixing bugs
- Development and testing

**When Caching Hurts**:
- Testing non-determinism (you want fresh responses)
- After model updates (cached responses are stale)

## Part 12: Key Design Decisions

### Why No Real-Time Streaming to UI?

Promptfoo uses a signal file for UI updates rather than WebSockets or polling. Why?

**Simplicity**: No need for connection management, heartbeats, or reconnection logic.

**Decoupling**: The evaluator doesn't know if any UI is watching. It just writes the signal file.

**Cross-Process**: Works whether UI is in the same process or a separate browser.

**Trade-off**: Slightly less real-time (250ms debounce) than WebSockets.

### Why Drizzle ORM?

Promptfoo uses Drizzle for database access rather than raw SQL or other ORMs.

**Type Safety**: Drizzle generates types from the schema. Typos in column names become compile errors.

**SQLite Compatibility**: Drizzle has excellent SQLite support, including features like JSON functions.

**Migration Tooling**: Drizzle Kit generates migration files automatically from schema changes.

**Trade-off**: Additional dependency, learning curve for contributors.

### Why SQLite?

See the [Persistence Layer Deep Dive](./persistence-layer-deep-dive.md) for detailed rationale. In brief: zero-configuration, single-file portability, reliability, and embedded performance.

## Part 13: Common Patterns

### Pattern: Progressive Assertion Complexity

Start simple, add complexity as needed:

```yaml
# Phase 1: Basic sanity check
assert:
  - type: not-contains
    value: "error"

# Phase 2: Add structural validation
assert:
  - type: is-json
  - type: javascript
    value: "JSON.parse(output).status === 'success'"

# Phase 3: Add semantic quality
assert:
  - type: llm-rubric
    value: "Response should be accurate and well-structured"
```

### Pattern: Provider Comparison

Compare providers on the same test suite:

```yaml
providers:
  - id: openai:gpt-4o
    label: GPT-4o
  - id: anthropic:claude-3-sonnet
    label: Claude Sonnet
  - id: openai:gpt-4o-mini
    label: GPT-4o Mini

assert:
  - type: select-best
    value: "Which response is most helpful?"
```

### Pattern: Regression Testing

Test that model changes don't break existing functionality:

```yaml
# Load tests from a file that can be version-controlled
tests: file://regression-tests.csv

# Assert minimum quality bar
defaultTest:
  threshold: 0.8
  assert:
    - type: similar
      value: "{{expected_output}}"
      threshold: 0.9
```

### Pattern: Cost Control

Ensure responses stay within budget:

```yaml
defaultTest:
  assert:
    - type: cost
      threshold: 0.05  # Max $0.05 per response
    - type: latency
      threshold: 5000  # Max 5 seconds
```

## File Reference

| File | Purpose |
|------|---------|
| [src/commands/eval.ts](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/commands/eval.ts#L1) | CLI entry point |
| [src/evaluator.ts](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/evaluator.ts#L1) | Core evaluation logic |
| [src/evaluatorHelpers.ts](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/evaluatorHelpers.ts#L1) | Prompt rendering, hooks |
| [src/assertions/index.ts](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/assertions/index.ts#L1) | Assertion dispatcher |
| [src/providers/index.ts](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/providers/index.ts#L1) | Provider loading |
| [src/providers/registry.ts](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/providers/registry.ts#L1) | Provider registry |
| [src/models/eval.ts](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/models/eval.ts#L1) | Eval model |
| [src/types/index.ts](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/types/index.ts#L1) | Type definitions |
| [src/prompts/index.ts](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/prompts/index.ts#L1) | Prompt loading |
| [src/redteam/](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/redteam) | Red team testing |
| [src/tracing/](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/tracing) | OpenTelemetry integration |
| [src/cache.ts](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/cache.ts#L1) | Caching system |
| [src/util/config/load.ts](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/util/config/load.ts#L1) | Configuration loading |
