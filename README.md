# pi-cli-tool-summarizer

**Cut your pi cli (ai agent) token usage by 40-60%.** A pi extension that summarizes verbose tool outputs (test results, build logs, file dumps) using a tiny local LLM before they reach your expensive API model.

## Why?

AI coding agents like pi, Claude Code, and OpenCode all share the same fundamental loop:

```
LLM → calls bash "npm test" → 5000 lines of output → fed back to LLM → LLM reads all 5000 lines
```

The expensive model doesn't need to read 5000 lines of test output. It just needs to know *what failed*. Sending raw output verbatim is pure token waste. summarizing is an easy task and small local models are greate at it. lets use them!

This extension intercepts tool results and runs them through a **local** 1B parameter model (llama3.2:1b) that compresses the noise into 1-2 sentences. The expensive model only sees the summary.

## How It Works

```
pi runs bash "npm test"
  ↓
Tool returns 5000 chars of output
  ↓
Extension fires (tool_result event)
  ↓
Calls llama3.2:1b running locally via Ollama
  ↓
Returns: "[summarized 5000 → 180 chars]
          3 tests failed: AuthService.login assertion error,
          DatabaseService.connect connection refused"
  ↓
Expensive LLM sees only the summary
  ↓
Token saved: ~4500 per large tool result
```

### What Gets Summarized

Any tool output (bash, read, etc.) that is:
- Text content (not binary)
- Longer than 500 characters
- Too large to justify sending raw to the expensive model

### What Stays Unchanged

- Small outputs (< 500 chars) pass through unmodified
- If Ollama isn't running, the extension fails silently and original content is preserved
- Binary outputs are skipped

## Requirements

- [Ollama](https://ollama.ai) running locally
- A small model pulled: `ollama pull llama3.2:1b`

## Installation

### Option 1: Auto-discover (easiest)

```bash
# Create the extensions directory if it doesn't exist
mkdir -p ~/.pi/agent/extensions

# Copy the extension file
cp src/summarize-tool-output.ts ~/.pi/agent/extensions/

# Restart pi — extension loads automatically
pi
```

### Option 2: Via settings

```bash
# Install from a path
pi install /path/to/summarize-tool-output.ts

# Or add to ~/.pi/settings.json:
# {
#   "extensions": ["/path/to/summarize-tool-output.ts"]
# }
```

### Option 3: Project-local

Place it in your project's `.pi/extensions/` directory for team-wide use:

```bash
mkdir -p .pi/extensions
cp src/summarize-tool-output.ts .pi/extensions/
```

## Configuration

Edit the constants at the top of `summarize-tool-output.ts`:

```typescript
const MODEL = "llama3.2:1b";        // Any Ollama model you have
const MIN_CHARS_TO_SUMMARIZE = 500;  // Skip small outputs
const MAX_INPUT_CHARS = 3000;        // Truncate very long inputs
```

To use a different model (e.g., `qwen2.5:1.5b` or `llama3.2:3b`):

```bash
ollama pull llama3.2:3b
# Then set MODEL = "llama3.2:3b" in the extension file
```

## Benchmarks

| Model | Size | Avg Time | Quality |
|-------|------|----------|---------|
| `qwen2.5:0.5b` | 397MB | 2.2s | ❌ Hallucinates facts |
| **`llama3.2:1b`** | ~700MB | **4.5s** | **✅ Reliable, recommended** |
| `llama3.2:3b` | ~2GB | ~8s | ✅ Better but slower |

The 0.5B model was tested and rejected — it hallucinated that *failing* tests were *passing*, which would mislead the expensive model. The 1B model is the minimum viable size for reliable summarization.

## How We Tested

We compared qwen2.5:0.5b and llama3.2:1b on 4 sample types:

1. **Test failures** — `15 tests, 3 failed, assertion errors + timeouts`
2. **Build errors** — `3 TypeScript compiler errors across 2 files`
3. **Git log** — `3 commits with messages`
4. **npm install** — `peer dependency conflict, 1423 packages added`

Results: llama3.2:1b correctly identified all failures and errors. qwen2.5:0.5b claimed failing tests were "successful."

## File Size

The extension is **~50 lines of actual logic**. The entire concept is:

1. Hook into pi's `tool_result` event
2. If the output is long, call a local model to summarize it
3. Replace the content with the summary
4. If anything fails, do nothing — original content stays

## License

MIT
