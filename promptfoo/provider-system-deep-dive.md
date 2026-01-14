# Provider System Deep Dive

This document explains how promptfoo abstracts away the complexity of different LLM APIs through its provider system. The provider system is the interface between your evaluation configuration and the actual LLM services—whether that's OpenAI, Anthropic, a local Ollama model, or a custom HTTP endpoint.

## Table of Contents

1. [The Core Problem: API Fragmentation](#the-core-problem-api-fragmentation)
2. [Architecture Overview](#architecture-overview)
3. [The ApiProvider Interface](#the-apiprovider-interface)
   - [Why This Interface Design?](#why-this-interface-design)
   - [The CallApiFunction Signature](#the-callapifunction-signature)
   - [ProviderResponse: What Comes Back](#providerresponse-what-comes-back)
4. [The Provider Registry Pattern](#the-provider-registry-pattern)
   - [How Provider IDs Work](#how-provider-ids-work)
   - [The Factory Pattern](#the-factory-pattern)
   - [Why Test/Create Instead of a Map?](#why-testcreate-instead-of-a-map)
5. [Provider Categories](#provider-categories)
   - [Cloud LLM Providers](#cloud-llm-providers)
   - [Local Model Providers](#local-model-providers)
   - [Gateway and Proxy Providers](#gateway-and-proxy-providers)
   - [Custom and Extensible Providers](#custom-and-extensible-providers)
   - [Specialized Providers](#specialized-providers)
6. [Provider Loading: From String to Object](#provider-loading-from-string-to-object)
   - [The loadApiProvider Function](#the-loadapiprovider-function)
   - [Environment Variable Rendering](#environment-variable-rendering)
   - [Cloud Provider Resolution](#cloud-provider-resolution)
   - [File-Based Provider Configuration](#file-based-provider-configuration)
7. [Provider Configuration](#provider-configuration)
   - [The ProviderOptions Structure](#the-provideroptions-structure)
   - [Config Inheritance and Merging](#config-inheritance-and-merging)
   - [Per-Provider vs Per-Test Configuration](#per-provider-vs-per-test-configuration)
8. [A Concrete Example: OpenAI Chat Provider](#a-concrete-example-openai-chat-provider)
   - [Constructor and Initialization](#constructor-and-initialization)
   - [Building the Request Body](#building-the-request-body)
   - [Making the API Call](#making-the-api-call)
   - [Tool/Function Calling](#toolfunction-calling)
9. [The HTTP Provider: Maximum Flexibility](#the-http-provider-maximum-flexibility)
   - [When to Use HTTP Provider](#when-to-use-http-provider)
   - [Request Templating](#request-templating)
   - [Response Extraction](#response-extraction)
10. [Script-Based Providers](#script-based-providers)
    - [Python Providers](#python-providers)
    - [JavaScript Providers](#javascript-providers)
11. [Provider Lifecycle](#provider-lifecycle)
    - [Initialization](#initialization)
    - [Cleanup](#cleanup)
    - [Abort Signals](#abort-signals)
12. [Environment Variables and Secrets](#environment-variables-and-secrets)
13. [Design Trade-offs and Alternatives](#design-trade-offs-and-alternatives)
14. [Adding a New Provider](#adding-a-new-provider)
15. [File Reference](#file-reference)

## The Core Problem: API Fragmentation

Every LLM provider has its own API:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           The API Zoo                                       │
│                                                                             │
│   OpenAI              Anthropic           Google                 AWS        │
│   POST /v1/chat/      POST /v1/messages   POST /v1/models/      POST to     │
│   completions                              gemini-pro:           Bedrock    │
│                                            generateContent       runtime    │
│                                                                             │
│   {"model": "gpt-4",  {"model": "claude", {"contents": [...]}   {"prompt":  │
│    "messages": [...]}  "messages": [...]}                        "...",     │
│                                                                   "model":} │
│                                                                             │
│   Response:           Response:           Response:              Response:  │
│   choices[0].message  content[0].text     candidates[0].content  output...  │
└─────────────────────────────────────────────────────────────────────────────┘
```

**Without abstraction**, you'd need to:
1. Learn each API's request format
2. Handle each API's response format
3. Manage authentication differently for each
4. Track token usage differently
5. Handle errors differently

**The provider system** gives you a uniform interface: all providers implement `callApi(prompt)` and return a standardized `ProviderResponse`.

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           User Configuration                                │
│                                                                             │
│   providers:                                                                │
│     - openai:gpt-4                    # String shorthand                    │
│     - id: anthropic:claude-3-opus     # Object with config                  │
│       config:                                                               │
│         temperature: 0.7                                                    │
│     - file://my-provider.js           # Custom code                         │
└──────────────────────────────────┬──────────────────────────────────────────┘
                                   │
                                   ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                          loadApiProvider()                                  │
│                                                                             │
│   1. Parse the provider string/object                                       │
│   2. Render environment variables                                           │
│   3. Look up in providerMap registry                                        │
│   4. Call factory.create() to instantiate                                   │
│   5. Apply transforms, delays, labels                                       │
└──────────────────────────────────┬──────────────────────────────────────────┘
                                   │
                                   ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                           providerMap Registry                              │
│                                                                             │
│   [ { test: (id) => id.startsWith('openai:'),                               │
│       create: async (id, options) => new OpenAiChatCompletionProvider(...)  │
│     },                                                                      │
│     { test: (id) => id.startsWith('anthropic:'),                            │
│       create: async (id, options) => new AnthropicMessagesProvider(...)     │
│     },                                                                      │
│     ... 70+ provider factories ...                                          │
│   ]                                                                         │
└──────────────────────────────────┬──────────────────────────────────────────┘
                                   │
                                   ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                           ApiProvider Instance                              │
│                                                                             │
│   {                                                                         │
│     id: () => "openai:gpt-4",                                               │
│     callApi: async (prompt, context) => { ... make HTTP call ... },         │
│     label: "My GPT-4",                                                      │
│     config: { temperature: 0.7 },                                           │
│     delay: 1000,                                                            │
│     transform: "output.toUpperCase()",                                      │
│   }                                                                         │
└─────────────────────────────────────────────────────────────────────────────┘
```

[📂 src/providers/index.ts](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/providers/index.ts)
[📂 src/providers/registry.ts](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/providers/registry.ts)

## The ApiProvider Interface

At the heart of the system is the `ApiProvider` interface—the contract that every provider must fulfill:

```typescript
interface ApiProvider {
  // Required: unique identifier like "openai:gpt-4" or "anthropic:claude-3-opus"
  id: () => string;
  
  // Required: the main method to call the LLM
  callApi: CallApiFunction;
  
  // Optional: for embedding models
  callEmbeddingApi?: (input: string) => Promise<ProviderEmbeddingResponse>;
  
  // Optional: for classification models
  callClassificationApi?: (prompt: string) => Promise<ProviderClassificationResponse>;
  
  // Optional: provider-specific configuration
  config?: any;
  
  // Optional: delay between API calls for rate limiting
  delay?: number;
  
  // Optional: human-readable name for display
  label?: string;
  
  // Optional: transform to apply to output
  transform?: string;
  
  // Optional: cleanup resources on abort
  cleanup?: () => void | Promise<void>;
}
```

[📂 src/types/providers.ts](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/types/providers.ts#L94-L112)

### Why This Interface Design?

**Minimal required surface**: Only `id()` and `callApi()` are required. This makes it trivial to create a custom provider—just implement two things.

**Optional capabilities**: Not every model supports embeddings or classification. By making these optional, providers only implement what they support.

**Lifecycle hooks**: The `cleanup()` method allows providers to release resources (close WebSocket connections, stop background processes) when evaluation is aborted.

**Metadata separation**: `config`, `delay`, `label`, and `transform` are metadata that the evaluator uses but aren't part of the core "call an LLM" logic.

### The CallApiFunction Signature

```typescript
type CallApiFunction = {
  (
    prompt: string,                    // The rendered prompt to send
    context?: CallApiContextParams,    // Runtime context (vars, original provider, etc.)
    options?: CallApiOptionsParams,    // Options like abort signal
  ): Promise<ProviderResponse>;
  label?: string;                      // Optional label for the function itself
};
```

The **context** parameter is crucial—it provides:
- `vars`: Template variables for this test case
- `prompt`: The original prompt object (with config)
- `originalProvider`: Reference to the provider (useful for multi-turn)
- `test`: The current test case
- `traceparent`: OpenTelemetry trace context
- `evaluationId`, `testCaseId`: For correlation

### ProviderResponse: What Comes Back

Every `callApi()` must return a `ProviderResponse`:

```typescript
interface ProviderResponse {
  // The LLM's output (string or structured)
  output?: string | any;
  
  // Error message if the call failed
  error?: string;
  
  // Was this served from cache?
  cached?: boolean;
  
  // Estimated cost in USD
  cost?: number;
  
  // Response time
  latencyMs?: number;
  
  // Token consumption
  tokenUsage?: {
    total: number;
    prompt: number;
    completion: number;
    cached?: number;
  };
  
  // Raw response for debugging
  raw?: string | any;
  
  // Why the model stopped generating
  finishReason?: string;
  
  // Audio/video attachments
  audio?: { data?: string; transcript?: string; ... };
  video?: { url?: string; format?: string; ... };
  
  // Guardrail flags
  guardrails?: { flagged?: boolean; reason?: string; };
  
  // HTTP metadata for debugging
  metadata?: { http?: { status: number; headers: Record<string, string>; }; };
}
```

[📂 src/types/providers.ts](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/types/providers.ts#L137-L199)

**Design philosophy**: The response structure is permissive. Providers fill in what they can. The evaluator handles missing fields gracefully.

## The Provider Registry Pattern

### How Provider IDs Work

Provider IDs follow a `prefix:suffix` convention:

```
openai:gpt-4                    → OpenAI's GPT-4 model
openai:chat:gpt-4-turbo         → Explicit chat endpoint
anthropic:claude-3-opus         → Anthropic Claude
anthropic:messages:claude-3     → Explicit messages API
ollama:llama3                   → Local Ollama model
bedrock:anthropic.claude-v2     → AWS Bedrock (Anthropic model)
http://localhost:8080           → Generic HTTP endpoint
file://my-provider.js           → Custom JavaScript provider
python:my_provider.py           → Custom Python provider
```

The prefix determines which factory handles the provider. The suffix typically specifies the model or additional configuration.

### The Factory Pattern

The registry is an array of factory objects:

```typescript
interface ProviderFactory {
  test: (providerPath: string) => boolean;    // Does this factory handle this ID?
  create: (
    providerPath: string,
    providerOptions: ProviderOptions,
    context: LoadApiProviderContext,
  ) => Promise<ApiProvider>;                   // Create the provider instance
}

export const providerMap: ProviderFactory[] = [
  // Script-based providers (must come first for file extensions)
  createScriptBasedProviderFactory('exec', null, ScriptCompletionProvider),
  createScriptBasedProviderFactory('python', 'py', PythonProvider),
  
  // Cloud providers
  {
    test: (id) => id.startsWith('openai:'),
    create: async (id, options) => {
      const [_, modelType, modelName] = id.split(':');
      if (modelType === 'chat') {
        return new OpenAiChatCompletionProvider(modelName, options);
      }
      // ... handle other model types
    },
  },
  {
    test: (id) => id.startsWith('anthropic:'),
    create: async (id, options) => new AnthropicMessagesProvider(...),
  },
  // ... 70+ more factories
];
```

[📂 src/providers/registry.ts](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/providers/registry.ts#L131-L1585)

### Why Test/Create Instead of a Map?

**Alternative considered**: A simple `Record<string, ProviderClass>` lookup:

```typescript
// NOT what promptfoo does
const providers = {
  'openai': OpenAiProvider,
  'anthropic': AnthropicProvider,
};
const Provider = providers[prefix];
```

**Why this doesn't work**:

1. **Prefix matching isn't enough**: `openai:chat:gpt-4` vs `openai:gpt-4` require different handling
2. **Dynamic imports**: Some providers are lazily imported to avoid loading unused dependencies
3. **Complex routing**: `anthropic:messages:claude-3` vs `anthropic:completion:claude-2` route to different classes
4. **Alias handling**: `google:`, `palm:`, and `vertex:` might all route to similar providers

The factory pattern with `test()` functions allows arbitrary matching logic.

## Provider Categories

### Cloud LLM Providers

These connect to hosted LLM APIs:

| Provider | Example ID | Notes |
|----------|-----------|-------|
| OpenAI | `openai:gpt-4o` | Chat, completion, embedding, moderation, image, video |
| Anthropic | `anthropic:claude-3-opus` | Messages and legacy completion APIs |
| Google | `google:gemini-pro` | AI Studio and Vertex AI |
| AWS Bedrock | `bedrock:anthropic.claude-v2` | Access multiple models through AWS |
| Azure OpenAI | `azure:chat:my-deployment` | Enterprise OpenAI deployments |
| Groq | `groq:llama-3.3-70b` | Fast inference |
| Mistral | `mistral:mistral-large` | Mistral AI models |
| Cohere | `cohere:command-r-plus` | Cohere models |

### Local Model Providers

For running models locally:

| Provider | Example ID | Notes |
|----------|-----------|-------|
| Ollama | `ollama:llama3` | Popular local model runner |
| llama.cpp | `llama:./model.gguf` | Direct GGUF model loading |
| LocalAI | `localai:chat:gpt4all` | LocalAI server |
| LM Studio | Uses OpenAI-compatible endpoint | Via HTTP provider |

### Gateway and Proxy Providers

Route through intermediaries:

| Provider | Example ID | Notes |
|----------|-----------|-------|
| OpenRouter | `openrouter:anthropic/claude-3` | Multi-provider gateway |
| Portkey | `portkey:gpt-4` | Observability gateway |
| Helicone | `helicone:openai/gpt-4` | Logging gateway |
| LiteLLM | `litellm:gpt-4` | Unified API for multiple providers |
| Cloudflare Gateway | `cloudflare-gateway:...` | Edge caching and routing |

### Custom and Extensible Providers

For your own integrations:

| Provider | Example ID | Notes |
|----------|-----------|-------|
| HTTP | `https://api.example.com/v1/chat` | Raw HTTP requests |
| WebSocket | `wss://api.example.com/ws` | WebSocket connections |
| JavaScript | `file://provider.js` | Custom JS/TS code |
| Python | `python:provider.py` | Custom Python code |
| Webhook | `webhook:https://...` | Webhook-based |

### Specialized Providers

For specific use cases:

| Provider | Example ID | Notes |
|----------|-----------|-------|
| Browser | `browser` | Headless browser interactions |
| MCP | `mcp:server-name` | Model Context Protocol |
| ElevenLabs | `elevenlabs:tts:...` | Text-to-speech, speech-to-text |
| Replicate | `replicate:model/version` | Replicate-hosted models |
| Simulated User | `promptfoo:simulated-user` | Multi-turn conversation simulation |

## Provider Loading: From String to Object

### The loadApiProvider Function

This is the main entry point for creating provider instances:

```typescript
export async function loadApiProvider(
  providerPath: string,                    // e.g., "openai:gpt-4"
  context: LoadApiProviderContext = {},    // basePath, env, options
): Promise<ApiProvider> {
  const { options = {}, basePath, env } = context;
  
  // 1. Merge environment overrides
  const mergedEnv = { ...env, ...options.env };
  
  // 2. Render environment variable templates in config
  const renderedConfig = renderEnvOnlyInObject(options.config, mergedEnv);
  
  // 3. Render the provider path itself (for dynamic IDs)
  const renderedProviderPath = nunjucks.renderString(providerPath, { env: mergedEnv });
  
  // 4. Handle special cases (cloud providers, file:// references)
  if (isCloudProvider(renderedProviderPath)) {
    return await loadCloudProvider(renderedProviderPath, context);
  }
  if (renderedProviderPath.endsWith('.yaml') || renderedProviderPath.endsWith('.json')) {
    return await loadProviderFromFile(renderedProviderPath, context);
  }
  
  // 5. Find matching factory in registry
  for (const factory of providerMap) {
    if (factory.test(renderedProviderPath)) {
      const provider = await factory.create(renderedProviderPath, providerOptions, context);
      
      // 6. Apply common properties
      provider.transform = options.transform;
      provider.delay = options.delay;
      provider.label ||= options.label;
      
      return provider;
    }
  }
  
  throw new Error(`Could not identify provider: ${providerPath}`);
}
```

[📂 src/providers/index.ts](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/providers/index.ts#L31-L177)

### Environment Variable Rendering

Provider configurations can reference environment variables:

```yaml
providers:
  - id: openai:gpt-4
    config:
      apiBaseUrl: "{{ env.OPENAI_API_BASE }}"
      apiKey: "{{ env.MY_OPENAI_KEY }}"
```

These are rendered at **load time**, not at call time. This is important because:
1. Constructors need real values to set up HTTP clients
2. Secrets should be resolved before logging (to avoid accidental exposure)
3. Runtime templates (`{{ vars.* }}`) are preserved for per-test customization

### Cloud Provider Resolution

Promptfoo Cloud allows storing provider configurations remotely:

```yaml
providers:
  - promptfoo://provider/abc123  # Cloud-stored configuration
```

When loading:
1. Fetch the configuration from Promptfoo Cloud API
2. Merge with any local overrides
3. Recursively call `loadApiProvider` with the resolved ID

This enables sharing provider configurations across teams without exposing API keys in config files.

### File-Based Provider Configuration

For complex configurations, you can define providers in separate files:

```yaml
# promptfooconfig.yaml
providers:
  - file://providers/my-gpt4.yaml
```

```yaml
# providers/my-gpt4.yaml
id: openai:gpt-4
config:
  temperature: 0.7
  max_tokens: 2000
  tools:
    - file://tools/search.json
```

The file is loaded, merged with context, and the resulting config is used to instantiate the provider.

## Provider Configuration

### The ProviderOptions Structure

```typescript
interface ProviderOptions {
  id?: string;           // Override the provider ID
  label?: string;        // Human-readable display name
  config?: any;          // Provider-specific configuration
  prompts?: string[];    // Which prompts this provider handles
  transform?: string;    // Output transformation
  delay?: number;        // Delay between calls (ms)
  env?: EnvOverrides;    // Environment variable overrides
  inputs?: Inputs;       // Input configuration
}
```

### Config Inheritance and Merging

Configuration can come from multiple sources, with later sources overriding earlier ones:

```
Environment Variables (lowest priority)
       ↓
Cloud Provider Configuration
       ↓
File-Based Configuration
       ↓
Inline Configuration in promptfooconfig.yaml
       ↓
Per-Test Provider Configuration (highest priority)
```

Example:

```yaml
# Environment: OPENAI_TEMPERATURE=0.5

providers:
  - id: openai:gpt-4
    config:
      temperature: 0.7  # Overrides environment

tests:
  - vars:
      question: "What is 2+2?"
    provider:
      id: openai:gpt-4
      config:
        temperature: 0.9  # Overrides provider-level config for this test
```

### Per-Provider vs Per-Test Configuration

**Provider-level config**: Applied to all tests using this provider

```yaml
providers:
  - id: openai:gpt-4
    config:
      temperature: 0.7
      max_tokens: 1000
```

**Test-level config**: Merged with provider config for specific tests

```yaml
tests:
  - vars:
      input: "Creative writing prompt"
    provider:
      id: openai:gpt-4
      config:
        temperature: 1.0  # Higher creativity for this test
```

**Prompt-level config**: Applied when rendering this prompt

```yaml
prompts:
  - label: concise-prompt
    raw: "Be concise: {{input}}"
    config:
      max_tokens: 100  # Enforce brevity
```

## A Concrete Example: OpenAI Chat Provider

Let's trace through how the OpenAI Chat provider works:

### Constructor and Initialization

```typescript
export class OpenAiChatCompletionProvider extends OpenAiGenericProvider {
  config: OpenAiCompletionOptions;
  private mcpClient: MCPClient | null = null;
  
  constructor(
    modelName: string,
    options: { config?: OpenAiCompletionOptions; id?: string; env?: EnvOverrides } = {},
  ) {
    super(modelName, options);
    this.config = options.config || {};
    
    // If MCP (Model Context Protocol) is enabled, initialize it
    if (this.config.mcp?.enabled) {
      this.initializationPromise = this.initializeMCP();
    }
  }
}
```

[📂 src/providers/openai/chat.ts](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/providers/openai/chat.ts#L36-L59)

**Design notes**:
- Extends a generic OpenAI provider (shared logic for API URL, auth)
- MCP initialization is async and stored as a promise (lazy)
- Config is stored for use in `callApi`

### Building the Request Body

```typescript
async getOpenAiBody(prompt: string, context?: CallApiContextParams) {
  const config = { ...this.config, ...context?.prompt?.config };
  
  // Parse the prompt into messages
  const messages = parseChatPrompt(prompt, [{ role: 'user', content: prompt }]);
  
  // Handle reasoning models (o1, o3) differently
  const isReasoningModel = this.isReasoningModel();
  const maxTokens = isReasoningModel ? undefined : (config.max_tokens ?? 1024);
  const temperature = this.supportsTemperature() ? (config.temperature ?? 0) : undefined;
  
  // Merge MCP tools with configured tools
  const mcpTools = this.mcpClient ? transformMCPToolsToOpenAi(...) : [];
  const fileTools = await maybeLoadToolsFromExternalFile(config.tools);
  const allTools = [...mcpTools, ...fileTools];
  
  return {
    model: this.modelName,
    messages,
    max_tokens: maxTokens,
    temperature,
    tools: allTools.length > 0 ? allTools : undefined,
    // ... other parameters
  };
}
```

[📂 src/providers/openai/chat.ts](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/providers/openai/chat.ts#L198-L288)

**Key patterns**:
- Config from provider is merged with config from prompt
- Model-specific logic (reasoning models don't support temperature)
- External file loading for tools
- MCP integration for dynamic tools

### Making the API Call

```typescript
async callApi(prompt: string, context?: CallApiContextParams): Promise<ProviderResponse> {
  const body = await this.getOpenAiBody(prompt, context);
  
  const response = await fetchWithCache(
    `${this.getApiUrl()}/chat/completions`,
    {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${this.getApiKey()}`,
      },
      body: JSON.stringify(body),
    },
    REQUEST_TIMEOUT_MS,
  );
  
  // Extract content from response
  const data = response.data;
  const message = data.choices[0].message;
  
  return {
    output: message.content,
    tokenUsage: getTokenUsage(data.usage),
    cost: calculateOpenAICost(this.modelName, data.usage),
    cached: response.cached,
    finishReason: normalizeFinishReason(data.choices[0].finish_reason),
  };
}
```

**Patterns**:
- Uses `fetchWithCache` for automatic caching
- Extracts standardized fields from OpenAI-specific response
- Calculates cost based on model and token usage

### Tool/Function Calling

When the model requests a tool call:

```typescript
// In the response handling
if (message.tool_calls) {
  for (const toolCall of message.tool_calls) {
    const functionName = toolCall.function.name;
    const args = toolCall.function.arguments;
    
    // Try MCP first
    if (this.mcpClient?.hasTool(functionName)) {
      const result = await this.mcpClient.callTool(functionName, JSON.parse(args));
      messages.push({ role: 'tool', content: result, tool_call_id: toolCall.id });
    }
    // Then try function callbacks
    else if (this.config.functionToolCallbacks?.[functionName]) {
      const result = await this.executeFunctionCallback(functionName, args);
      messages.push({ role: 'tool', content: result, tool_call_id: toolCall.id });
    }
    // Otherwise return the tool call as output
    else {
      return { output: message.tool_calls };
    }
  }
  
  // Continue the conversation with tool results
  return this.callApi(newPromptWithToolResults, context);
}
```

## The HTTP Provider: Maximum Flexibility

For APIs that don't have dedicated providers, the HTTP provider allows direct HTTP configuration:

### When to Use HTTP Provider

- Internal/proprietary LLM endpoints
- Self-hosted models behind a REST API
- APIs with non-standard authentication
- Prototyping new integrations

### Request Templating

```yaml
providers:
  - id: https://api.mycompany.com/v1/generate
    config:
      method: POST
      headers:
        Authorization: "Bearer {{ env.MY_API_KEY }}"
        Content-Type: application/json
      body:
        prompt: "{{ prompt }}"
        max_tokens: 1000
        user_id: "{{ vars.user_id }}"
```

The `{{ prompt }}` and `{{ vars.* }}` are rendered at call time using Nunjucks templating.

### Response Extraction

```yaml
providers:
  - id: https://api.mycompany.com/v1/generate
    config:
      responseParser: "json.data.generated_text"
      # or for complex extraction:
      responseParser: |
        (json) => {
          return {
            output: json.result.text,
            tokenUsage: { total: json.usage.tokens }
          };
        }
```

[📂 src/providers/http.ts](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/providers/http.ts)

## Script-Based Providers

### Python Providers

```yaml
providers:
  - python:my_provider.py
```

```python
# my_provider.py
def call_api(prompt, options, context):
    """
    Args:
        prompt: The rendered prompt string
        options: Provider options from config
        context: Runtime context (vars, etc.)
    
    Returns:
        dict with 'output' and optionally 'tokenUsage', 'cost', 'error'
    """
    import my_custom_llm
    
    response = my_custom_llm.generate(prompt)
    
    return {
        "output": response.text,
        "tokenUsage": {"total": response.tokens}
    }
```

### JavaScript Providers

```yaml
providers:
  - file://my_provider.js
```

```javascript
// my_provider.js
module.exports = class MyProvider {
  id() {
    return 'my-custom-provider';
  }
  
  async callApi(prompt, context) {
    const response = await fetch('https://my-api.com/generate', {
      method: 'POST',
      body: JSON.stringify({ prompt }),
    });
    const data = await response.json();
    
    return {
      output: data.text,
      tokenUsage: { total: data.tokens },
    };
  }
};
```

## Provider Lifecycle

### Initialization

Providers can have async initialization:

```typescript
class MyProvider implements ApiProvider {
  private client: MyClient | null = null;
  private initPromise: Promise<void> | null = null;
  
  constructor(options) {
    // Start async initialization
    this.initPromise = this.initialize();
  }
  
  private async initialize() {
    this.client = await MyClient.connect(this.config.endpoint);
  }
  
  async callApi(prompt) {
    // Wait for initialization before first call
    if (this.initPromise) {
      await this.initPromise;
      this.initPromise = null;
    }
    
    return this.client.generate(prompt);
  }
}
```

### Cleanup

When evaluation is aborted (timeout, user interrupt), providers can clean up:

```typescript
async cleanup(): Promise<void> {
  if (this.mcpClient) {
    await this.mcpClient.cleanup();
    this.mcpClient = null;
  }
  if (this.websocket) {
    this.websocket.close();
    this.websocket = null;
  }
}
```

[📂 src/providers/openai/chat.ts](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/providers/openai/chat.ts#L66-L72)

### Abort Signals

Providers receive abort signals for cancellation:

```typescript
async callApi(prompt, context, options) {
  const { abortSignal } = options || {};
  
  const response = await fetch(url, {
    signal: abortSignal,  // Pass to fetch for network cancellation
    // ...
  });
  
  return { output: response.data };
}
```

## Environment Variables and Secrets

Common patterns for API key management:

```yaml
# Option 1: Environment variable reference
providers:
  - id: openai:gpt-4
    config:
      apiKey: "{{ env.OPENAI_API_KEY }}"

# Option 2: Provider-specific environment variable (auto-detected)
# Just set OPENAI_API_KEY in your environment

# Option 3: Config-level override
providers:
  - id: openai:gpt-4
    config:
      apiKeyEnvar: "MY_CUSTOM_OPENAI_KEY"  # Use different env var
```

**Security note**: Never hardcode API keys in config files. Use environment variables or secret management systems.

## Design Trade-offs and Alternatives

### Why Not a Single Universal Provider?

**Alternative**: One `UniversalLLMProvider` that handles all APIs

**Problems**:
1. Each API has unique features (tools, function calling, streaming protocols)
2. Authentication differs (API keys, OAuth, AWS signatures)
3. Request/response formats are incompatible
4. Would become unmaintainably complex

### Why Inheritance Over Composition?

Many providers extend base classes:

```
OpenAiGenericProvider
  ├── OpenAiChatCompletionProvider
  ├── OpenAiCompletionProvider
  ├── OpenAiEmbeddingProvider
  └── OpenAiModerationProvider
```

**Trade-off**: Inheritance can lead to deep hierarchies. Promptfoo keeps it shallow (1-2 levels) and uses composition for cross-cutting concerns (caching, tracing).

### Why Factory Pattern Instead of Dependency Injection?

**Alternative**: DI container that manages provider lifecycles

**Why factories**:
1. Providers are short-lived (one evaluation)
2. No complex inter-dependencies
3. Simpler mental model for users writing custom providers
4. Lazy loading without DI framework overhead

## Adding a New Provider

To add a new provider:

1. **Create the provider class**:

```typescript
// src/providers/myservice.ts
export class MyServiceProvider implements ApiProvider {
  constructor(modelName: string, options: ProviderOptions) {
    this.modelName = modelName;
    this.config = options.config || {};
  }
  
  id() {
    return `myservice:${this.modelName}`;
  }
  
  async callApi(prompt: string, context?: CallApiContextParams): Promise<ProviderResponse> {
    // Implementation
  }
}
```

2. **Add to the registry**:

```typescript
// src/providers/registry.ts
{
  test: (id) => id.startsWith('myservice:'),
  create: async (id, options) => {
    const modelName = id.split(':')[1];
    return new MyServiceProvider(modelName, options);
  },
},
```

3. **Export if needed**:

```typescript
// src/providers/index.ts
export { MyServiceProvider } from './myservice';
```

## File Reference

| File | Purpose |
|------|---------|
| [src/providers/index.ts](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/providers/index.ts) | `loadApiProvider` and provider resolution logic |
| [src/providers/registry.ts](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/providers/registry.ts) | Factory registry with all provider mappings |
| [src/types/providers.ts](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/types/providers.ts) | TypeScript interfaces for providers |
| [src/providers/openai/chat.ts](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/providers/openai/chat.ts) | OpenAI Chat Completion implementation |
| [src/providers/http.ts](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/providers/http.ts) | Generic HTTP provider |
| [src/providers/anthropic/messages.ts](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/providers/anthropic/messages.ts) | Anthropic Messages API |
| [src/providers/ollama.ts](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/providers/ollama.ts) | Local Ollama provider |
| [src/providers/pythonCompletion.ts](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/providers/pythonCompletion.ts) | Python script provider |

## Summary

The provider system is promptfoo's abstraction layer over LLM APIs:

1. **Uniform interface**: All providers implement `callApi()` returning `ProviderResponse`
2. **Registry pattern**: Provider IDs are matched against factories in `providerMap`
3. **Configuration merging**: Environment → file → inline → per-test
4. **Extensibility**: HTTP provider and script providers for custom integrations
5. **Lifecycle management**: Initialization, cleanup, and abort signal handling

The design prioritizes:
- **Ease of use**: Simple string IDs for common cases
- **Flexibility**: Full configurability for complex scenarios
- **Extensibility**: Easy to add new providers
- **Reliability**: Proper resource cleanup and error handling
