# Prompt Templates Deep Dive

This document explores how promptfoo handles prompt templates—from simple string interpolation to complex multi-file prompt functions. Understanding the prompt system is essential because prompts are the primary input to your LLM, and how they're processed directly affects your evaluation results.

## Table of Contents

1. [The Fundamental Challenge: Dynamic Prompts](#the-fundamental-challenge-dynamic-prompts)
2. [What Is a Prompt in Promptfoo?](#what-is-a-prompt-in-promptfoo)
3. [The Prompt Lifecycle](#the-prompt-lifecycle)
4. [Prompt Sources: Where Prompts Come From](#prompt-sources-where-prompts-come-from)
   - [Inline Strings](#inline-strings)
   - [Text Files](#text-files)
   - [JSON Files](#json-files)
   - [YAML Files](#yaml-files)
   - [JavaScript Functions](#javascript-functions)
   - [Python Functions](#python-functions)
   - [Jinja2 Templates](#jinja2-templates)
   - [Markdown Files](#markdown-files)
   - [External Prompt Services](#external-prompt-services)
5. [The Nunjucks Template Engine](#the-nunjucks-template-engine)
   - [Why Nunjucks?](#why-nunjucks)
   - [Basic Variable Substitution](#basic-variable-substitution)
   - [Filters](#filters)
   - [Conditionals and Loops](#conditionals-and-loops)
   - [Custom Filters](#custom-filters)
   - [The Environment Object](#the-environment-object)
6. [Variable Resolution](#variable-resolution)
   - [Static Variables](#static-variables)
   - [File-Based Variables](#file-based-variables)
   - [Computed Variables](#computed-variables)
   - [Variable Expansion (Cartesian Product)](#variable-expansion-cartesian-product)
7. [JSON Prompts: A Special Case](#json-prompts-a-special-case)
8. [Prompt Functions: Maximum Flexibility](#prompt-functions-maximum-flexibility)
   - [When to Use Prompt Functions](#when-to-use-prompt-functions)
   - [JavaScript Prompt Functions](#javascript-prompt-functions)
   - [Python Prompt Functions](#python-prompt-functions)
   - [Returning Config from Prompt Functions](#returning-config-from-prompt-functions)
9. [Provider-Specific Prompts](#provider-specific-prompts)
10. [Special Variables](#special-variables)
11. [Security Considerations](#security-considerations)
    - [The skipRenderVars Mechanism](#the-skiprendervars-mechanism)
    - [Environment Variable Access Control](#environment-variable-access-control)
12. [Prompt Labeling and Identification](#prompt-labeling-and-identification)
13. [Design Decisions and Trade-offs](#design-decisions-and-trade-offs)
14. [Common Patterns](#common-patterns)
15. [File Reference](#file-reference)

## The Fundamental Challenge: Dynamic Prompts

Consider a simple evaluation scenario: you want to test how your LLM handles translation requests across different languages and texts. A naive approach would be to write out every combination:

```yaml
prompts:
  - "Translate 'Hello' to French"
  - "Translate 'Hello' to Spanish"
  - "Translate 'Hello' to German"
  - "Translate 'Goodbye' to French"
  - "Translate 'Goodbye' to Spanish"
  # ... 100 more combinations
```

This approach has serious problems:

1. **Combinatorial explosion**: With 10 texts and 10 languages, you need 100 prompts. With 100 texts and 20 languages, you need 2,000 prompts.

2. **Maintenance nightmare**: Changing the prompt structure (e.g., adding "Please" at the start) requires editing every single prompt.

3. **Separation of concerns**: The prompt *structure* (what you're asking the LLM to do) is mixed with the *data* (specific texts and languages).

The solution is **templating**: write the prompt structure once, and fill in the data dynamically.

```yaml
prompts:
  - "Translate '{{text}}' to {{language}}"

tests:
  - vars:
      text: Hello
      language: French
  - vars:
      text: Hello
      language: Spanish
  # ... data-driven tests
```

This is what the prompt template system enables—and it goes far beyond simple string substitution.

## What Is a Prompt in Promptfoo?

At its core, a `Prompt` in promptfoo is a data structure with these key fields:

```typescript
interface Prompt {
  // The template content (before variable substitution)
  raw: string;
  
  // Human-readable identifier for display and matching
  label: string;
  
  // Optional unique identifier
  id?: string;
  
  // Optional function for dynamic prompt generation
  function?: PromptFunction;
  
  // Configuration that merges with provider config
  config?: any;
}
```

[📂 src/types/prompts.ts](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/types/prompts.ts#L46-L55)

**Why this structure?**

- **`raw`**: The unrendered template. This is what you write in your config. It may contain Nunjucks syntax like `{{variable}}` or `{% if condition %}`.

- **`label`**: Used for display in the UI and for matching when you want specific providers to use specific prompts. Without explicit labels, promptfoo generates one from the content.

- **`function`**: For prompts that can't be expressed as static templates, you can provide a function that generates the prompt dynamically at runtime.

- **`config`**: Prompt-specific configuration that overrides or extends provider configuration. For example, you might want different `max_tokens` for different prompts.

## The Prompt Lifecycle

Understanding when each transformation happens is crucial for debugging:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           PROMPT LIFECYCLE                                  │
│                                                                             │
│  1. LOADING PHASE (at startup)                                              │
│     ─────────────────────────                                               │
│     • Read prompt source (string, file, glob pattern)                       │
│     • Determine file type (txt, json, js, py, etc.)                         │
│     • Process file-specific format (parse JSON, import JS module, etc.)     │
│     • Create Prompt objects with raw templates                              │
│                                                                             │
│  2. EXPANSION PHASE (after loading)                                         │
│     ────────────────────────────                                            │
│     • Cross-product: prompts × providers × test cases                       │
│     • Each combination becomes a RunEvalOptions                             │
│                                                                             │
│  3. RENDERING PHASE (per test case, at execution time)                      │
│     ──────────────────────────────────────────────────                      │
│     • Load file:// variables from disk                                      │
│     • Execute prompt function (if present)                                  │
│     • Resolve variable mappings                                             │
│     • Fetch external prompts (Langfuse, Portkey, Helicone)                  │
│     • Apply Nunjucks templating                                             │
│     • Return final string to send to provider                               │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

**Why separate phases?**

The loading phase happens once at startup—expensive operations like reading files and importing modules are done upfront. The rendering phase happens for each test case, so it needs to be fast. By loading the structure early and only doing variable substitution at runtime, promptfoo keeps evaluation fast even with thousands of test cases.

[📂 src/prompts/index.ts](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/prompts/index.ts)
[📂 src/evaluatorHelpers.ts](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/evaluatorHelpers.ts#L220-L496)

## Prompt Sources: Where Prompts Come From

Promptfoo supports an unusually wide variety of prompt sources. This flexibility exists because different teams have different workflows—some prefer prompts in version control, others in prompt management systems, others generated dynamically.

### Inline Strings

The simplest form—the prompt is right there in the config:

```yaml
prompts:
  - "Translate '{{text}}' to {{language}}"
  - "You are a translator. Translate the following: {{text}} → {{language}}"
```

**When to use**: Quick experiments, simple prompts, or when you want everything in one file.

**Trade-off**: Long prompts make the config file hard to read.

### Text Files

Move the prompt content to a separate file:

```yaml
prompts:
  - file://prompts/translator.txt
```

```text
# prompts/translator.txt
You are a professional translator with expertise in {{language}}.

Please translate the following text accurately, preserving tone and meaning:

Text: {{text}}

Translation:
```

**How it works**: Promptfoo reads the file content and treats it as the raw template string.

**When to use**: Long prompts, prompts you want to version separately, or when non-technical team members edit prompts.

[📂 src/prompts/processors/text.ts](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/prompts/processors/text.ts)

### JSON Files

For chat-format prompts (message arrays):

```yaml
prompts:
  - file://prompts/chat-translator.json
```

```json
[
  {
    "role": "system",
    "content": "You are a translator specializing in {{language}}."
  },
  {
    "role": "user", 
    "content": "Translate: {{text}}"
  }
]
```

**Why JSON?**: Many LLM APIs (OpenAI, Anthropic) expect message arrays. JSON prompts let you define multi-turn conversations or system prompts directly.

**Special handling**: Promptfoo detects that the raw prompt is valid JSON and recursively renders variables *within* each string value, then re-serializes to JSON. This means `{{text}}` inside a JSON string gets substituted correctly.

[📂 src/prompts/processors/json.ts](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/prompts/processors/json.ts)

### YAML Files

Similar to JSON, but more human-readable:

```yaml
prompts:
  - file://prompts/chat-translator.yaml
```

```yaml
# prompts/chat-translator.yaml
- role: system
  content: You are a translator specializing in {{language}}.
- role: user
  content: "Translate: {{text}}"
```

**Why YAML over JSON?**: YAML supports multi-line strings naturally, making it better for long prompts. It also supports comments.

### JavaScript Functions

For prompts that need logic:

```yaml
prompts:
  - file://prompts/dynamic.js
```

```javascript
// prompts/dynamic.js
module.exports = async function(context) {
  const { vars, provider } = context;
  
  // Logic that can't be expressed in templates
  const tone = vars.formality === 'high' ? 'formal' : 'casual';
  const length = vars.text.length > 100 ? 'concise' : 'detailed';
  
  return `Translate to ${vars.language} in a ${tone} tone. Be ${length}.
  
Text: ${vars.text}`;
};
```

**Why functions?**

1. **Conditional logic**: Choose different prompt structures based on variables
2. **External data**: Fetch context from databases or APIs
3. **Complex formatting**: Build prompts programmatically
4. **Provider-aware**: Adjust based on which provider is being used

**The function receives**:
- `vars`: The test case variables
- `provider`: The current provider (id, label)
- `config`: Any prompt-level config

[📂 src/prompts/processors/javascript.ts](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/prompts/processors/javascript.ts)

### Python Functions

Same concept as JavaScript, but in Python:

```yaml
prompts:
  - file://prompts/dynamic.py:generate_prompt
```

```python
# prompts/dynamic.py
def generate_prompt(context):
    vars = context['vars']
    provider = context['provider']
    
    # Python-native string manipulation
    text = vars['text'].strip()
    language = vars['language'].capitalize()
    
    return f"""Translate to {language}:

{text}
"""
```

**When to prefer Python over JavaScript**:
- Your team knows Python better
- You need Python-specific libraries (pandas, numpy)
- Integration with ML workflows

### Jinja2 Templates

For those who prefer Jinja2 syntax over Nunjucks:

```yaml
prompts:
  - file://prompts/template.j2
```

```jinja2
{# prompts/template.j2 #}
{% if vars.formal %}
Please translate the following text to {{ vars.language }}:
{% else %}
Hey, translate this to {{ vars.language }}:
{% endif %}

{{ vars.text }}
```

**Why offer Jinja2?**: Jinja2 is the dominant templating language in the Python ecosystem. Teams using promptfoo alongside Python ML tools may prefer Jinja2's familiar syntax.

### Markdown Files

For documentation-friendly prompts:

```yaml
prompts:
  - file://prompts/translator.md
```

```markdown
# Translation Request

You are translating from English to **{{language}}**.

## Instructions

1. Preserve the original meaning
2. Maintain appropriate formality
3. Do not add explanations

## Text to Translate

{{text}}
```

**Why Markdown?**: Some LLMs respond well to structured Markdown prompts. Also, Markdown files render nicely in GitHub, making them self-documenting.

### External Prompt Services

Promptfoo integrates with prompt management systems:

```yaml
prompts:
  - langfuse://translator-prompt:v2
  - portkey://my-prompt-id
  - helicone://prompt-123:1.0
```

**Why external services?**

1. **Version management**: Track prompt changes without code commits
2. **A/B testing**: Route to different prompt versions
3. **Non-technical access**: Let product managers edit prompts in a UI
4. **Analytics**: Track which prompts perform best

The prompt is fetched at render time, meaning changes in the external system are reflected immediately.

[📂 src/evaluatorHelpers.ts](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/evaluatorHelpers.ts#L393-L465)

## The Nunjucks Template Engine

At the heart of prompt rendering is [Nunjucks](https://mozilla.github.io/nunjucks/)—a JavaScript templating engine inspired by Jinja2.

### Why Nunjucks?

**Alternatives considered**:

1. **Plain string interpolation** (e.g., `"Hello ${name}"`): Too limited. No conditionals, no loops, no filters.

2. **Mustache/Handlebars**: Simpler syntax but less powerful. No native filter support.

3. **EJS**: More powerful but allows arbitrary JavaScript execution—a security concern.

4. **Jinja2**: Great, but Python-only. Since promptfoo is TypeScript, native Jinja2 isn't available.

**Why Nunjucks won**:

- **Jinja2-compatible syntax**: Familiar to Python developers
- **Rich feature set**: Filters, conditionals, loops, inheritance
- **Safe by default**: Templates can't execute arbitrary code
- **Well-maintained**: Mozilla-backed project

[📂 src/util/templates.ts](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/util/templates.ts)

### Basic Variable Substitution

```
Input template: "Hello, {{name}}! You are {{age}} years old."
Variables: { name: "Alice", age: 30 }
Output: "Hello, Alice! You are 30 years old."
```

The `{{ }}` delimiters mark variable insertion points. Whitespace inside is ignored: `{{name}}`, `{{ name }}`, and `{{  name  }}` are equivalent.

### Filters

Filters transform variable values:

```
{{ name | upper }}          → "ALICE"
{{ name | lower }}          → "alice"
{{ name | capitalize }}     → "Alice"
{{ text | truncate(50) }}   → First 50 characters + "..."
{{ items | join(", ") }}    → "item1, item2, item3"
{{ data | dump }}           → JSON string of data
```

**Chaining filters**:

```
{{ name | lower | capitalize }}   → "Alice"
```

### Conditionals and Loops

```jinja2
{% if language == "French" %}
Traduisez en français:
{% elif language == "Spanish" %}
Traduzca al español:
{% else %}
Translate to {{ language }}:
{% endif %}

{{ text }}

---

{% for item in items %}
- {{ item.name }}: {{ item.value }}
{% endfor %}
```

**When to use conditionals vs. separate prompts**:

- Use conditionals for minor variations (greetings, formatting)
- Use separate prompts when the structure fundamentally differs

### Custom Filters

You can define custom filters in your config:

```yaml
nunjucksFilters:
  pluralize: |
    function(word, count) {
      return count === 1 ? word : word + 's';
    }
  formatDate: |
    function(date) {
      return new Date(date).toLocaleDateString();
    }

prompts:
  - "You have {{ count }} {{ 'item' | pluralize(count) }}."
```

**Use cases for custom filters**:
- Domain-specific formatting (currency, dates, units)
- Text transformations (anonymization, normalization)
- Complex logic encapsulation

### The Environment Object

Templates have access to environment variables via the `env` object:

```
API Key: {{ env.OPENAI_API_KEY }}
Deploy environment: {{ env.NODE_ENV }}
```

**Security consideration**: By default in self-hosted mode, `process.env` access is disabled. Only explicitly configured environment variables (from the config file's `env` section) are available.

```yaml
env:
  SAFE_VAR: "This is accessible"
  # OPENAI_API_KEY is NOT accessible unless explicitly listed
```

[📂 src/util/templates.ts](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/util/templates.ts#L31-L43)

## Variable Resolution

Variables come from test cases, but their values can be more than simple strings.

### Static Variables

The simplest form—values are literals in the test case:

```yaml
tests:
  - vars:
      name: Alice
      age: 30
      items:
        - first
        - second
```

### File-Based Variables

Variables can load content from files:

```yaml
tests:
  - vars:
      document: file://data/document.txt
      image: file://images/photo.png
      config: file://configs/settings.yaml
```

**What happens**:

1. `file://data/document.txt` → Reads file, stores text content
2. `file://images/photo.png` → Reads file, converts to base64 data URL
3. `file://configs/settings.yaml` → Parses YAML, stores as JSON string

**Why this design?**

Many evaluations involve external content (documents to summarize, images to describe). Loading these as variables keeps the test definition clean and reusable.

[📂 src/evaluatorHelpers.ts](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/evaluatorHelpers.ts#L232-L355)

### Computed Variables

Variables can be computed by scripts:

```yaml
tests:
  - vars:
      timestamp: file://scripts/timestamp.js
      summary: file://scripts/summarize.py
```

```javascript
// scripts/timestamp.js
module.exports = function(varName, prompt, vars, provider) {
  return { output: new Date().toISOString() };
};
```

**When to use**:
- Dynamic values (timestamps, random IDs)
- Derived values (computed from other variables)
- External data (database lookups, API calls)

### Variable Expansion (Cartesian Product)

When a variable is an array, promptfoo expands it into multiple test cases:

```yaml
tests:
  - vars:
      language:
        - French
        - Spanish
        - German
      text:
        - Hello
        - Goodbye
```

This creates **6 test cases**: every combination of language and text.

```
(French, Hello), (French, Goodbye),
(Spanish, Hello), (Spanish, Goodbye),
(German, Hello), (German, Goodbye)
```

**Why this exists**: Writing out all combinations is tedious. Array expansion lets you define the dimensions and get the full matrix automatically.

**Disabling expansion**: If you want an array as a literal value (not expanded), use `disableVarExpansion`:

```yaml
tests:
  - vars:
      items: ["one", "two", "three"]  # This is a list, not 3 test cases
    options:
      disableVarExpansion: true
```

## JSON Prompts: A Special Case

When the raw prompt is valid JSON, promptfoo applies special handling:

```yaml
prompts:
  - |
    [
      {"role": "system", "content": "You are a {{role}}."},
      {"role": "user", "content": "{{question}}"}
    ]
```

**The algorithm**:

1. Try to parse `raw` as JSON
2. If successful, recursively walk the structure
3. For each string value, apply Nunjucks rendering
4. Re-serialize to JSON

**Why special handling?**

Without this, you'd have to write:

```yaml
prompts:
  - '[{"role": "system", "content": "You are a {{role}}."}, ...]'
```

The entire JSON would be one string, and escaping gets messy. By parsing first, promptfoo can handle variable substitution cleanly within each field.

[📂 src/evaluatorHelpers.ts](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/evaluatorHelpers.ts#L474-L478)

## Prompt Functions: Maximum Flexibility

When templates aren't enough, prompt functions give you full programmatic control.

### When to Use Prompt Functions

1. **Complex conditionals**: More than a few `{% if %}` blocks
2. **Dynamic structure**: The number of messages varies per test
3. **External dependencies**: Need to call APIs or databases
4. **Type-specific handling**: Different logic for images vs. text
5. **Provider-specific prompts**: Different formats for different providers

### JavaScript Prompt Functions

```javascript
// prompts/smart-prompt.js
module.exports = async function({ vars, provider }) {
  const messages = [];
  
  // System message varies by provider
  if (provider.id.includes('claude')) {
    messages.push({
      role: 'system',
      content: 'You are Claude, an AI assistant by Anthropic.'
    });
  } else {
    messages.push({
      role: 'system',
      content: 'You are a helpful AI assistant.'
    });
  }
  
  // Handle different input types
  if (vars.image) {
    messages.push({
      role: 'user',
      content: [
        { type: 'image_url', image_url: { url: vars.image } },
        { type: 'text', text: vars.question }
      ]
    });
  } else {
    messages.push({
      role: 'user',
      content: vars.question
    });
  }
  
  return JSON.stringify(messages);
};
```

### Python Prompt Functions

```python
# prompts/smart-prompt.py
import json

def generate_prompt(context):
    vars = context['vars']
    provider = context['provider']
    
    messages = []
    
    # Build system message
    if 'claude' in provider['id']:
        messages.append({
            'role': 'system',
            'content': 'You are Claude.'
        })
    else:
        messages.append({
            'role': 'system',
            'content': 'You are a helpful assistant.'
        })
    
    # Add user message
    messages.append({
        'role': 'user',
        'content': vars['question']
    })
    
    return json.dumps(messages)
```

### Returning Config from Prompt Functions

Prompt functions can also return configuration:

```javascript
module.exports = async function({ vars }) {
  // Longer inputs need more output tokens
  const maxTokens = vars.text.length > 1000 ? 2000 : 500;
  
  return {
    prompt: `Summarize: ${vars.text}`,
    config: {
      max_tokens: maxTokens,
      temperature: 0.3
    }
  };
};
```

The returned `config` merges with the provider's config, allowing prompt-specific settings.

[📂 src/types/prompts.ts](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/types/prompts.ts#L34-L37)

## Provider-Specific Prompts

Sometimes different providers need different prompts:

```yaml
providers:
  - id: openai:gpt-4
    prompts:
      - "openai-prompt"
  - id: anthropic:claude-3-opus
    prompts:
      - "claude-prompt"

prompts:
  - label: openai-prompt
    raw: |
      You are a helpful assistant.
      User: {{question}}
  
  - label: claude-prompt
    raw: |
      Human: {{question}}
      Assistant:
```

**How it works**: The `prompts` field on a provider specifies which prompt labels that provider should use. Promptfoo filters the prompt-provider combinations during expansion.

**Use cases**:
- Format differences (OpenAI chat format vs. Anthropic Human/Assistant)
- Provider-specific instructions ("You are Claude...")
- Feature support (some providers support system messages, others don't)

## Special Variables

Promptfoo provides built-in variables:

```
{{_conversation}}  - Multi-turn conversation history
{{env.VAR_NAME}}   - Environment variables
```

**`_conversation`**: In multi-turn evaluations, this contains the conversation so far. It's automatically populated when you use conversation features.

**Disabling**: If you use `_conversation` literally in a prompt (not as the variable), set `disableConversationVar: true`.

## Security Considerations

### The skipRenderVars Mechanism

For red team testing, you might have attack payloads in variables:

```yaml
tests:
  - vars:
      inject: "{{constructor.constructor('return this')()}}"  # SSTI attempt
```

If Nunjucks renders this, it might execute or break. The `skipRenderVars` option prevents rendering specific variables:

```typescript
renderPrompt(prompt, vars, filters, provider, ['inject']);
// 'inject' value is passed through unchanged
```

**Why this matters**: Red team tests need to send malicious payloads to the target LLM to test its defenses. If promptfoo's template engine processes these payloads, you're testing promptfoo's security, not your LLM's.

[📂 src/evaluatorHelpers.ts](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/evaluatorHelpers.ts#L214-L217)

### Environment Variable Access Control

By default in self-hosted deployments, templates cannot access `process.env`:

```typescript
const processEnvVarsDisabled = getEnvBool(
  'PROMPTFOO_DISABLE_TEMPLATE_ENV_VARS',
  getEnvBool('PROMPTFOO_SELF_HOSTED', false)
);
```

**Why**:
- Templates might be user-provided in hosted scenarios
- Prevents accidental exposure of secrets
- Defense in depth

Only variables explicitly defined in the config's `env` section are accessible.

## Prompt Labeling and Identification

Every prompt gets a label for identification:

```yaml
prompts:
  # Explicit label
  - label: translator-v2
    raw: "Translate {{text}} to {{language}}"
  
  # Auto-generated label from content
  - "Summarize: {{text}}"  # Label: "Summarize: {{text}}"
  
  # Auto-generated label from file path
  - file://prompts/analyzer.txt  # Label: "prompts/analyzer.txt"
```

**Why labels matter**:

1. **Display**: The UI shows labels, not raw content
2. **Provider matching**: `providers[].prompts` references labels
3. **Deduplication**: Same label = same prompt (for caching)
4. **Debugging**: Logs reference labels

## Design Decisions and Trade-offs

### Why Support So Many Formats?

**Alternative**: Just support YAML/JSON config with inline prompts

**Why diverse formats**:
1. **Team preferences**: Some teams love Markdown, others prefer code
2. **Tooling integration**: Prompts in .txt files work with any editor
3. **Gradual adoption**: Start simple (inline), grow complex (functions)
4. **External systems**: Many teams already have prompts in Langfuse/Portkey

**Trade-off**: More complexity in the codebase, more documentation needed.

### Why Nunjucks Instead of Native Template Literals?

**Alternative**: JavaScript template literals (`\`Hello ${name}\``)

**Why Nunjucks**:
1. **Config-file friendly**: Template literals require JavaScript context
2. **Security**: Template literals execute code; Nunjucks is sandboxed
3. **Features**: Filters, loops, conditionals without escaping to JS
4. **Non-JS sources**: YAML, JSON, and text files can use templates

### Why Parse JSON Prompts Specially?

**Alternative**: Treat all prompts as strings, let users handle JSON escaping

**Why special handling**:
1. **Common case**: Chat APIs use JSON message arrays
2. **Usability**: JSON escaping is error-prone
3. **Readability**: Formatted JSON in YAML is cleaner

**Trade-off**: Magic behavior can surprise users. Documented clearly.

## Common Patterns

### Pattern: Prompt Variants

Test different prompt styles:

```yaml
prompts:
  - label: concise
    raw: "Answer briefly: {{question}}"
  
  - label: detailed
    raw: |
      Please provide a comprehensive answer to the following question.
      Include relevant context and examples where appropriate.
      
      Question: {{question}}
  
  - label: structured
    raw: |
      Question: {{question}}
      
      Please structure your answer with:
      1. A brief summary
      2. Key points
      3. A conclusion
```

### Pattern: System + User Separation

```yaml
prompts:
  - |
    [
      {"role": "system", "content": "{{system_prompt}}"},
      {"role": "user", "content": "{{user_input}}"}
    ]

tests:
  - vars:
      system_prompt: "You are a helpful coding assistant."
      user_input: "How do I reverse a string in Python?"
```

### Pattern: Few-Shot Examples

```yaml
prompts:
  - |
    Classify the sentiment of the following text.
    
    Examples:
    {% for example in examples %}
    Text: "{{example.text}}"
    Sentiment: {{example.sentiment}}
    
    {% endfor %}
    Text: "{{text}}"
    Sentiment:

tests:
  - vars:
      examples:
        - text: "I love this product!"
          sentiment: positive
        - text: "This is terrible."
          sentiment: negative
      text: "It's okay, nothing special."
```

### Pattern: Multi-Modal Prompts

```yaml
prompts:
  - file://prompts/vision.js

# prompts/vision.js
module.exports = async function({ vars }) {
  return JSON.stringify([
    {
      role: "user",
      content: [
        { type: "image_url", image_url: { url: vars.image } },
        { type: "text", text: vars.question }
      ]
    }
  ]);
};
```

## File Reference

| File | Purpose |
|------|---------|
| [src/prompts/index.ts](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/prompts/index.ts) | Main entry point for prompt loading and processing |
| [src/prompts/utils.ts](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/prompts/utils.ts) | Utility functions (file path detection, normalization) |
| [src/evaluatorHelpers.ts](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/evaluatorHelpers.ts) | `renderPrompt` function and variable resolution |
| [src/util/templates.ts](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/util/templates.ts) | Nunjucks engine configuration |
| [src/types/prompts.ts](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/types/prompts.ts) | TypeScript types for prompts |
| [src/prompts/processors/](https://github.com/promptfoo/promptfoo/blob/cccb985605edf448546fa996467a4ba6e169129d/src/prompts/processors/) | File-type-specific processors (json.ts, javascript.ts, etc.) |

## Summary

The prompt template system in promptfoo provides a spectrum of capabilities:

1. **Simple strings**: Quick inline prompts for experiments
2. **File-based templates**: Separation of prompts from config
3. **Nunjucks templating**: Variables, filters, conditionals, loops
4. **JSON handling**: First-class support for chat message arrays
5. **Prompt functions**: Full programmatic control when needed
6. **External services**: Integration with prompt management systems

The design philosophy prioritizes:

- **Progressive complexity**: Start simple, add power as needed
- **Format flexibility**: Support how teams actually work
- **Security**: Sandboxed templates, controlled env access
- **Performance**: Load once, render per-test efficiently
