# Coding Agents Comparison — April 2026

**A technical comparison for developers who actually need to choose.**

Last updated: 2026-04-02. Based on source code analysis, not marketing.

---

## TL;DR

- **Claude Code** is the best coding agent. It's also locked to one provider with aggressive rate limits.
- **OpenCode** is the best open-source alternative. 23+ providers, switch mid-session, MIT license.
- **OpenClaude** is Claude Code with a shim for OpenAI-compatible APIs. Fast hack, fragile long-term.
- **None of them** have automatic failover when you hit rate limits.
- **OpenClaw and Hermes Agent** are not coding agents — they're orchestrators that *call* coding agents.

---

## Table of Contents

1. [The Three Terminal Agents](#1-the-three-terminal-agents)
2. [Architecture Deep Dive](#2-architecture-deep-dive)
3. [Tool System Comparison](#3-tool-system-comparison)
4. [The Critical Problem: tool_use Translation](#4-the-critical-problem-tool_use-translation)
5. [Multi-Provider & Seamless Relay](#5-multi-provider--seamless-relay)
6. [Context Management](#6-context-management)
7. [Agent & Orchestration](#7-agent--orchestration)
8. [Extensibility](#8-extensibility)
9. [Other Terminal Agents](#9-other-terminal-agents)
10. [IDE Agents](#10-ide-agents)
11. [The Orchestrator Layer](#11-the-orchestrator-layer-openclaw-hermes-pinokio)
12. [Model Ranking for Agent Work](#12-model-ranking-for-agent-work-april-2026)
13. [Models You Should Know About](#13-models-you-should-know-about-april-2026)
14. [Practical Recommendations](#14-practical-recommendations)
15. [Benchmark Sources & References](#15-benchmark-sources--references)

---

## 1. The Three Terminal Agents

| | Claude Code | OpenCode | OpenClaude |
|---|---|---|---|
| **Repo** | Proprietary (Anthropic) | [anomalyco/opencode](https://github.com/anomalyco/opencode) | [Gitlawb/openclaude](https://github.com/Gitlawb/openclaude) |
| **Stars** | N/A (84K+ on leak mirrors) | 135K | 4.2K |
| **License** | Proprietary | MIT | "Educational" (based on leaked code) |
| **Age** | 2+ years | ~1 year | 2 days (April 1, 2026) |
| **Language** | TypeScript (Node.js) | TypeScript (Bun) | TypeScript (Node.js) |
| **Origin** | Anthropic | SST team (Dax Raad) | Fork of leaked Claude Code |
| **Version** | Latest | v1.3.13 | v0.1.6 |

---

## 2. Architecture Deep Dive

### Claude Code: Deep Anthropic Coupling

The entire codebase is built directly on `@anthropic-ai/sdk`. There is **zero abstraction layer** between the application and the Anthropic API.

- **~50+ files** import Anthropic SDK types (`BetaMessage`, `BetaToolUseBlock`, `BetaRawMessageStreamEvent`, etc.)
- **`claude.ts` (~3,400 lines)**: The query engine. Handles streaming, thinking, caching, retries. 100% Anthropic-specific.
- **`query.ts` (~1,730 lines)**: The agentic loop. Tool execution, follow-up, compaction triggers.
- **4 providers**: All Claude — firstParty, Bedrock, Vertex, Foundry. Each cast to `Anthropic` type.

Blast radius of changing the API: **everything**. No adapter pattern exists.

### OpenCode: AI SDK Abstraction

Built on **Vercel AI SDK v6**, which abstracts provider differences at the protocol level.

- **Provider-agnostic messages**: Stored as `ModelMessage[]`, converted per-provider at send time
- **23 bundled providers**: Each is an AI SDK adapter (`@ai-sdk/anthropic`, `@ai-sdk/openai`, `@ai-sdk/google`, etc.)
- **`ProviderTransform.message()`**: Normalizes message history per-provider (tool_call ID formats, reasoning block placement, empty content filtering)
- **`streamText()`**: Single function call that works identically across all providers
- **Client/server architecture**: HTTP + SSE. TUI, web app, desktop app, and IDE all connect to the same backend.

### OpenClaude: Duck-Type Shim

A single conditional in `client.ts` redirects to a fake Anthropic SDK:

```typescript
if (isEnvTruthy(process.env.CLAUDE_CODE_USE_OPENAI)) {
    const { createOpenAIShimClient } = await import('./openaiShim.js')
    return createOpenAIShimClient({...}) as unknown as Anthropic
}
```

The returned object mimics `{ beta: { messages: { create() } } }`. The entire rest of Claude Code — including `claude.ts` (3,400 lines, **never modified**) — thinks it's talking to real Anthropic.

**4 new files** (~2,400 lines total):
- `openaiShim.ts` (865 lines): OpenAI Chat Completions translation
- `codexShim.ts` (886 lines): OpenAI Responses API translation
- `providerConfig.ts` (313 lines): Provider resolution
- `openaiContextWindows.ts` (138 lines): Hardcoded context limits

**6 existing files** surgically modified (~200 lines of changes).

---

## 3. Tool System Comparison

| Tool | Claude Code | OpenCode | OpenClaude |
|---|---|---|---|
| bash | Yes | Yes (+ bun-pty) | Yes (via CC) |
| read/write/edit | Yes | Yes | Yes (via CC) |
| glob/grep | Yes | Yes | Yes (via CC) |
| webfetch | Yes | Yes | Yes (via CC) |
| **websearch** | No (via MCP) | **Yes (native)** | No |
| **codesearch** | No | **Yes (semantic)** | No |
| **LSP** | No | **Yes (experimental)** | No |
| **multiedit** | No | **Yes** | No |
| **batch** | No | **Yes (experimental)** | No |
| **apply_patch** | No | **Yes (auto for GPT-5+)** | Yes (via shim) |
| agent/task | Yes (subagents) | Yes (task tool) | Yes (via CC) |
| **ToolSearch (deferred)** | Yes | No | **Broken (filtered by shim)** |
| **Custom tools** | No | **Yes (.opencode/tool/*.ts)** | No |
| MCP tools | Yes | Yes | Yes |

OpenCode automatically swaps `edit`/`write` for `apply_patch` when using GPT-5+ models (which perform better with patch format).

---

## 4. The Critical Problem: tool_use Translation

This is the technical crux of the "seamless relay" question.

### Claude Code: Native Anthropic Format

```
Assistant → tool_use { id, name, input }  (streamed via input_json_delta)
User → tool_result { tool_use_id, content, is_error }
```

Streaming events: `message_start → content_block_start → content_block_delta → content_block_stop → message_delta → message_stop`

### OpenClaude: Manual Translation (865 lines)

The shim translates bidirectionally:

| Anthropic | OpenAI | Status |
|---|---|---|
| `tool_use` content blocks | `tool_calls[].function` | **Works** |
| `tool_result` user messages | `role: "tool"` messages | **Works** |
| `tool_choice` (auto/any/tool) | auto/required/{function} | **Works** |
| Streaming SSE events | Re-translated to Anthropic format | **Works** |
| `thinking` blocks | **Dropped** (silently ignored) | **Broken** |
| `cache_control` (prompt caching) | **Dropped** (hardcoded to 0) | **Broken** |
| Beta headers | Sent but meaningless to OpenAI | **Wasted** |
| Fast mode | **Dropped** | **Broken** |
| Images in `tool_result` | **Lost** (flattened to text) | **Broken** |
| `ToolSearchTool` | **Filtered out** (deferred tools disabled) | **Broken** |
| Token costs | Shows Claude pricing, not actual | **Misleading** |

**What still works through the shim**: All 15+ base tools, streaming, multi-step tool chains, vision in user messages, sub-agents, MCP, skills, hooks, CLAUDE.md, memory, permissions, sessions.

### OpenCode: AI SDK Does Everything

OpenCode doesn't translate. It defines tools as provider-agnostic `ai.Tool` objects with JSON Schema. Each AI SDK provider adapter handles native conversion:

- `@ai-sdk/anthropic` → Anthropic `tool_use` blocks
- `@ai-sdk/openai` → OpenAI `function` calls / Responses API
- `@ai-sdk/google` → Gemini `functionDeclaration`
- etc.

Additionally, `ProviderTransform.message()` normalizes history per-provider:
- **Anthropic**: Filters empty content, applies cache_control breakpoints
- **Mistral**: Reformats tool_call_id (9-char limit), injects dummy assistant messages
- **DeepSeek/reasoning models**: Moves reasoning parts to `providerOptions`

---

## 5. Multi-Provider & Seamless Relay

| Scenario | Claude Code | OpenCode | OpenClaude |
|---|---|---|---|
| Switch provider mid-session | **Impossible** | **Yes (manual keybind)** | **Impossible** (env var, restart) |
| Messages from multiple providers in one session | No | **Yes** (each msg stores its provider) | No |
| Provider-specific system prompts | 1 prompt | **6 different prompts** per model family | 1 (Claude's) |
| Provider-specific tool swaps | No | **Yes** (edit/write ↔ apply_patch) | No |
| Automatic fallback on 429 | **No** | **No** | **No** |
| Retry on rate limit | Same model, exponential backoff | Same model, exponential backoff | Same model |

### The honest truth about "seamless relay"

**No tool today provides automatic failover when you hit rate limits.** Not Claude Code, not OpenCode, not OpenClaude, not any of them.

OpenCode comes closest: you can manually switch models mid-session with a keybind, and the conversation history is preserved and re-serialized for the new provider. But it's manual.

### OpenCode's Provider-Specific System Prompts

This matters more than people think. When you switch providers in OpenCode, the system prompt changes completely:

| Model Family | Prompt Character | Key Differences |
|---|---|---|
| **Claude** (anthropic.txt) | "Best coding agent on the planet" | TodoWrite-focused, verbose, professional |
| **GPT-5+** (gpt.txt) | "Deeply pragmatic engineer" | `apply_patch`, `multi_tool_use.parallel`, minimal output |
| **Gemini** (gemini.txt) | Convention-focused | "Core Mandates" style, very structured |
| **Kimi** (kimi.txt) | Separate prompt | Kimi-specific adjustments |
| **Default** (default.txt) | Minimal | "Fewer than 4 lines" constraint |

This means the agent doesn't just change its brain when you switch — it changes its entire personality and tool strategy.

---

## 6. Context Management

| | Claude Code | OpenCode | OpenClaude |
|---|---|---|---|
| **Compaction strategies** | 5 (micro, context collapse, reactive, full, PTL) | 2 (pruning 40K + summary) | Same as CC |
| **Prompt caching** | Yes (cache_control, breakpoints, global/ephemeral scope) | Anthropic only (via AI SDK provider options) | **Dropped** |
| **Extended thinking** | Yes (adaptive, budget tokens, signature) | Partial (reasoning via providerOptions) | **Dropped** |
| **Token tracking** | API-reported + hardcoded pricing | API-reported + **models.dev dynamic pricing** | API-reported, **wrong pricing** |
| **Max context** | 200K (1M with Opus) | Depends on model (models.dev database) | Hardcoded lookup ~30 models |

### What "dropped prompt caching" actually costs you

Claude Code's prompt caching (`cache_control: { type: 'ephemeral' }`) means repeated tool schemas and system prompts are cached server-side. This reduces costs by 50-90% on long sessions. OpenClaude hardcodes cache tokens to 0, meaning:
- You pay full price for every message
- Long sessions become proportionally more expensive vs real Claude Code

---

## 7. Agent & Orchestration

| | Claude Code | OpenCode | OpenClaude |
|---|---|---|---|
| Built-in agents | Agent, Explore, Plan (implicit) | build, plan, general, explore, compaction | Same as CC |
| Custom agents | `.claude/agents/*.md` | `.opencode/agent/*.md` + JSON config | Same as CC |
| **Model per agent** | No (inherited) | **Yes** (each agent can use a different provider/model) | No |
| **Agent Teams** | **Yes** | No | Same as CC |
| Background agents | **Yes (experimental)** | No (but client/server architecture enables it) | Same as CC |
| ACP (Agent Client Protocol) | No | **Yes** (server for Zed, etc.) | No |

OpenCode's "model per agent" is significant: you can run your main agent on Claude, your explore sub-agent on Ollama, and your compaction agent on a cheap model. No other tool offers this.

---

## 8. Extensibility

| | Claude Code | OpenCode | OpenClaude |
|---|---|---|---|
| **Hooks** | 7 events | **15+ events** | Same as CC |
| **Plugin system** | Basic (marketplace) | **Rich** (npm plugins, auth providers, TUI extensions) | Same as CC |
| **Custom tools at runtime** | No | **Yes** (.opencode/tool/*.ts) | No |
| **Custom slash commands** | Skills (.claude/skills/) | Commands (.opencode/command/) | Same as CC |
| **Themes** | No | **25+ TUI themes** | No |
| **Session export** | No | **Yes** (export/import/share/fork) | No |
| **Desktop app** | Yes (Anthropic) | **Yes (Tauri)** | No |
| **Web app** | Yes (claude.ai/code) | **Yes (SolidJS)** | No |

OpenCode's hook system is substantially more powerful:

| Hook | Claude Code | OpenCode |
|---|---|---|
| PreToolUse | Yes | Yes (`tool.execute.before`) |
| PostToolUse | Yes | Yes (`tool.execute.after`) |
| **Modify tool definitions** | No | Yes (`tool.definition`) |
| **Modify LLM params** | No | Yes (`chat.params`) |
| **Add HTTP headers** | No | Yes (`chat.headers`) |
| **Transform messages** | No | Yes (`experimental.chat.messages.transform`) |
| **Transform system prompt** | No | Yes (`experimental.chat.system.transform`) |
| **Override permissions** | No | Yes (`permission.ask`) |
| **Inject shell env** | No | Yes (`shell.env`) |
| **Custom compaction** | No | Yes (`experimental.session.compacting`) |

---

## 9. Other Terminal Agents

| | Aider | Codex CLI | Goose |
|---|---|---|---|
| **Repo** | [Aider-AI/aider](https://github.com/Aider-AI/aider) | [openai/codex](https://github.com/openai/codex) | [block/goose](https://github.com/block/goose) |
| **Stars** | 42K | 67K | 29K |
| **License** | Apache 2.0 | Apache 2.0 | Apache 2.0 |
| **Providers** | 100+ LLMs | OpenAI only | Any |
| **Tools** | File edit, git | File, bash, web, image | File, bash, browser, API |
| **MCP** | No (native) | Yes | **Yes (3,000+ servers)** |
| **Sub-agents** | No | Yes | Recipes |
| **Context mgmt** | Repo map (manual) | 256K-1M | Basic |
| **Unique strength** | Best git integration | Kernel-level sandbox | Free, deepest MCP ecosystem |
| **Weakness** | No session resume, no bash | OpenAI lock-in | Less polished for coding |

---

## 10. IDE Agents

| | Cursor | Cline | Roo Code | Zed AI |
|---|---|---|---|---|
| **Type** | VS Code fork | VS Code extension | VS Code extension | Native IDE (Rust) |
| **Stars** | N/A (closed) | 59K | 23K | 76K |
| **Providers** | Multi (limited) | 15+ | Any | 8+ |
| **MCP** | Yes | **Deepest** (100+ marketplace) | Yes | Partial |
| **Sub-agents** | Cloud + Background | Yes (v3.58) | Custom Modes | spawn_agent |
| **Cost** | $20-40/mo | Free + API | Free + API | $0-20/mo |
| **Unique strength** | Most polished IDE | Browser automation | Multi-persona | Fastest editor |

---

## 11. The Orchestrator Layer (OpenClaw, Hermes, Pinokio)

These are **not competitors** to Claude Code / OpenCode. They're the layer above.

```
Your phone (Discord / WhatsApp / Telegram)
        |
  OpenClaw or Hermes Agent    <-- autonomous brain, 24/7
        |
    Pinokio 7                  <-- app launcher, SKILL.md layer
        |
  +--------+--------+--------+
  | WanGP  | ComfyUI | Ollama | ...any local AI app
  +--------+--------+--------+
        |
  Claude Code / Codex / OpenCode  <-- coding executors, on-demand
```

### OpenClaw (345K stars, MIT)

- **What it is**: Personal AI assistant gateway. Bridges messaging platforms to LLM providers.
- **What it isn't**: A coding agent. It delegates coding to Claude Code / Codex / OpenCode via ACPX.
- **Messaging**: WhatsApp, Telegram, Slack, Discord, Signal, iMessage, IRC, Teams, Matrix, LINE, WeChat
- **Ecosystem**: ClawHub (13,729+ community skills)
- **Security warning**: CVE-2026-25253 (RCE, CVSS 8.8). 22-26% of ClawHub skill submissions contain malicious code.

### Hermes Agent (Nous Research, 21.8K stars, MIT)

- **What it is**: Self-improving autonomous agent with persistent cross-session memory.
- **Key differentiator**: Closed learning loop — saves successful approaches as reusable skills, improves them over time.
- **Self-evolution**: GEPA (Genetic Evolution of Prompt Architectures), ICLR 2026 Oral paper.
- **40+ tools**: Terminal, filesystem, browser, code execution, vision, image gen, TTS.
- **Messaging**: Telegram, Discord, Slack, WhatsApp, Signal.
- **Simpler than OpenClaw**: Python-based, linear architecture.

### Pinokio 7 (7.2K stars, MIT)

- **What it is**: 1-click launcher and orchestrator for local AI apps.
- **Key feature (v7)**: Every installed app auto-exposes a SKILL.md, making it discoverable by any agent.
- **Not a competitor**: It's infrastructure. It manages app lifecycle so agents don't have to.

---

## 12. Model Ranking for Agent Work (April 2026)

When your primary model hits limits, which model should you fall back to?

Data sourced from: SWE-bench Verified/Pro, SWE-rebench, BFCL V4 (Berkeley Function Calling), Chatbot Arena (LMArena), Artificial Analysis Intelligence Index, OpenRouter usage rankings, and MCP Atlas benchmark. All numbers verified April 2, 2026.

### Frontier Models (Closed-Source)

| Model | SWE-bench V. | SWE-bench Pro | BFCL V4 | Arena ELO | Context | Price (in/out per Mtok) |
|---|---|---|---|---|---|---|
| **Claude Opus 4.6** | 80.8% | ~46% | 70.4% | 1504 | 1M | $5/$25 |
| **Gemini 3.1 Pro** | 80.6% | — | — | 1500 | **2M** | **$2/$12** |
| **GPT-5.4** | ~80% | **57.7%** | 59.2% | ~1490 | 1M | $2.50/$15 |
| **Grok 4.1** | — | — | — | **1483** | 2M | **$0.20/$0.50** (Fast) |
| Claude Sonnet 4.6 | 79.6% | — | 70.3% | — | 1M | $3/$15 |
| GPT-5.4 Mini | — | 54.4% | — | — | 400K | $0.75/$4.50 |

**Key findings:**
- Opus 4.6 leads SWE-bench Verified but GPT-5.4 leads the harder, decontaminated SWE-bench Pro
- Claude models dominate function calling (BFCL V4: 70%+). GPT-5 is weaker at tool use (59%)
- Gemini 3.1 Pro leads **MCP Atlas** (tool coordination benchmark) at 69.2% and costs 2.5x less than Opus
- Grok 4.1 Fast at $0.20/$0.50 is absurdly cheap for its Arena ELO

### Open-Source Models (S-Tier for Coding)

| Model | Params (total/active) | SWE-bench V. | License | Context | Price (OpenRouter) | Tool Calling |
|---|---|---|---|---|---|---|
| **MiniMax M2.5** | 230B | **80.2%** | MIT | 196K | $0.28/$1.00 | Strong |
| **GLM-5** | 744B/40B | 77.8% | MIT | — | $0.80/$1.00 | Strong |
| **Kimi K2.5** | 1T/32B | 76.8% | MIT | 256K | $0.50/$2.00 | Strong (100 sub-agents) |
| **Step 3.5 Flash** | 196B/11B | 74.4% | Apache 2.0 | — | **$0.10/$0.30** | Good |
| **MiMo-V2-Flash** | 309B/15B | 73.4% | MIT | — | **$0.10/$0.30** | **Built for agents** |
| **DeepSeek V3.2** | 685B/37B | 73.0% | MIT | — | **$0.25/$0.38** | Strong + thinking |
| **Devstral 2** | 123B | 72.2% | Modified MIT | 256K | **$0.05/$0.22** | Native code agent |
| **Qwen3-Coder-Next** | 80B/3B | 70.6% | Apache 2.0 | — | — | SWE-rebench Pass@5 #1 |
| **Qwen 3.5 (397B)** | 397B/17B | — | Apache 2.0 | 262K+ | — | BFCL V4: 72.2% |

**The open-source gap has nearly closed.** MiniMax M2.5 at 80.2% SWE-bench matches Claude Opus 4.5. Six open models exceed 73% (where Claude Haiku sits). Open models cost 10-100x less.

### Best Local Models (24GB VRAM, for your RTX 5090)

| Model | Type | VRAM (Q4) | Coding Quality | Speed | Best For |
|---|---|---|---|---|---|
| **GLM-4.7-Flash** | 30B MoE | Fits 24GB | Coding Index 25.9 | Good | **Best local coding agent** |
| **Qwen3-Coder-30B-A3B** | 30B MoE | ~18GB | SWE-bench 50.3% | Very fast | Dedicated coding |
| **Devstral Small 2** | 24B | Fits 24GB | SWE-bench 72.2% (full) | Good | **Native tool calling** |
| **Mistral Small 3.2** | 24B | Fits 24GB | Good | Good | **Most reliable tool calling** |
| **Qwen 3.5 27B** | 27B dense | ~22GB | Ties GPT-5 Mini SWE-bench | Good | General + code |
| **Qwen 3.5 35B-A3B** | 35B MoE | ~22GB | Good | **112 tok/s** | Fast agentic loops |
| **GPT-OSS 20B** | 21B MoE | Fits 16GB | Decent | Good | Edge deployment, tool use |

### Free Models on OpenRouter (rate-limited but $0)

GPT-5.4 Nano, DeepSeek R1, Qwen3 Coder 480B, MiMo-V2-Flash, Devstral, Nemotron 3 Super 120B, Llama 3.3 70B, Mistral Small 3.1. Plus **Qwen 3.6 Plus Preview** (free during preview, 1M context, beats Opus 4.5 on Terminal-Bench).

### Tool Calling Quality Ranking (BFCL V4)

This is what matters most for coding agents. Tool calling ≠ general intelligence.

| Rank | Model | BFCL V4 Score | Notes |
|---|---|---|---|
| 1 | **GLM-4.5 (FC)** | **70.85%** | Best tool caller overall |
| 2 | **Claude Opus 4.1** | 70.36% | Anthropic dominates tool use |
| 3 | **Claude Sonnet 4** | 70.29% | Nearly identical to Opus |
| 4 | **Qwen 3.5 122B** | 72.2% | Best open-source tool caller |
| 7 | **GPT-5** | 59.22% | Notably weaker at tool use |

**Counterintuitive**: GPT-5 leads SWE-bench Pro but is significantly worse at tool calling than Claude. A model's coding ability and its tool-use reliability are separate skills.

### Priority Fallback Chain (with ChatGPT Plus + Gemini $20 + OpenRouter)

| Priority | Model | Via | Cost | Why |
|---|---|---|---|---|
| 1 | **Claude Opus 4.6** | Claude Code | Max plan | Best harness + model combo |
| 2 | **Gemini 3.1 Pro** | OpenCode | Gemini sub | 2M context, MCP Atlas #1, 2.5x cheaper than Opus |
| 3 | **GPT-5.4** | OpenCode / Codex CLI | ChatGPT Plus | Best SWE-bench Pro, included in sub |
| 4 | **DeepSeek V3.2** | OpenCode / OpenRouter | $0.25/Mtok | 90% of frontier at 1/50th cost |
| 5 | **GLM-4.7-Flash** | Ollama local | Free | Best local coding agent |
| 6 | **Qwen 3.5 35B-A3B** | Ollama local | Free | Fastest local agentic loops |
| 7 | **Qwen 3.6 Plus Preview** | OpenRouter | Free (preview) | 1M context, free right now |
| 8 | **Devstral 2** | OpenRouter | $0.05/Mtok | Cheapest non-free coding model |

---

## 13. Models You Should Know About (April 2026)

The landscape has changed dramatically. Here are models that didn't exist 3 months ago:

- **Qwen 3.6 Plus Preview** (March 30, 2026): FREE on OpenRouter during preview. 1M context, 65K output, always-on chain-of-thought, beats Opus 4.5 on Terminal-Bench 2.0 (61.6 vs 59.3).
- **MiniMax M2.7**: Self-evolving, performs 30-50% of its own RL research workflow autonomously. Supersedes M2.5.
- **MiMo-V2-Pro** (Xiaomi, March 2026): >1T total, 42B active. ClawEval 61.5 (approaching Opus 4.6 at 66.3). $1/$3 per Mtok.
- **GLM-5.1** (March 27, 2026): 94.6% of Claude Opus 4.6 coding score. Open-source. MIT license.
- **Devstral 2** (Mistral): 123B agentic coding specialist. 72.2% SWE-bench. Ships with **Mistral Vibe CLI** for end-to-end code automation.
- **Step 3.5 Flash** (StepFun): 196B/11B active. Average 81.0 across 8 benchmarks (above Opus 4.5). $0.10/$0.30 per Mtok. Topped OpenRouter Trending in 2 days.
- **GPT-OSS** (OpenAI): First open-weight since GPT-2. 120B model runs on single 80GB GPU. 20B model runs on 16GB. Apache 2.0.
- **Kimi K2.5** (Moonshot): 1T total, 384 experts. "Agent Swarm" capability: up to 100 sub-agents, 1,500 parallel tool calls. MIT license.
- **DeepSeek V3.2-Speciale**: Reasoning variant that beats GPT-5 on LiveCodeBench (88.7%). No tool calling though.
- **Grok 4.20 Beta**: Already in testing. Multi-agent architecture (4 agents debating per query). 75% SWE-bench. Grok 5 in training (rumored 6T-param MoE).
- **Holo 3** (H Company, March 31): SOTA computer-use agent (OSWorld 78.85%). NOT a coding model — it clicks on screens. 122B/10B active (API) + 35B/3B active (Apache 2.0). Irrelevant for coding agents, relevant for GUI automation.
- **Claude "Mythos"**: Leaked March 26. Anthropic's next-tier model, reportedly "by far the most powerful" they've built. In early access. No public release date.
- **GPT-5.5 "Spud"**: Pretraining complete. Expected Q2 2026.

### The meta-story of 2026

Benchmark convergence. GPT-5.4, Opus 4.6, and Gemini 3.1 Pro score within 2-3 percentage points of each other. Open-source models are trading blows with frontier models on specific benchmarks. **A mid-tier model in a great harness beats a frontier model in a bad one.** Pricing and developer experience now matter more than raw model performance.

---

## 14. Practical Recommendations

### If you hit Claude Code limits often:

1. **Install OpenCode** (`curl -fsSL https://opencode.ai/install | bash`). Configure with your existing API keys. Switch mid-session when Claude limits hit.

2. **Keep Claude Code for critical work**. The prompt caching, extended thinking, and Anthropic-optimized agentic loop are genuinely superior.

3. **Configure OpenCode sub-agents on cheaper models**. Run explore agents on Ollama, main agent on Claude/GPT.

4. **Skip OpenClaude**. OpenCode does everything it does, better, legally, with a real community.

### If you want 24/7 autonomous agents:

1. **Hermes Agent** on a dedicated machine. Simpler, safer than OpenClaw.
2. **Pinokio 7** for local app management. Auto-SKILL.md exposure.
3. **OpenClaw** only if you need the messaging gateway ecosystem. Be cautious of ClawHub security.

### If you want to extend your Claude Code rate limit:

The safest approach is **multi-source Claude** via LiteLLM passthrough:
- Personal Anthropic API key
- AWS Bedrock Claude (separate rate limits)
- Google Vertex AI Claude (separate rate limits)
- LiteLLM round-robin across all three

This gives 2-3x effective rate limit with **zero tool_use format risk**.

---

## Methodology

This comparison is based on:
- Direct source code analysis of all three repositories (not README marketing)
- Examination of tool implementations, provider systems, and message formats
- Testing of installed tools (OpenCode v1.3.13, Claude Code latest)
- Cross-referencing with community reports and issue trackers

The Claude Code analysis is based on the leaked source (512K+ lines) examined in [our previous analysis](README.md).

---

---

## 15. Benchmark Sources & References

### Leaderboards (live)
- [SWE-bench Verified Leaderboard](https://www.swebench.com/) — coding bug-fix benchmark
- [SWE-bench Pro](https://labs.scale.com/leaderboard/swe_bench_pro_public) — decontaminated version (OpenAI recommends over Verified)
- [SWE-rebench](https://swe-rebench.com/) — contamination-resistant alternative
- [LMArena / Chatbot Arena](https://arena.ai/leaderboard/text) — 5.4M+ human votes, 323 models
- [Berkeley Function Calling Leaderboard V4](https://gorilla.cs.berkeley.edu/leaderboard.html) — tool use quality
- [Artificial Analysis](https://artificialanalysis.ai/leaderboards/models) — 299 models, speed/price/quality
- [OpenRouter Rankings](https://openrouter.ai/rankings) — real usage data
- [Open LLM Leaderboard (HuggingFace)](https://huggingface.co/spaces/open-llm-leaderboard/open_llm_leaderboard)
- [BigCodeBench (HuggingFace)](https://huggingface.co/spaces/bigcode/bigcodebench-leaderboard) — code-specific

### Model announcements
- [Gemini 3.1 Pro](https://blog.google/innovation-and-ai/models-and-research/gemini-models/gemini-3-1-pro/) — Google, Feb 19 2026
- [GPT-5.4](https://openai.com/index/introducing-gpt-5-4/) — OpenAI, March 5 2026
- [Qwen 3.5](https://qwen.ai/blog?id=qwen3.5) — Alibaba, Feb 2026
- [GLM-5](https://huggingface.co/zai-org/GLM-5) — Zhipu AI, Feb 2026
- [DeepSeek V3.2](https://api-docs.deepseek.com/news/news251201) — DeepSeek, Jan 2026
- [MiMo-V2-Pro](https://venturebeat.com/technology/xiaomi-stuns-with-new-mimo-v2-pro-llm-nearing-gpt-5-2-opus-4-6-performance) — Xiaomi, March 2026
- [Grok 4.1](https://x.ai/news/grok-4-1) — xAI
- [Devstral 2](https://mistral.ai/news/devstral-2-vibe-cli) — Mistral
- [GPT-OSS](https://openai.com/index/introducing-gpt-oss/) — OpenAI open-weight

### Specialized models
- [Holo3 - H Company](https://hcompany.ai/holo3) — SOTA computer-use agent (OSWorld 78.85%), NOT a coding model
- [Holo3-35B-A3B on HuggingFace](https://huggingface.co/Hcompany/Holo3-35B-A3B) — Apache 2.0, Qwen3.5-based

### Analysis articles
- [MindStudio: GPT-5.4 vs Opus 4.6 vs Gemini 3.1 Pro](https://www.mindstudio.ai/blog/gpt-54-vs-claude-opus-46-vs-gemini-31-pro-benchmarks)
- [BenchLM: Best LLM for Coding 2026](https://benchlm.ai/blog/posts/best-llm-for-coding)
- [Best Local LLMs for 24GB VRAM](https://localllm.in/blog/best-local-llms-24gb-vram)
- [Adaline: Top Agentic LLM Models 2026](https://www.adaline.ai/blog/top-agentic-llm-models-frameworks-for-2026)

---

*Generated 2026-04-02 by [12georgiadis/claude-code-analysis](https://github.com/12georgiadis/claude-code-analysis)*
