# Common Patterns (JavaScript / TypeScript)

Shared patterns for JS/TS integration types (Patterns A–D). Python patterns are self-contained in their own reference files.

## Instrumentation Coverage

Every LLM call in the user's code must produce a span. For each call site, use the highest-fidelity approach available:

1. **Auto-telemetry** — `experimental_telemetry` (Vercel AI SDK), auto-instrumentation (LangChain). Zero manual work.
2. **Provider wrappers** — `trackOpenAI`, `trackAnthropic`, `trackVercelAI` / `trackVercelAIStream`. Use when auto-telemetry is unavailable.
3. **Core SDK** — `trackLlm`, `trackLlmStream`, `recordLlm`. Fallback for any uncovered call.

After applying the base integration, scan for any LLM call that would not produce a span and wrap it.

## `init()` and `shutdown()`

### Scripts and CLIs

Always call `shutdown()` before the process exits to flush pending traces:

```typescript
import { init } from "@spanora-ai/sdk";

const { shutdown } = init({
  apiKey: process.env.SPANORA_API_KEY,
});

try {
  // ... agent logic ...
} finally {
  await shutdown();
}
```

### Long-running servers (Express, Next.js, etc.)

`shutdown()` is not needed — traces are flushed automatically in batches:

```typescript
import { init } from "@spanora-ai/sdk";

init({ apiKey: process.env.SPANORA_API_KEY });

// No shutdown() needed for servers
```

## Tool Tracking

### `trackToolHandler()` — wrap a tool handler

Returns a function with the same signature. Works with all patterns:

```typescript
import { trackToolHandler } from "@spanora-ai/sdk";

const getWeather = trackToolHandler(
  "getWeather",
  async (input: { city: string }) => {
    const data = await weatherApi.lookup(input.city);
    return { temperature: data.temp, condition: data.condition };
  },
);

// Use it like a normal function — spans are created automatically
const result = await getWeather({ city: "Paris" });
```

### `runTool()` — error-safe tool execution

Catches errors instead of throwing. Returns `{ output, error }`:

```typescript
import { runTool } from "@spanora-ai/sdk";

const result = await runTool("getWeather", { city: "Paris" }, async (input) =>
  weatherApi.lookup(input.city),
);

if (result.error) {
  console.error("Tool failed:", result.error);
} else {
  console.log("Weather:", result.output);
}
```

## Multi-Agent — Shared Context

When multiple agents share the same session, pass `userId`, `orgId`, and `agentSessionId` to each `track()` call to link traces together in the dashboard:

```typescript
const sharedContext = {
  userId: "user-42",
  orgId: "org-acme",
  agentSessionId: "session-abc-123",
};

// Agent 1
await track({ agent: "planner-agent", ...sharedContext }, async () => {
  /* planning logic */
});

// Agent 2
await track({ agent: "executor-agent", ...sharedContext }, async () => {
  /* execution logic */
});
```

## Agent Naming Guidance

Use descriptive, purpose-based names. The `agent` appears in the Spanora dashboard.

**Good names:**

- `"support-agent"` — handles customer support
- `"planner-agent"` — creates execution plans
- `"code-reviewer"` — reviews code changes
- `"data-extractor"` — extracts structured data
- `"summarizer"` — summarizes documents

**Bad names:**

- `"agent-1"`, `"my-agent"`, `"test"`, `"default"`

## API Key Setup

### Spanora API Key

Required for sending traces. Store in `.env`:

```
SPANORA_API_KEY=ak_your_api_key
```

If `.env` is not in `.gitignore`, remind the user to add it.

### LLM Provider API Keys

Do not ask for these — just inform the user if they appear to be missing:

- **OpenAI SDK** → needs `OPENAI_API_KEY` in `.env`
- **Anthropic SDK** → needs `ANTHROPIC_API_KEY` in `.env`
- **Vercel AI SDK** → depends on the provider used (e.g. `OPENAI_API_KEY` for OpenAI provider)

Example `.env`:

```
SPANORA_API_KEY=ak_...
OPENAI_API_KEY=sk-...
ANTHROPIC_API_KEY=sk-ant-...
```

## Migration Checklist

When instrumenting existing code:

1. Add `init()` at application startup (entry point)
2. Wrap existing LLM calls with the appropriate tracking function
3. Wrap tool handlers with `trackToolHandler()`
4. Add `track()` around execution boundaries (per-request, per-job, per-agent-run)
5. Name agents based on what they do
6. Add `shutdown()` at application teardown (scripts/CLIs only)
7. Pass shared context (`userId`, `orgId`, `agentSessionId`) for multi-agent setups
