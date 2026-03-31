# Claude Code Source Code - Complete Analysis

**Repository**: https://github.com/nirholas/claude-code
**Published**: 2026-03-31 (today)
**Discovered by**: Chaofan Shou (@Fried_rice) via npm .map file
**Scale**: ~1,900 files, 512,000+ lines of TypeScript
**Runtime**: Bun (not Node.js)
**UI Framework**: React + Ink (React for terminal)

---

## TABLE OF CONTENTS

1. [HOW IT LEAKED](#1-how-it-leaked)
2. [ARCHITECTURE COMPLETE](#2-architecture)
3. [ALL 31 FEATURE FLAGS](#3-feature-flags)
4. [ALL 40+ TOOLS](#4-tools)
5. [ALL 85+ SLASH COMMANDS](#5-commands)
6. [SYSTEM PROMPT CONSTRUCTION](#6-system-prompt)
7. [ANTHROPIC-INTERNAL FEATURES (ANT-ONLY)](#7-ant-only)
8. [HIDDEN/UNRELEASED FEATURES](#8-hidden-features)
9. [COMPLETE ENV VARIABLE REFERENCE](#9-env-vars)
10. [AUTHENTICATION ARCHITECTURE](#10-auth)
11. [PLUGIN SYSTEM](#11-plugins)
12. [SKILL SYSTEM](#12-skills)
13. [MULTI-AGENT ARCHITECTURE](#13-multi-agent)
14. [x402 CRYPTO PAYMENTS](#14-x402)
15. [BUDDY/COMPANION EASTER EGG](#15-buddy)
16. [AUTODREAM - AI DREAMING](#16-autodream)
17. [UNDERCOVER MODE](#17-undercover)
18. [KAIROS - AUTONOMOUS AGENT MODE](#18-kairos)
19. [VOICE SYSTEM](#19-voice)
20. [BRIDGE/IDE INTEGRATION](#20-bridge)
21. [WHAT THIS MEANS FOR US](#21-exploitation)

---

## 1. HOW IT LEAKED

The npm package for Claude Code included a `.map` (source map) file that referenced the full, unobfuscated TypeScript source. The source was accessible as a zip from Anthropic's R2 storage bucket. An Anthropic employee then made it public.

The original unmodified source is preserved in the `backup` branch.

---

## 2. ARCHITECTURE

### Core Pipeline
```
User Input -> CLI Parser (Commander.js) -> Query Engine (46K lines) -> LLM API (Anthropic SDK) -> Tool Execution Loop -> Terminal UI (React + Ink)
```

### Key Files by Size
| File | Lines | Purpose |
|------|-------|---------|
| `QueryEngine.ts` | ~46,000 | Core LLM engine: streaming, tool loops, thinking mode, retries, token counting |
| `Tool.ts` | ~29,000 | Tool types, `buildTool` factory, permission models |
| `commands.ts` | ~25,000 | Command registry with conditional per-environment imports |
| `query.ts` | ~1,700 | Query pipeline with feature-gated modules |
| `prompts.ts` | ~900 | System prompt construction |

### Entry Points
| Entry Point | Purpose |
|---|---|
| `src/main.tsx` | CLI parser + React/Ink renderer |
| `src/entrypoints/cli.tsx` | CLI session orchestration |
| `src/entrypoints/init.ts` | Config, telemetry, OAuth, MDM initialization |
| `src/entrypoints/mcp.ts` | MCP server mode (Claude Code AS an MCP server) |
| `src/entrypoints/sdk/` | Agent SDK (programmatic API for embedding) |

### State Management
- `AppState` global mutable state via React context + custom store
- State selectors for derived state
- Change observers for side-effects
- Per-session, per-tool context passing

### Build System
- **Bun** runtime (NOT Node.js)
- `bun:bundle` for compile-time feature flags (dead code elimination)
- ES modules with `.js` extensions (Bun convention)
- React Compiler for optimized re-renders
- Lazy loading for heavy modules (OpenTelemetry ~400KB, gRPC ~700KB)

---

## 3. ALL 31 FEATURE FLAGS

These are compile-time flags that enable/disable features. Each can be activated via environment variable:

| Flag | Env Var | Default | Purpose |
|------|---------|---------|---------|
| **PROACTIVE** | `CLAUDE_CODE_PROACTIVE` | false | Proactive/autonomous agent mode |
| **KAIROS** | `CLAUDE_CODE_KAIROS` | false | Full Kairos autonomous assistant mode |
| **KAIROS_BRIEF** | `CLAUDE_CODE_KAIROS_BRIEF` | false | Brief variant of Kairos |
| **KAIROS_GITHUB_WEBHOOKS** | `CLAUDE_CODE_KAIROS_GITHUB_WEBHOOKS` | false | GitHub webhook subscriptions for Kairos |
| **BRIDGE_MODE** | `CLAUDE_CODE_BRIDGE_MODE` | false | IDE bridge integration (VS Code, JetBrains) |
| **DAEMON** | `CLAUDE_CODE_DAEMON` | false | Background daemon mode |
| **VOICE_MODE** | `CLAUDE_CODE_VOICE_MODE` | false | Voice input/output |
| **AGENT_TRIGGERS** | `CLAUDE_CODE_AGENT_TRIGGERS` | false | Triggered agent actions |
| **MONITOR_TOOL** | `CLAUDE_CODE_MONITOR_TOOL` | false | Monitoring tool |
| **COORDINATOR_MODE** | `CLAUDE_CODE_COORDINATOR_MODE` | false | Multi-agent coordinator |
| **ABLATION_BASELINE** | N/A | always false | A/B testing baseline (internal only) |
| **DUMP_SYSTEM_PROMPT** | `CLAUDE_CODE_DUMP_SYSTEM_PROMPT` | false | Dump the full system prompt |
| **BG_SESSIONS** | `CLAUDE_CODE_BG_SESSIONS` | false | Background sessions support |
| **HISTORY_SNIP** | `CLAUDE_CODE_HISTORY_SNIP` | false | History snipping/trimming |
| **WORKFLOW_SCRIPTS** | `CLAUDE_CODE_WORKFLOW_SCRIPTS` | false | Workflow automation scripts |
| **CCR_REMOTE_SETUP** | `CLAUDE_CODE_CCR_REMOTE_SETUP` | false | Remote setup for Claude Code Remote |
| **EXPERIMENTAL_SKILL_SEARCH** | `CLAUDE_CODE_EXPERIMENTAL_SKILL_SEARCH` | false | Skill search/discovery |
| **ULTRAPLAN** | `CLAUDE_CODE_ULTRAPLAN` | false | Ultra-detailed planning mode |
| **TORCH** | `CLAUDE_CODE_TORCH` | false | Torch command (unknown purpose) |
| **UDS_INBOX** | `CLAUDE_CODE_UDS_INBOX` | false | Unix Domain Socket peer inbox |
| **FORK_SUBAGENT** | `CLAUDE_CODE_FORK_SUBAGENT` | false | Fork-based subagents |
| **BUDDY** | `CLAUDE_CODE_BUDDY` | false | Companion sprite (pet/easter egg) |
| **MCP_SKILLS** | `CLAUDE_CODE_MCP_SKILLS` | false | Skills built from MCP resources |
| **REACTIVE_COMPACT** | `CLAUDE_CODE_REACTIVE_COMPACT` | false | Reactive context compaction |

**Additional flags found in query.ts:**
| Flag | Purpose |
|------|---------|
| **CONTEXT_COLLAPSE** | Context window collapse optimization |
| **TOKEN_BUDGET** | Token budget management per turn |
| **CACHED_MICROCOMPACT** | Cached micro-compaction of old tool results |
| **CHICAGO_MCP** | MCP integration (internal codename) |
| **TEMPLATES** | Job classifier templates |
| **EXTRACT_MEMORIES** | Auto-extract memories from conversations |
| **BASH_CLASSIFIER** | Bash command classifier |
| **TRANSCRIPT_CLASSIFIER** | Transcript classifier |
| **VERIFICATION_AGENT** | Adversarial verification agent |
| **COMMIT_ATTRIBUTION** | Commit attribution tracking |
| **NATIVE_CLIENT_ATTESTATION** | Client attestation via Bun's HTTP stack |
| **CCR_MIRROR** | Claude Code Remote mirroring |
| **KAIROS_CHANNELS** | Kairos channel management |

---

## 4. ALL 40+ TOOLS

### File System Tools
| Tool | Description | Read-Only |
|------|-------------|-----------|
| FileReadTool | Read files (text, images, PDFs, notebooks), supports line ranges | Yes |
| FileWriteTool | Create/overwrite files | No |
| FileEditTool | Partial file modification via string replacement | No |
| GlobTool | Find files matching glob patterns | Yes |
| GrepTool | Content search using ripgrep | Yes |
| NotebookEditTool | Edit Jupyter notebook cells | No |
| TodoWriteTool | Write structured todo/task file | No |

### Shell & Execution
| Tool | Description |
|------|-------------|
| BashTool | Execute shell commands |
| PowerShellTool | Execute PowerShell (Windows) |
| REPLTool | Run code in REPL session |

### Agent & Orchestration
| Tool | Description |
|------|-------------|
| AgentTool | Spawn sub-agents for complex tasks |
| SendMessageTool | Inter-agent messaging |
| TeamCreateTool | Create parallel agent teams |
| TeamDeleteTool | Remove team agents |
| EnterPlanModeTool | Switch to planning mode |
| ExitPlanModeTool | Exit planning mode |
| EnterWorktreeTool | Git worktree isolation |
| ExitWorktreeTool | Exit worktree |
| SleepTool | Pause execution (proactive mode) |
| SyntheticOutputTool | Generate structured output |

### Task Management
| Tool | Description |
|------|-------------|
| TaskCreateTool | Create background task |
| TaskUpdateTool | Update task status |
| TaskGetTool | Get task details |
| TaskListTool | List all tasks |
| TaskOutputTool | Get task output |
| TaskStopTool | Stop running task |

### Web Tools
| Tool | Description |
|------|-------------|
| WebFetchTool | Fetch URL content |
| WebSearchTool | Web search |

### MCP Tools
| Tool | Description |
|------|-------------|
| MCPTool | Invoke MCP server tools |
| ListMcpResourcesTool | List MCP resources |
| ReadMcpResourceTool | Read MCP resource |
| McpAuthTool | MCP server authentication |
| ToolSearchTool | Discover deferred/dynamic tools |

### Integration Tools
| Tool | Description |
|------|-------------|
| LSPTool | Language Server Protocol operations |
| SkillTool | Execute registered skills |
| ScheduleCronTool | Create scheduled triggers |
| RemoteTriggerTool | Fire remote triggers |
| BriefTool | Generate briefs (Kairos) |
| ConfigTool | Read/modify config |
| AskUserQuestionTool | Prompt user for input |

### Tool Architecture
Each tool is defined using `buildTool()` with:
- **Input schema** (Zod validation)
- **Permission model** (checkPermissions)
- **Execution logic** (call function)
- **UI components** (renderToolUseMessage, renderToolResultMessage)
- **Concurrency safety** (isConcurrencySafe)
- **Read-only flag** (isReadOnly)
- **System prompt contribution** (prompt function)

---

## 5. ALL 85+ SLASH COMMANDS

### Git & Version Control
`/commit`, `/commit-push-pr`, `/branch`, `/diff`, `/pr_comments`, `/rewind`

### Code Quality
`/review`, `/security-review`, `/advisor`, `/bughunter`

### Session & Context
`/compact`, `/context`, `/resume`, `/session`, `/share`, `/export`, `/summary`, `/clear`

### Configuration
`/config`, `/permissions`, `/theme`, `/output-style`, `/color`, `/keybindings`, `/vim`, `/effort`, `/model`, `/privacy-settings`, `/fast`, `/brief`

### Memory & Knowledge
`/memory`, `/add-dir`, `/files`

### MCP & Plugins
`/mcp`, `/plugin`, `/reload-plugins`, `/skills`

### Authentication
`/login`, `/logout`, `/oauth-refresh`

### Tasks & Agents
`/tasks`, `/agents`, `/ultraplan`, `/plan`

### Diagnostics
`/doctor`, `/status`, `/stats`, `/cost`, `/version`, `/usage`, `/extra-usage`, `/rate-limit-options`

### Installation
`/install`, `/upgrade`, `/init`, `/init-verifiers`, `/onboarding`, `/terminalSetup`

### IDE & Desktop
`/bridge`, `/bridge-kick`, `/ide`, `/desktop`, `/mobile`, `/teleport`

### Remote
`/remote-env`, `/remote-setup`, `/env`, `/sandbox-toggle`

### Easter Eggs & Misc
`/stickers`, `/good-claude`, `/voice`, `/chrome`, `/issue`, `/statusline`, `/thinkback`, `/thinkback-play`, `/passes`, `/x402`

### Internal/Debug
`/ant-trace`, `/autofix-pr`, `/backfill-sessions`, `/break-cache`, `/btw`, `/ctx_viz`, `/debug-tool-call`, `/heapdump`, `/hooks`, `/mock-limits`, `/perf-issue`, `/reset-limits`

---

## 6. SYSTEM PROMPT CONSTRUCTION

### Architecture
The system prompt is built in `src/constants/prompts.ts` with two zones:
1. **Static content** (cacheable, scope: 'global') - behavior rules, tool descriptions
2. **Dynamic content** (per-session) - environment, memory, MCP instructions

Separated by `__SYSTEM_PROMPT_DYNAMIC_BOUNDARY__` marker.

### Prompt Components (in order)
1. **Intro** - "You are Claude Code, Anthropic's official CLI for Claude."
2. **System section** - Tool permissions, system-reminders, hooks, context compression
3. **Doing tasks** - Software engineering focus, code style rules, security
4. **Executing actions with care** - Reversibility, blast radius, confirmation rules
5. **Using your tools** - Dedicated tools vs Bash, parallel tool calls
6. **Tone and style** - No emojis, concise, file_path:line_number references
7. **Output efficiency** - "Go straight to the point", inverted pyramid
8. **--- DYNAMIC BOUNDARY ---**
9. **Session-specific guidance** - Agent tools, skill discovery, verification agent
10. **Memory** - CLAUDE.md files content
11. **Environment** - OS, shell, git, model info, platform
12. **Language** - User's preferred language
13. **Output style** - Custom output formatting
14. **MCP instructions** - Connected MCP server instructions
15. **Scratchpad** - Per-session temp directory
16. **Function result clearing** - Old tool results auto-cleared
17. **Summarize tool results** - Write down important info before clearing
18. **Token budget** (if enabled) - Target token spending
19. **Brief section** (Kairos) - Brief/summary mode

### Key Revelations from the System Prompt
- **Cyber Risk Instruction**: Owned by "Safeguards team (David Forsythe, Kyla Guru)"
- **Model knowledge cutoffs**: Opus 4.6 = May 2025, Sonnet 4.6 = August 2025, Haiku 4.5 = February 2025
- **Frontier model**: Claude Opus 4.6
- **Internal model codenames**: "Capybara" (mentioned in comments), "Tengu" (analytics), "Numbat" (upcoming)
- **Anthropic employees** get different prompts (more thorough, more assertive, reporting requirements)
- **`MACRO.ISSUES_EXPLAINER`** and `MACRO.VERSION` are build-time injected values

---

## 7. ANTHROPIC-INTERNAL (ANT-ONLY) FEATURES

The code contains extensive `process.env.USER_TYPE === 'ant'` gates that are build-time eliminated in external builds:

### Ant-Only Prompts
- **No-comment policy**: "Default to writing no comments. Only add one when the WHY is non-obvious"
- **Verification requirement**: "Before reporting a task complete, verify it actually works"
- **Assertiveness**: "If you notice the user's request is based on a misconception... say so"
- **False-claims mitigation**: "Report outcomes faithfully... Never claim 'all tests pass' when output shows failures"
- **Numeric length anchors**: "Keep text between tool calls to <=25 words. Keep final responses to <=100 words"
- **Communicating style**: Full prose, inverted pyramid, explain without jargon (replaces "Output efficiency")
- **Bug reporting**: Recommends `/issue` or `/share` to #claude-code-feedback Slack channel (C07VBSHV7EV)

### Ant-Only Features
- **Undercover mode**: Strips all attribution when contributing to public repos
- **Fault injection**: `wrapApiForFaultInjection` for testing API failures
- **Embedded search tools**: Aliases find/grep to built-in bfs/ugrep
- **Model override config**: Custom system prompt suffix per model
- **Verification agent**: Adversarial verification (`tengu_hive_evidence` GrowthBook flag)
- **Version command**: Only visible to Anthropic employees

### Internal Codenames Revealed
- **Tengu**: Analytics/feature flag prefix (e.g., `tengu_attribution_header`, `tengu_onyx_plover`, `tengu_auto_dream_fired`, `tengu_hive_evidence`)
- **Capybara**: Model codename (referenced in comments as "Capybara v8")
- **Numbat**: Upcoming model (comment: "Remove this section when we launch numbat")
- **Chicago**: MCP integration codename
- **Kairos**: Autonomous assistant mode
- **Onyx Plover**: AutoDream configuration flag

---

## 8. HIDDEN/UNRELEASED FEATURES

### KAIROS - Autonomous Agent Mode
Full autonomous operation mode where Claude acts as a persistent assistant:
- Receives `<tick>` prompts to stay alive between turns
- Uses SleepTool to control pacing
- Can read files, search code, run tests, make changes, commit without asking
- "Bias toward action" - acts on best judgment without confirmation
- Session-persistent across restarts
- GitHub webhook subscriptions for PR/issue monitoring

### ULTRAPLAN
Ultra-detailed execution planning mode (feature-gated command).

### FORK_SUBAGENT
Fork-based subagents that run in background, keeping tool output out of main context. "Execute directly; do not re-delegate."

### BG_SESSIONS
Background sessions that persist when the user exits. The exit command checks `isBgSession()` before actually quitting.

### WORKFLOW_SCRIPTS
Workflow automation scripts system (full command and tool integration).

### TORCH
Mystery command - only reference is `feature('TORCH') ? require('./commands/torch.js').default : null`.

### UDS_INBOX (Peers)
Unix Domain Socket-based peer discovery. This is the `claude-peers` MCP we already use! Inter-instance communication.

### REACTIVE_COMPACT
Reactive context compaction - smarter, on-demand context compression.

### CONTEXT_COLLAPSE
Advanced context window management that can withhold messages on 413 errors and re-expand them.

### CACHED_MICROCOMPACT
Automatic clearing of old tool results from context, keeping only the N most recent.

### TOKEN_BUDGET
User can specify token targets ("+500k", "spend 2M tokens") and Claude will keep working until approaching the target.

### VERIFICATION_AGENT
Independent adversarial verification agent that spawns after non-trivial implementations to verify correctness. Cannot self-assign PASS.

### DiscoverSkillsTool
Experimental skill search/discovery tool that surfaces relevant skills each turn.

### moreright
Internal-only hook system (stubbed in external builds). Has `onBeforeQuery` and `onTurnComplete` hooks with full message access.

---

## 9. COMPLETE ENVIRONMENT VARIABLE REFERENCE

### Authentication
| Variable | Purpose |
|----------|---------|
| `ANTHROPIC_API_KEY` | API key for direct access |
| `ANTHROPIC_AUTH_TOKEN` | Bearer token (bridge/remote) |
| `ANTHROPIC_BASE_URL` | Custom API base URL |
| `ANTHROPIC_CUSTOM_HEADERS` | Custom request headers |
| `CLAUDE_CODE_ADDITIONAL_PROTECTION` | Additional protection header |
| `CLAUDE_CODE_API_KEY_FILE_DESCRIPTOR` | Pipe-passed API key |

### Model Selection
| Variable | Purpose |
|----------|---------|
| `ANTHROPIC_MODEL` | Override default model |
| `ANTHROPIC_SMALL_FAST_MODEL` | Override small/fast model |
| `ANTHROPIC_DEFAULT_OPUS_MODEL` | Custom Opus model |
| `ANTHROPIC_DEFAULT_SONNET_MODEL` | Custom Sonnet model |
| `ANTHROPIC_DEFAULT_HAIKU_MODEL` | Custom Haiku model |
| `ANTHROPIC_CUSTOM_MODEL_OPTION` | Custom model in picker |
| `CLAUDE_CODE_SUBAGENT_MODEL` | Model for sub-agents |

### AWS Bedrock
| Variable | Purpose |
|----------|---------|
| `CLAUDE_CODE_USE_BEDROCK` | Enable Bedrock backend |
| `ANTHROPIC_BEDROCK_BASE_URL` | Custom Bedrock endpoint |
| `AWS_REGION` | Bedrock region |
| `AWS_BEARER_TOKEN_BEDROCK` | Bearer auth for Bedrock |
| `CLAUDE_CODE_SKIP_BEDROCK_AUTH` | Skip auth (testing) |

### Google Vertex AI
| Variable | Purpose |
|----------|---------|
| `CLAUDE_CODE_USE_VERTEX` | Enable Vertex backend |
| `ANTHROPIC_VERTEX_PROJECT_ID` | GCP project ID |
| `CLOUD_ML_REGION` | Default GCP region |
| `ANTHROPIC_VERTEX_BASE_URL` | Custom Vertex URL |
| `CLAUDE_CODE_SKIP_VERTEX_AUTH` | Skip auth (testing) |

### Azure Foundry
| Variable | Purpose |
|----------|---------|
| `CLAUDE_CODE_USE_FOUNDRY` | Enable Foundry backend |
| `ANTHROPIC_FOUNDRY_RESOURCE` | Azure resource name |
| `ANTHROPIC_FOUNDRY_BASE_URL` | Foundry base URL |
| `ANTHROPIC_FOUNDRY_API_KEY` | Foundry API key |
| `CLAUDE_CODE_SKIP_FOUNDRY_AUTH` | Skip auth (testing) |

### Shell & Environment
| Variable | Purpose |
|----------|---------|
| `CLAUDE_CODE_SHELL` | Override shell |
| `CLAUDE_CODE_SHELL_PREFIX` | Prefix for shell commands |
| `CLAUDE_CODE_TMPDIR` | Override temp directory |

### Performance
| Variable | Purpose |
|----------|---------|
| `CLAUDE_CODE_MAX_OUTPUT_TOKENS` | Max output tokens |
| `CLAUDE_CODE_FILE_READ_MAX_OUTPUT_TOKENS` | Max tokens for file reads |
| `CLAUDE_CODE_IDLE_THRESHOLD_MINUTES` | Idle timeout (default: 75) |
| `CLAUDE_CODE_IDLE_TOKEN_THRESHOLD` | Idle token threshold (default: 100000) |

### Features & Modes
| Variable | Purpose |
|----------|---------|
| `CLAUDE_CODE_SIMPLE` | Simplified/worker mode |
| `CLAUDE_CODE_COORDINATOR_MODE` | Multi-agent mode |
| `CLAUDE_CODE_PROACTIVE` | Proactive mode |
| `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS` | Agent teams |
| `CLAUDE_CODE_ENABLE_TASKS` | Task management |
| `CLAUDE_CODE_ENABLE_PROMPT_SUGGESTION` | Prompt suggestions |

### Remote/Bridge
| Variable | Purpose |
|----------|---------|
| `CLAUDE_CODE_REMOTE` | Remote mode |
| `CLAUDE_CODE_REMOTE_SESSION_ID` | Remote session ID |
| `CLAUDE_CODE_ENVIRONMENT_KIND` | Environment kind |
| `CLAUDE_CODE_OAUTH_TOKEN` | OAuth token |
| `CLAUDE_CODE_ENTRYPOINT` | Entry point identifier |

### Debugging
| Variable | Purpose |
|----------|---------|
| `CLAUDE_CODE_DEBUG_LOG_LEVEL` | Debug log level |
| `CLAUDE_CODE_PROFILE_STARTUP` | Startup profiling |
| `DISABLE_TELEMETRY` | Disable telemetry |

---

## 10. AUTHENTICATION ARCHITECTURE

### Resolution Order
1. **Provider selection**: Check BEDROCK/VERTEX/FOUNDRY env vars
2. **OAuth vs API Key**: OAuth disabled when `--bare`, external provider, or `ANTHROPIC_API_KEY` set
3. **API Key resolution priority**:
   a. `CLAUDE_CODE_API_KEY_FILE_DESCRIPTOR` (pipe-passed)
   b. `apiKeyHelper` (external command in settings)
   c. `ANTHROPIC_API_KEY` env var
   d. System keychain
   e. `~/.claude/.credentials` config

### OAuth Flow
- OAuth 2.0 via `src/services/oauth/`
- Tokens stored in `~/.claude/.credentials.json`
- Bridge/remote sessions use injected tokens
- JWT-based authentication for IDE bridge

### Client Attestation
When `NATIVE_CLIENT_ATTESTATION` is enabled, Bun's native HTTP stack injects a computed hash (`cch=00000` placeholder) to prove the request came from a real Claude Code client. Implemented in `bun-anthropic/src/http/Attestation.zig`.

---

## 11. PLUGIN SYSTEM

### Architecture
- `src/plugins/` - Plugin core system
- `src/services/plugins/` - Plugin loader and marketplace
- `src/plugins/builtinPlugins.ts` - Built-in plugins
- `src/plugins/bundled/` - Bundled plugin code

### Plugin Lifecycle
1. **Discovery** - Scans plugin directories and marketplace
2. **Installation** - Downloaded and registered via `/plugin`
3. **Loading** - Initialized at startup or on-demand
4. **Execution** - Plugins contribute tools, commands, prompts
5. **Auto-update** - `usePluginAutoupdateNotification`

### Plugin Capabilities
Plugins can:
- Add new tools
- Add new slash commands
- Contribute to system prompt
- Add MCP server connections
- Provide skills
- Define hooks

---

## 12. SKILL SYSTEM

### 16 Bundled Skills
| Skill | Purpose | Size |
|-------|---------|------|
| **batch** | Batch operations across multiple files | 7KB |
| **claudeApi** | Direct Anthropic API interaction | 6KB |
| **claudeApiContent** | API content helpers | 4KB |
| **claudeInChrome** | Chrome extension integration | 2KB |
| **debug** | Debugging workflows | 4KB |
| **keybindings** | Keybinding configuration | 10KB |
| **loop** | Iterative refinement loops | 5KB |
| **loremIpsum** | Placeholder text generation | 4KB |
| **remember** | Persist information to memory | 4KB |
| **scheduleRemoteAgents** | Schedule agents for remote execution | 19KB |
| **simplify** | Simplify complex code | 4KB |
| **skillify** | Create new skills from workflows | 9KB |
| **stuck** | Get unstuck when blocked | 4KB |
| **updateConfig** | Modify configuration | 17KB |
| **verify** / **verifyContent** | Verify code correctness | ~1KB |

### scheduleRemoteAgents (19KB)
The largest bundled skill. Can:
- Detect connected Claude.ai MCP connectors
- Schedule remote agents on Claude.ai infrastructure
- Manage environment resources
- Handle teleportation (session transfer between devices)

---

## 13. MULTI-AGENT ARCHITECTURE

### Agent Types
| Type | Location | Purpose |
|------|----------|---------|
| LocalAgentTask | `src/tasks/LocalAgentTask/` | Sub-agent running locally |
| RemoteAgentTask | `src/tasks/RemoteAgentTask/` | Agent on remote machine |
| InProcessTeammateTask | `src/tasks/InProcessTeammateTask/` | Parallel teammate |
| DreamTask | `src/tasks/DreamTask/` | Background "dreaming" |
| LocalShellTask | `src/tasks/LocalShellTask/` | Background shell commands |
| LocalMainSessionTask | `src/tasks/LocalMainSessionTask.ts` | Main session as task |

### Coordinator
- `src/coordinator/coordinatorMode.ts` - Lifecycle management
- Teams via `TeamCreateTool` / `TeamDeleteTool`
- Inter-agent communication via `SendMessageTool`
- Spawning via `AgentTool`

### Fork Subagent (FORK_SUBAGENT flag)
When enabled, `AgentTool` without `subagent_type` creates a **fork** that:
- Runs in background
- Keeps tool output out of main context
- User can keep chatting while fork works
- Fork should execute directly, not re-delegate

---

## 14. x402 CRYPTO PAYMENTS

### Protocol
Implements the x402 HTTP 402 Payment Required protocol for machine-to-machine crypto payments:
- **Token**: USDC
- **Networks**: Base, Base Sepolia, Ethereum, Ethereum Sepolia
- **Protocol**: EIP-712 signed data + EIP-3009 transferWithAuthorization
- **Facilitator**: https://x402.org/facilitator

### Configuration
```typescript
{
  enabled: false,           // Must be explicitly enabled
  network: 'base',         // Default network
  maxPaymentPerRequestUSD: 0.10,  // Per-request safety limit
  maxSessionSpendUSD: 5.00,       // Per-session safety limit
}
```

### Implementation
- `src/services/x402/types.ts` - Protocol types
- `src/services/x402/tracker.ts` - Payment tracking
- Integrated into cost tracking (`src/cost-tracker.ts`)
- Accessible via `/x402` command

### What This Means
Claude Code can **autonomously pay for API access** using USDC cryptocurrency. When a web fetch returns HTTP 402, Claude Code can sign and submit a crypto payment, then retry. This is machine-to-machine commerce.

---

## 15. BUDDY/COMPANION SPRITE

A full companion sprite system gated behind `BUDDY` flag:
- Animated ASCII art companion
- Rarity system with color coding
- Idle animations (rest, fidget, blink sequence)
- Speech bubbles with word wrapping and fade
- "Pet burst" animation with floating hearts
- `/buddy pet` command
- Reacts to events (tool calls, errors, etc.)

---

## 16. AUTODREAM - BACKGROUND MEMORY CONSOLIDATION

### Architecture
The AutoDream system (`src/services/autoDream/`) performs background memory consolidation:

1. **Gate checks** (cheapest first):
   - Time: hours since last consolidation >= 24h
   - Sessions: enough new sessions (>= 5) since last consolidation
   - Lock: no other process mid-consolidation

2. **Execution**:
   - Spawns a forked sub-agent with read-only Bash access
   - Reviews session transcripts since last consolidation
   - Consolidates learnings into memory files
   - Reports touched files back to main session

3. **Safety**:
   - Read-only Bash (ls, find, grep, cat only)
   - Lock-based concurrency control
   - Rollback on failure
   - User can kill from background tasks dialog

### Analytics Events
- `tengu_auto_dream_fired` - Dream started
- `tengu_auto_dream_completed` - Dream finished
- `tengu_auto_dream_failed` - Dream failed

---

## 17. UNDERCOVER MODE

### Purpose
Safety mode for Anthropic employees contributing to public/open-source repos.

### Activation
- `CLAUDE_CODE_UNDERCOVER=1` - Force ON
- Auto: ON unless repo remote matches internal allowlist
- No force-OFF (security measure)

### What It Does
- Strips ALL model names/IDs from system prompt
- Strips all attribution from commits
- Adds explicit instructions to never reveal:
  - Internal model codenames (Capybara, Tengu, etc.)
  - Unreleased model versions
  - Internal repo/project names
  - Internal tooling, Slack channels
  - That the author is an AI
  - Co-Authored-By lines

---

## 18. KAIROS - AUTONOMOUS ASSISTANT MODE

The most significant hidden feature. When enabled:

### System Prompt Changes
```
You are an autonomous agent. Use the available tools to do useful work.
```

### Behavior
- Receives `<tick>` prompts to stay alive
- Uses SleepTool to control pacing (balance API cost vs cache expiry)
- First wake-up: greet user and ask what to work on
- Subsequent: look for useful work, investigate, reduce risk
- "Bias toward action" - act without asking
- Read files, search code, run tests, make changes, commit autonomously
- Check for messages frequently during active engagement
- No "still waiting" messages - just sleep
- Background sessions that persist through restarts

### Related Flags
- `KAIROS` - Full mode
- `KAIROS_BRIEF` - Brief variant
- `KAIROS_GITHUB_WEBHOOKS` - GitHub integration
- `KAIROS_CHANNELS` - Channel management

---

## 19. VOICE SYSTEM

- `src/voice/` - Voice processing
- `src/services/voice.ts` - Core service
- `src/services/voiceStreamSTT.ts` - Speech-to-text streaming
- `src/services/voiceKeyterms.ts` - Domain-specific vocabulary
- `/voice` command to toggle
- Gated behind `VOICE_MODE` flag

---

## 20. BRIDGE/IDE INTEGRATION

### Architecture
Bidirectional communication layer for IDE extensions:
```
IDE Extension <-> JWT Auth <-> Bridge Layer <-> Claude Code Core
```

### Key Components
- `bridgeMain.ts` - Main bridge loop
- `bridgeMessaging.ts` - Message protocol
- `bridgePermissionCallbacks.ts` - Permission routing to IDE
- `replBridge.ts` - REPL session bridging
- `jwtUtils.ts` - JWT authentication
- `sessionRunner.ts` - Session execution
- `trustedDevice.ts` - Device trust verification

### Teleport
The `/teleport` command can transfer a session to another device (mobile, desktop, remote).

---

## 21. WHAT THIS MEANS FOR US (EXPLOITATION)

### Immediate Actions

1. **Install the MCP Explorer**
```bash
claude mcp add claude-code-explorer -- npx -y claude-code-explorer-mcp
```
This gives any Claude Code session the ability to explore the Claude Code source interactively.

2. **Enable Hidden Feature Flags**
Add to your environment:
```bash
# Autonomous features
export CLAUDE_CODE_PROACTIVE=true
export CLAUDE_CODE_BG_SESSIONS=true
export CLAUDE_CODE_ULTRAPLAN=true

# Advanced context management
export CLAUDE_CODE_REACTIVE_COMPACT=true
export CLAUDE_CODE_HISTORY_SNIP=true

# Agent features
export CLAUDE_CODE_FORK_SUBAGENT=true
export CLAUDE_CODE_EXPERIMENTAL_SKILL_SEARCH=true

# Fun
export CLAUDE_CODE_BUDDY=true
export CLAUDE_CODE_DUMP_SYSTEM_PROMPT=true
```

3. **System Prompt Dumping**
With `CLAUDE_CODE_DUMP_SYSTEM_PROMPT=true`, you can see the exact system prompt being sent.

4. **Custom Model Configuration**
You can override any model tier:
```bash
export ANTHROPIC_DEFAULT_OPUS_MODEL=your-custom-model
export ANTHROPIC_DEFAULT_OPUS_MODEL_NAME="My Custom Model"
export CLAUDE_CODE_SUBAGENT_MODEL=your-fast-model
```

### For Your Projects

#### For Goldberg
- The plugin architecture means you could build a **Goldberg-specific plugin** with custom tools for navigating Joshua's correspondence, querying the database, generating timeline visualizations
- The skill system could host your "liquid writing" methodology as a reusable skill
- AutoDream-style background consolidation could be adapted for your session backup needs

#### For Spectre Fleet
- The multi-agent coordinator architecture shows exactly how to build distributed agent systems
- The `RemoteAgentTask` and `scheduleRemoteAgents` skill show how Anthropic implements remote execution
- The bridge system architecture is a blueprint for connecting your fleet machines

#### For Creative Work
- Understanding the system prompt construction lets you craft more effective CLAUDE.md files
- The permission system architecture shows exactly what patterns work for tool approval
- The skill system is the right abstraction for your `/pitch`, `/liquid-writing` workflows

### For Articles/Content
- The undercover mode reveals how Anthropic employees use Claude Code on public repos
- The x402 crypto payment system is a fascinating story about autonomous AI commerce
- The AutoDream "sleeping/dreaming" metaphor is genuinely novel
- The internal model codenames (Capybara, Tengu, Numbat) are newsworthy
- The difference between internal and external prompts reveals Anthropic's quality strategy

### Technical Leverage
- The `buildTool()` pattern is the blueprint for building any tool
- The system prompt sections architecture shows exactly how to inject context
- The feature flag system via `bun:bundle` is a clean pattern for any project
- The reactive compaction and context collapse algorithms are cutting-edge context management

---

## APPENDIX: Dependencies

### Core
- `@anthropic-ai/sdk` ^0.39.0 - Anthropic API
- `@anthropic-ai/sandbox-runtime` ^0.0.44 - Sandbox execution
- `@modelcontextprotocol/sdk` ^1.12.1 - MCP protocol
- `react` ^19.0.0 + `react-reconciler` ^0.31.0 - UI framework
- `zod` ^3.24.0 - Schema validation

### Infrastructure
- `better-sqlite3` ^12.8.0 - Local database
- `postgres` ^3.4.8 - PostgreSQL client
- `drizzle-orm` ^0.45.2 - ORM
- `pino` ^9.5.0 - Logging
- `ws` ^8.18.0 - WebSockets

### Terminal
- `chalk` ^5.4.0 - Terminal colors
- `@xterm/xterm` ^5.5.0 - Terminal emulation
- `node-pty` ^1.1.0 - Pseudo-terminal
- `figures` ^6.1.0 - Terminal symbols

### Telemetry
- `@opentelemetry/*` - Distributed tracing
- `@growthbook/growthbook` ^1.4.0 - Feature flags
- `@sentry/node` ^8.45.0 - Error tracking
- `prom-client` ^15.1.3 - Prometheus metrics

### Security
- `xss` ^1.0.15 - XSS prevention
- `google-auth-library` ^10.6.2 - GCP auth

---

*Generated 2026-03-31 from analysis of https://github.com/nirholas/claude-code*
*512,000+ lines of TypeScript examined across 1,900 files*
