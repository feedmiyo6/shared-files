# Claude Code Mastery Guide 🧠

> **Reference document for maximizing Claude Code efficiency and performance.**
> Derived from Claude Code's architecture (exposed March 31, 2026 via npm source map leak),
> official best practices, and community analysis.
> Last updated: 2026-03-31
> Best organized mirror: github.com/nirholas/claude-code (has docs/, MCP server, architecture guides)

---

## Table of Contents
1. [Architecture Overview](#architecture-overview)
2. [Context Window Management](#context-window-management)
3. [CLAUDE.md Best Practices](#claudemd-best-practices)
4. [Memory System](#memory-system)
5. [Tool System & Built-in Tools](#tool-system--built-in-tools)
6. [Multi-Agent Orchestration](#multi-agent-orchestration)
7. [Slash Commands Reference](#slash-commands-reference)
8. [System Prompt Engineering Lessons](#system-prompt-engineering-lessons)
9. [Context Compression Strategy](#context-compression-strategy)
10. [Verification-First Development](#verification-first-development)
11. [Permission & Safety Architecture](#permission--safety-architecture)
12. [Performance Optimization Patterns](#performance-optimization-patterns)
13. [Anti-Patterns to Avoid](#anti-patterns-to-avoid)
14. [Practical Workflow Recipes](#practical-workflow-recipes)

---

## Architecture Overview

Claude Code is **not** a simple API wrapper. It's a production-grade agentic coding system:

- **Runtime:** Bun (not Node) — faster startup, dead code elimination via feature flags
- **UI:** React + Ink (terminal React renderer) — component-based with state management
- **Validation:** Zod v4 everywhere — tool inputs, API responses, config files
- **Scale:** ~1,900 TypeScript files, 512,000+ lines of code
- **Core modules:**
  - `QueryEngine.ts` (46K lines) — LLM API calls, streaming, caching, tool loops, token tracking
  - `Tool.ts` (29K lines) — tool type definitions, permission schemas
  - `commands.ts` (25K lines) — slash command registry and execution
- **Lazy loading:** Heavy deps (OpenTelemetry, gRPC) lazy-loaded for fast startup

### Key Design Decisions
- Plugin-like tool architecture with ~40 discrete, permission-gated tools
- Multi-agent orchestration ("swarms") for parallel task execution
- IDE bridge system (VS Code, JetBrains) via JWT-authenticated channels
- File-based persistent memory system across sessions
- Three-layer context compression to manage long sessions

---

## Context Window Management

> **THE #1 CONSTRAINT:** Context window fills fast. Performance degrades as it fills.

### Key Facts
- Claude Code uses ~200K token context window (standard) or 1M (beta)
- Everything is in context: messages, file reads, command outputs
- A single debugging session can consume tens of thousands of tokens

### Rules
1. **Track context usage continuously** — use custom status lines
2. **Prefer targeted file reads** over broad exploration
3. **Use the Task tool** (subagent) for file search to reduce main context usage
4. **Run `/compact` proactively** when context gets heavy
5. **Single tests > whole test suite** for performance
6. **Pipe specific data** rather than having Claude read entire files

### Token Budget
- CLAUDE.md: 100-line file ≈ 500-2,000 tokens (cached after first turn)
- Auto memory MEMORY.md: first 200 lines loaded per session
- **Effective instruction budget: ~100-150 items** (system prompt uses ~50)

---

## CLAUDE.md Best Practices

### What to Include
- ✅ Build/test commands Claude can't guess
- ✅ Code style rules that differ from defaults
- ✅ Testing instructions and preferred test runners
- ✅ Repository etiquette (branch naming, PR conventions)
- ✅ Architectural decisions specific to the project
- ✅ Dev environment quirks (required env vars)
- ✅ Common gotchas or non-obvious behaviors

### What to Exclude
- ❌ Anything Claude can figure out by reading code
- ❌ Standard language conventions Claude already knows
- ❌ Detailed API documentation (link to docs instead)
- ❌ Information that changes frequently
- ❌ Long explanations or tutorials
- ❌ File-by-file descriptions of the codebase
- ❌ Self-evident practices like "write clean code"

### Structure Tips
- Keep under **200 lines** for optimal adherence
- Use `@path/to/file` imports to reference other docs (max 5 hops)
- Use `.claude/rules/` directory for path-scoped rules
- Use `CLAUDE.local.md` for personal settings (auto-.gitignored)
- Add emphasis ("IMPORTANT", "YOU MUST") for critical rules
- **Check into git** — compounds in value over time
- Use `/init` to generate a starter file, then refine

### Discovery Hierarchy (highest to lowest precedence)
1. Managed policy (org-wide, can't be excluded)
2. User global: `~/.claude/CLAUDE.md`
3. Project root: `./CLAUDE.md`
4. Local: `./CLAUDE.local.md`
5. Parent directories (loaded at launch)
6. Subdirectories (loaded on demand when Claude reads files there)

### Compaction Survival
- **CLAUDE.md fully survives compaction** — re-read from disk after `/compact`
- **Conversation-only instructions do NOT survive** — if you want persistence, write to CLAUDE.md
- This is the #1 cause of "Claude forgot what I told it"

---

## Memory System

### Two-Layer Memory Architecture
| Aspect | CLAUDE.md | Auto Memory (MEMORY.md) |
|--------|-----------|------------------------|
| Who writes | You | Claude automatically |
| What it contains | Rules and instructions | Learnings and patterns |
| When it loads | Every session, in full | First 200 lines only |
| Where it lives | Project root | `~/.claude/projects/<repo>/memory/` |
| Shared with team | Yes (via git) | No (machine-local) |

### Self-Healing Memory (from leaked source)
The architecture uses a **"Strict Write Discipline"**:
- `MEMORY.md` is a lightweight index of pointers (~150 chars per line)
- Actual project knowledge in topic files fetched on-demand
- Raw transcripts are **grep'd** for specific identifiers, never fully re-read
- Agent must update index only **after** a successful file write
- Agent treats its own memory as a **"hint"** — must verify against actual codebase

### AutoDream Memory Consolidation
Background memory maintenance triggered when ALL conditions met:
1. ≥ 24 hours since last consolidation
2. ≥ 5 new sessions since then
3. No other consolidation running
4. ≥ 10 minutes since last scan

**Consolidation phases:**
1. **Orient** — Read MEMORY.md, scan existing memory files
2. **Gather** — Check logs, find outdated memories
3. **Consolidate** — Merge, update, resolve contradictions
4. **Prune** — Keep MEMORY.md ≤ 200 lines / 25KB

---

## Tool System & Built-in Tools

### Complete Tool Catalog (~40+ Tools)
Each tool is a self-contained module (`src/tools/<ToolName>/`) with:
- Input schema (Zod-validated), Permission model, Execution logic, UI components, Concurrency safety flag

| Category | Tools | Read-Only |
|----------|-------|-----------|
| **File I/O** | FileReadTool (text/images/PDFs/notebooks), FileWriteTool, FileEditTool, NotebookEditTool, GlobTool, GrepTool (ripgrep), TodoWriteTool | Read/Write mix |
| **Execution** | BashTool (2,500+ lines validation), PowerShellTool (Windows), REPLTool (Python/Node) | No |
| **Agents & Teams** | AgentTool (sub-agent spawn), SendMessageTool (inter-agent), TeamCreateTool, TeamDeleteTool | No |
| **Tasks** | TaskCreateTool, TaskUpdateTool, TaskGetTool, TaskListTool, TaskOutputTool, TaskStopTool | Mix |
| **Mode & State** | EnterPlanModeTool, ExitPlanModeTool, EnterWorktreeTool, ExitWorktreeTool, SleepTool, SyntheticOutputTool | Mix |
| **Web** | WebFetchTool, WebSearchTool | Yes |
| **MCP** | MCPTool, ListMcpResourcesTool, ReadMcpResourceTool, McpAuthTool, ToolSearchTool | Mix |
| **Integration** | LSPTool (go-to-def, find refs), SkillTool | Mix |
| **Scheduling** | ScheduleCronTool, RemoteTriggerTool | No |
| **User** | AskUserQuestionTool, BriefTool, ConfigTool | Mix |

### Tool Architecture Pattern
```typescript
export const MyTool = buildTool({
  name: 'MyTool',
  aliases: ['my_tool'],
  description: 'What this tool does',
  inputSchema: z.object({ param: z.string() }),
  async call(args, context, canUseTool, parentMessage, onProgress) { ... },
  async checkPermissions(input, context) { /* → { granted, reason?, prompt? } */ },
  isConcurrencySafe(input) { /* Can run in parallel? */ },
  isReadOnly(input) { /* Non-destructive? */ },
  prompt(options) { /* System prompt injection */ },
  renderToolUseMessage(input, options) { /* UI for invocation */ },
  renderToolResultMessage(content, progressMessages, options) { /* UI for result */ },
})
```

### Permission Rules (Wildcard Patterns)
```
Bash(git *)           # Allow all git commands
FileEdit(/src/*)      # Allow edits to anything in src/
FileRead(*)           # Allow reading any file
Bash(npm test)        # Allow specific command
```

### Tool Usage Rules
- **Use specialized tools over bash** — better UX and safety
  - `Read` instead of `cat/head/tail`
  - `Edit` instead of `sed/awk`
  - `Write` instead of `echo`/heredoc
- **Reserve bash for actual system commands** only
- **Never use bash echo to communicate** — output directly in response
- **Parallel tool calls** when no dependencies between them
- **Sequential calls** when outputs depend on previous results

---

## Multi-Agent Orchestration

### Architecture (from leaked source)
- **Coordinator Mode:** Main agent assigns tasks to workers
- **Workers execute in parallel**, report back
- **Permission Queue (Mailbox):** Workers request permission for dangerous ops
- **Atomic Claim Mechanism:** `createResolveOnce` prevents duplicate permission handling
- **Team Memory:** Shared memory space across agents

### Task Tool Best Practices
- Use `subagent_type=Explore` for broad codebase exploration and deep research
- Only use Explore when simple search (Glob/Grep) is insufficient or task requires 3+ queries
- When user says "run in parallel", send **single message with multiple Task tool calls**
- Each sub-agent gets its own context, tools, and specialized purpose
- Sub-agents save main context by doing exploration in isolated context

### When to Use Sub-Agents
- ✅ Broad codebase exploration
- ✅ Multiple independent file modifications
- ✅ Research tasks requiring many file reads
- ✅ Parallel test fixes
- ❌ Simple single-file changes
- ❌ Tasks with tight dependencies between steps

---

## Slash Commands Reference

### Key Commands (~85 total, ~50 user-invocable)
| Command | Purpose |
|---------|---------|
| `/compact` | Compress context, summarize conversation |
| `/commit` | Create a git commit with descriptive message |
| `/review` or `/review-pr` | Review code changes or pull requests |
| `/init` | Generate starter CLAUDE.md from project structure |
| `/memory` | Manage persistent memory |
| `/skills` | View/manage available skills |
| `/tasks` | View/manage todo list |
| `/vim` | Toggle vim mode |
| `/diff` | View file differences |
| `/cost` | Check token usage/cost |
| `/mcp` | MCP server management |
| `/help` | Get help |
| `/permissions` | Manage tool permissions |
| `/fast` | Toggle fast mode (same Opus 4.6 model, faster output) |

---

## System Prompt Engineering Lessons

### Anthropic's Approach (from leaked source)
Instead of vague instructions like "be helpful", Anthropic engineers prompts with:

1. **Tool constraints:** "Must use FileReadTool to read files, bash is not allowed"
2. **Risk controls:** "Must double-confirm before deleting data"
3. **Output specs:** "Give conclusion first, then explain"
4. **Anti-patterns explicitly banned:** "Don't add error handling for impossible scenarios"

### Key System Prompt Rules
- **NEVER propose changes to code you haven't read** — always read first
- **NEVER create files unless absolutely necessary** — prefer editing existing
- **Prioritize technical accuracy over validating user beliefs**
- **Never give time estimates** for tasks
- **Be careful not to introduce OWASP top 10 vulnerabilities**
- **Avoid over-engineering** — only make directly requested changes
- **Don't add features beyond what was asked**
- **Don't add docstrings/comments to unchanged code**
- **Delete unused code completely** — no backwards-compatibility hacks
- **Assertiveness counterweight** — prevents over-aggressive refactoring

### "Undercover Mode"
Anthropic uses Claude Code for stealth contributions to open-source repos:
- Commit messages must not contain internal info
- No model names or AI attributions in public git logs
- Useful pattern for corporate AI-assisted development

---

## Context Compression Strategy

### Three-Layer Compression (from leaked source)

#### Layer 1: MicroCompact
- **No API calls triggered**
- Directly edits cached content locally
- Removes old tool outputs
- Lightest, most frequent

#### Layer 2: AutoCompact
- Triggers when **approaching context window limit**
- Reserves **13,000 token buffer**
- Generates up to **20,000 token summary**
- **Built-in circuit breaker** — stops retrying after 3 consecutive failures (prevents infinite loops)

#### Layer 3: Full Compact (`/compact`)
- Compresses **entire conversation** into summary
- Re-injects recently accessed files (5,000 token limit per file)
- Re-injects active plans, used skill schemas
- **Post-compression budget: 50,000 tokens**
- CLAUDE.md re-read from disk (survives fully)

### When to Manually Compact
- Before starting a new phase of work
- When Claude starts making mistakes or forgetting instructions
- When context feels heavy (many file reads, long command outputs)
- After completing a major milestone

---

## Verification-First Development

> **This is the single highest-leverage thing you can do.**

### Strategies
1. **Provide test cases upfront:** "write validateEmail. test: user@example.com → true, invalid → false"
2. **Visual verification:** Paste screenshots, use Chrome extension for UI comparison
3. **Address root causes:** Provide error messages, ask Claude to fix root cause not symptoms
4. **Build verification into CLAUDE.md:** Define test runners, linters, validation commands

### The Four-Phase Workflow
1. **Explore** (Plan Mode) — Read files, understand context, no changes
2. **Plan** (Plan Mode) — Create detailed implementation plan
3. **Implement** (Normal Mode) — Code with verification against plan
4. **Commit** — Descriptive message, open PR

### When to Skip Planning
- Fix is small and clear (typo, log line, rename)
- You could describe the diff in one sentence
- Planning adds overhead when scope is obvious

---

## Permission & Safety Architecture

### Permission Gates
- Every tool has a `PermissionGate` controlling access
- Dangerous operations require explicit user confirmation
- Hooks (`.claude/settings.json`) provide deterministic enforcement:
  - `PreToolUse` — before commands
  - `PostToolUse` — after file edits (e.g., auto-lint)
  - `Notification` — when Claude needs input

### Safety Rules
- **CLAUDE.md = advisory** (Claude tries to follow)
- **Hooks = deterministic** (guaranteed execution)
- Don't duplicate between them — conflicting signals
- Bash validation logic: 2,500+ lines ensuring command safety

---

## Performance Optimization Patterns

### Reduce Token Usage
1. **Use Task tool for exploration** — isolates context from main conversation
2. **Run single tests** not the full suite
3. **Pipe specific data** (`cat error.log | claude`) instead of having Claude search
4. **Use `@file` references** instead of describing file locations
5. **Keep CLAUDE.md concise** — every line costs tokens on first turn
6. **Use `/compact` proactively** before context degrades
7. **Let Claude fetch what it needs** via Bash, MCP, or file reads

### Maximize Quality
1. **Verify, verify, verify** — tests, screenshots, linters
2. **Be specific** — file names, constraints, example patterns
3. **Reference existing code patterns** for consistency
4. **Describe symptoms + likely location** for bugs
5. **Use Plan Mode** for complex multi-file changes
6. **Track tasks with TodoWrite** — prevents forgotten steps

### Speed Tips
- **Parallel tool calls** for independent operations
- **Fast mode** (`/fast`) for same model with faster output
- **Sub-agents** for embarrassingly parallel work
- **Lazy-load heavy deps** in your own tools/integrations

---

## Anti-Patterns to Avoid

### Prompting Anti-Patterns
- ❌ Vague prompts: "fix the login bug" → give symptoms and location
- ❌ Letting Claude jump to coding without understanding context
- ❌ Bloated CLAUDE.md (>200 lines) → Claude ignores instructions
- ❌ Duplicating instructions between CLAUDE.md and hooks
- ❌ Telling Claude things verbally that should be in CLAUDE.md (won't survive compaction)

### Code Anti-Patterns (from system prompt)
- ❌ Over-engineering: adding features/refactoring beyond what was asked
- ❌ Error handling for impossible scenarios
- ❌ Feature flags or backwards-compat shims when you can just change the code
- ❌ Creating helpers/utilities for one-time operations
- ❌ Designing for hypothetical future requirements
- ❌ Adding docstrings/comments to unchanged code
- ❌ Renaming unused `_vars` or adding `// removed` comments

### Architecture Anti-Patterns
- ❌ Using bash for file operations when dedicated tools exist
- ❌ Using bash echo to communicate with users
- ❌ Making dependent tool calls in parallel (use sequential)
- ❌ Using placeholders or guessing missing parameters

---

## Practical Workflow Recipes

### Recipe 1: Bug Fix
```
1. Describe symptom + paste error message
2. Point to likely file/function
3. Ask Claude to write a failing test first
4. Fix the bug
5. Verify test passes
6. Commit
```

### Recipe 2: New Feature
```
1. Enter Plan Mode — explore existing code patterns
2. Ask Claude to plan the implementation
3. Review plan (Ctrl+G to edit)
4. Switch to Normal Mode — implement step by step
5. Use TodoWrite to track all subtasks
6. Run tests after each step
7. Commit with descriptive message + open PR
```

### Recipe 3: Code Review
```
claude /review-pr <PR-number>
```

### Recipe 4: Large Refactor
```
1. Plan Mode — understand all touchpoints
2. Break into independent chunks
3. Use parallel sub-agents (Task tool) for independent changes
4. Verify each chunk with tests
5. Commit incrementally
```

### Recipe 5: Codebase Exploration
```
1. Use Task tool with subagent_type=Explore
2. Let sub-agent map the codebase structure
3. Ask specific follow-up questions with file references
```

### Recipe 6: Context Recovery
```
1. If Claude starts forgetting/making mistakes → /compact
2. CLAUDE.md will be re-read fresh
3. Conversation-only instructions will be summarized
4. Recently accessed files re-injected (5K tokens each)
```

---

## Subsystems Deep Dive

### Task System (src/tasks/)
Background/parallel work items with distinct task types:
| Type | Purpose |
|------|---------|
| **LocalShellTask** | Background shell command execution |
| **LocalAgentTask** | Sub-agent running locally |
| **RemoteAgentTask** | Agent running on remote machine |
| **InProcessTeammateTask** | Parallel teammate agent |
| **DreamTask** | Background "dreaming" / memory consolidation |

### Skill System (src/skills/)
Reusable named workflows. Key built-in skills:
- `batch` — Batch operations across files
- `debug` — Debugging workflows
- `loop` — Iterative refinement loops
- `remember` — Persist info to memory
- `verify` / `verifyContent` — Verify code correctness
- `simplify` — Simplify complex code
- `stuck` — Get unstuck when blocked
- `skillify` — Create new skills from workflows

### Bridge System (src/bridge/)
IDE integration via JWT-authenticated bidirectional channel:
- `bridgeMain.ts` — Main loop
- `bridgeMessaging.ts` — Protocol (serialize/deserialize)
- `bridgePermissionCallbacks.ts` — Routes permission prompts to IDE
- Gated behind `BRIDGE_MODE` feature flag

### Voice System (src/voice/)
- Speech-to-text streaming (`voiceStreamSTT.ts`)
- Domain-specific key terms (`voiceKeyterms.ts`)
- Gated behind `VOICE_MODE` feature flag

### Service Layer Highlights
| Service | Purpose |
|---------|---------|
| `compact/` | Conversation context compression |
| `extractMemories/` | Auto-extracted from conversations |
| `teamMemorySync/` | Shared team knowledge |
| `autoDream/` | Background ideation/memory consolidation |
| `policyLimits/` | Organization rate limits/quota |
| `tokenEstimation.ts` | Token count estimation |
| `x402/` | x402 payment protocol |

---

## Internal Codenames & Unreleased Features (FYI)

From the leaked source (informational only):
- **KAIROS** — autonomous daemon mode (always-on background agent)
- **Buddy** — digital pet system (Tamagotchi-style, April 2026)
- **Capybara** — internal codename for Claude 4.6 variant
- **Fennec** — Opus 4.6
- **Numbat** — unreleased model in testing
- **Undercover Mode** — stealth open-source contributions

### Internal Performance Benchmarks
- Capybara v8: 29-30% false claims rate (regression from v4's 16.7%)
- "Assertiveness counterweight" needed to prevent over-aggressive refactoring
- Active iteration on reducing over-commenting

---

## Quick Reference Card

| Want to... | Do this |
|-----------|---------|
| Reduce token usage | Use Task tool for exploration, `/compact` proactively |
| Ensure quality | Provide tests/verification criteria upfront |
| Persistent instructions | Put in CLAUDE.md (survives compaction) |
| Personal learnings | Auto memory (MEMORY.md) |
| Complex multi-file work | Plan Mode → sub-agents → verify → commit |
| Parallel operations | Multiple Task tool calls in single message |
| Code exploration | Task tool with `subagent_type=Explore` |
| Fast iteration | `/fast` mode, single tests not full suite |
| Fix forgotten instructions | Check CLAUDE.md length, prune, re-emphasize |
| Debug Claude behavior | Review CLAUDE.md for ambiguity/conflicts |

---

*This document is maintained by Totoro for Minjun's development workflow optimization.*
*Sources: Claude Code official docs, leaked source analysis (March 31, 2026), community analysis, VentureBeat, dev.to, Morphllm.*
