# Design notes

One entry per real design decision: the decision, the options considered, and why.

## 2026-11-25: LLM provider is Groq, default model `openai/gpt-oss-120b`

**Decision.** The agent calls Groq's Chat Completions API through the official Groq Python SDK.
The default model is `openai/gpt-oss-120b`, set with `LLM_MODEL`. The constitution names no
provider; AGENTS.md section 6 holds the provider-specific rules.

**Options considered.**

| Option | For | Against |
|---|---|---|
| Claude API (`claude-opus-5-5`) | Strong tool use; the SDK has refusal and caching features | Paid per token from the first call; no free tier for a two-person portfolio project |
| OpenAI API, paid with ChatGPT Pro API credits | Credits come with an existing plan | Needs a Pro plan ($100+/month); ChatGPT Plus and the Codex sign-in give no API credits |
| Groq, `openai/gpt-oss-120b` | Free plan with no card; Developer plan at $0.15 / $0.60 per 1M tokens; tool use, strict structured outputs, `reasoning_effort`, automatic prompt caching | Free plan limits (8,000 tokens/min, 200,000 tokens/day per model); no parallel tool calls |
| Groq, `qwen/qwen3.8-27b` | Parallel tool calls | Preview model; $0.80 / $4 per 1M tokens; no `reasoning_effort` |
| Groq, `openai/gpt-oss-20b` | Cheaper and faster | Weaker reasoning, so less trustworthy evaluation numbers |

**Why.** Cost: development and tests use a scripted fake LLM, and the few live runs fit in Groq's
free plan. `openai/gpt-oss-120b` is a production model on Groq, so it is less likely to disappear
than a preview model. Sequential tool calls are enough for our loop and are simpler to explain.

**Consequences.**
- The live evaluation (feature 005) and the public demo (feature 007) must respect the rate
  limits. A full evaluation run may not fit in one day of the free plan; the Developer plan
  raises the limits.
- The constitution's principle VIII is now provider-neutral, so a later provider change
  only edits AGENTS.md section 6, not the constitution.

Prices and limits were read on 2026-11-25 from Groq's models, rate limits, and prompt caching
documentation.
