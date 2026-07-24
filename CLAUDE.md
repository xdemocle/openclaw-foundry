# Foundry — The Forge That Forges Itself

Self-writing meta-extension for OpenClaw. Researches docs, learns from failures, writes new capabilities.

## Quick Reference

### Tools
```
foundry_research         — Search docs.openclaw.ai (fetches llms.txt index)
foundry_docs             — Read specific doc pages
foundry_implement        — Research + implement end-to-end
foundry_write_extension  — Write new extension
foundry_write_skill      — Write OpenClaw/AgentSkills-compatible skill
foundry_write_browser_skill — Write browser automation skill (gated on browser.enabled)
foundry_write_hook       — Write standalone hook (HOOK.md + handler.ts)
foundry_add_tool         — Add tool to extension
foundry_add_hook         — Add hook to extension
foundry_extend_self      — Add capability to Foundry itself
foundry_list             — List written artifacts
foundry_restart          — Restart gateway with resume
foundry_learnings        — View patterns/insights
foundry_meta_search      — ADAS: LLM designs + scores novel agents (needs LLM key)
foundry_self_write       — Write a tool/hook/technique to the self-written store
```

> All **25** tool objects defined in index.ts (grep `name: "foundry_`) are now registered,
> including `foundry_write_browser_skill` and `foundry_write_hook` (fixed since an earlier
> version of this doc). Registration is **not** a single batch call — the runtime rejects
> an array-returning tool factory (`"plugin must declare contracts.tools before registering
> agent tools"`), so `register(api)` (index.ts ~line 5433) builds the `toolNames` allowlist,
> materializes the tool list once, then calls `api.registerTool(tool, { name: tool.name })`
> **per tool** in a loop (index.ts:5469-5474). The non-curated registered tools are the
> Learning/Overseer surface: `foundry_overseer`, `foundry_crystallize`, `foundry_save_hook`,
> `foundry_metrics`, `foundry_evolve`, `foundry_track_outcome`, `foundry_record_feedback`,
> `foundry_get_insights`, `foundry_pending_feedback`, `foundry_apply_improvement`.

> **LLM-backed features** (`foundry_meta_search`) need an API key: set `ANTHROPIC_API_KEY`
> (or `foundry` config `llmApiKey`). Base URL/model default to Anthropic; override with
> `llmBaseUrl`/`llmModel` (any Anthropic-compatible `/v1/messages` endpoint works).

### Key Directories
```
~/.openclaw/foundry/            — Data directory
~/.openclaw/extensions/         — Generated extensions go here
~/.openclaw/skills/             — Generated skills go here
~/.openclaw/hooks/              — Generated hooks go here
~/.openclaw/hooks/foundry-resume/ — Restart resume hook
./skills/                       — Bundled skills (shipped with plugin)
```

## Development

### Type Check / Build
```bash
npx tsc --noEmit       # canonical type check — reads tsconfig.json, emits nothing
npm run typecheck      # same as the canonical check
npm run build          # `tsc -p tsconfig.json` → emits JS + d.ts + sourcemaps to dist/
npm run clean          # rm -rf dist
```

`dist/` is the build output (git-ignored) and is what `package.json` `main` (`dist/index.js`)
points to for npm consumers. The **gateway host loads `./index.ts` directly** (see the plugin
manifests), so a build is only needed for publishing — not for local gateway testing. The repo is
TypeScript-only: there are no committed `.js` artifacts.

> ⚠️ **Never run `tsc index.ts`.** Passing a filename makes `tsc` ignore `tsconfig.json`
> and fall back to default options (old `target`, no `downlevelIteration`, no `skipLibCheck`).
> You'll get a flood of bogus `downlevelIteration` / regex-flag / `bs58.default` errors that
> do NOT reflect the real type state. Always use `tsc --noEmit` or `tsc -p tsconfig.json`.

There is **no test runner and no linter** configured in this package — don't go looking for
`vitest`/`jest`/`eslint`. Validation happens at runtime via the sandbox (see below) and by
loading the extension in a live gateway.

### Test Extension Locally
```bash
openclaw gateway restart
tail -f ~/.openclaw/logs/gateway.log | grep foundry
```

## Codebase Layout

Almost all logic is in a single monolithic **`index.ts` (~6k lines)**. When navigating, jump to
these classes (line numbers drift — grep for `class <Name>`):

| Symbol | Role |
|--------|------|
| `DocsFetcher` (index.ts) | Fetches `docs.openclaw.ai/llms.txt` + pages, 30-min cache |
| `CodeWriter` (index.ts) | Generates extensions/skills/hooks from templates, writes to `~/.openclaw/...` |
| `LearningEngine` (index.ts) | Records failures/resolutions → patterns; runs the hourly Overseer |
| `CodeValidator` (index.ts) | Static security scan + isolated-process sandbox validation |
| `register(api)` (index.ts, ~line 5433) | Plugin entry: builds the `toolNames` allowlist (all 25 `foundry_*` tools — see the tool-count note above), then calls `api.registerTool(tool, {name})` per tool in a loop, plus two real hooks — `api.on("before_tool_call")` (learning capture) and `api.on("before_agent_start")` (context injection). The `command:new`/`gateway:startup` events elsewhere in these docs are for *generated* hooks, not Foundry's own. |

Helper modules in **`src/`** are **lazy-loaded at call time** via `await import("./src/<name>.js")`
(note the `.js` specifier even though sources are `.ts`):

| File | Role |
|------|------|
| `src/meta-agent-search.ts` | `MetaAgentSearch` + `ArchiveManager` — ADAS agent-design search; wired via `foundry_meta_search` |
| `src/self-writer.ts` | `SelfWriter` + code/hook/tool `TEMPLATES` — wired via `foundry_self_write` |
| `src/llm-client.ts` | `AnthropicLLMClient` (fetch-based, no SDK) + `LLMTaskEvaluator` — LLM access for ADAS; config via `llmApiKey`/`ANTHROPIC_API_KEY` |

`types/clawdbot-plugin-sdk.d.ts` provides a **fallback ambient declaration** for the optional
`clawdbot/plugin-sdk` peer dep (the real types ship with the host install; this lets Foundry
typecheck standalone). The host package (`clawdbot`/`openclaw`/`moltbot`) is an optional peer dep
and is normally **not** in `node_modules` here.

### Plugin manifests
Three near-identical manifests exist for the rebranding lineage — keep them in sync:
`openclaw.plugin.json`, `clawdbot.plugin.json`, and the `moltbot`/`openclaw`/`clawdbot` keys in
`package.json`. All point the entry at `./index.ts`.

> ⚠️ **Version drift is real and intentional-ish:** `package.json` is the npm version
> (currently `1.2.0`), while `*.plugin.json` carry their own `version` (`0.2.3`). They are
> *not* auto-synced — bump the one you mean. The publishing examples below show `0.2.0`;
> ignore that literal and use the current npm version.

Background docs live in `docs/`: `docs/ARCHITECTURE.md` (system overview) and
`docs/PROACTIVE-LEARNING.md` (the Learning/Overseer design).

## Architecture

```
User Request
     │
     ▼
Research (docs.openclaw.ai)
     │
     ▼
Generate Code (templates)
     │
     ▼
Validate (static + sandbox)
     │
     ▼
Deploy (write to extensions/)
     │
     ▼
Restart Gateway (with resume)
```

## Key Classes

### DocsFetcher
Fetches docs.openclaw.ai with 30-minute cache:
```
Available topics: plugin, hooks, tools, browser, skills, agent, gateway, channels, memory, automation
```

### CodeWriter
Generates extensions/skills/tools. Validates in sandbox before writing.

### LearningEngine
Records patterns from failures/successes. Injects context into conversations.

**Key data:**
- `~/.openclaw/foundry/learnings.json` — All failures, patterns, insights
- Patterns = failures with linked resolutions

### CodeValidator
Static security scan + isolated process sandbox testing.

## Sandbox Validation

Extensions are tested in isolated process before deployment:
1. Write to temp directory
2. Spawn Node process with tsx
3. Mock OpenClaw API
4. Try to import and run register()
5. If fails → reject, gateway stays safe
6. If passes → deploy to real extensions

## Learning Engine

### How Patterns Are Created
1. **Extension fails** → `recordFailure()` stores error with context
2. **Extension succeeds** (same ID) → `recordResolution()` links success to previous failure
3. **Pattern created** → failure + resolution = reusable pattern

### Auto-Promotion
Known error patterns are auto-promoted with standard resolutions:
- `Cannot use import statement` → Use inline code only
- `BLOCKED: Child process import` → Use HTTP APIs instead
- `BLOCKED: Shell execution` → Use direct API calls
- `Sandbox failed` → Handle null/undefined, use try/catch

### Overseer (Autonomous Actions)
Runs every hour to:
1. Auto-crystallize high-value patterns (5+ uses) into hooks
2. Prune stale unused patterns (30+ days)
3. Create insights for recurring failures (5+ occurrences)
4. Auto-promote known error patterns

### Pattern Lifecycle
```
failure (unresolved)
    ↓ success with same extension ID
pattern (has resolution)
    ↓ used 3+ times successfully
crystallization candidate
    ↓ used 5+ times
crystallized hook (permanent)
```

### Debugging
```bash
# Check learnings
cat ~/.openclaw/foundry/learnings.json | jq '.[] | select(.type=="failure")'

# Check logs
tail -f ~/.openclaw/logs/gateway.log | grep foundry
```

## Security

Blocked patterns (instant reject):
- `child_process`, `exec`, `spawn` — Shell execution
- `eval`, `new Function` — Dynamic code
- `~/.ssh`, `~/.aws` — Credential access

Flagged patterns (warning):
- `process.env` — Environment access
- `fs.readFile`, `fs.writeFile` — Filesystem access

## Integration

### Restart Resume
```typescript
// Saves context before restart
learningEngine.savePendingSession({ context, reason, lastMessage });

// foundry-resume hook injects resume message on startup
```

## Example: Write an Extension

```
1. Research what you need:
   foundry_research query="how to register tools"

2. Implement:
   foundry_write_extension({
     id: "my-tool",
     name: "My Tool",
     description: "Does something useful",
     tools: [{
       name: "do_thing",
       description: "Does the thing",
       properties: { input: { type: "string", description: "Input" } },
       required: ["input"],
       code: `return { content: [{ type: "text", text: p.input }] };`
     }],
     hooks: []
   })

3. Restart:
   foundry_restart reason="Added my-tool extension"
```

## Example: Self-Modification

```
foundry_extend_self({
  action: "add_tool",
  toolName: "foundry_my_feature",
  toolDescription: "My new feature",
  toolParameters: { ... },
  toolCode: `...`
})
```

## Config

```json
{
  "plugins": {
    "entries": {
      "foundry": {
        "enabled": true,
        "config": {
          "dataDir": "~/.openclaw/foundry",
          "autoLearn": true
        }
      }
    }
  }
}
```

## Example: Write a Skill (OpenClaw-compatible)

Skills follow the [AgentSkills](https://agentskills.io) / OpenClaw format with YAML frontmatter.

### General Skill
```typescript
foundry_write_skill({
  name: "my-skill",
  description: "Does something useful",
  content: "## How to use\n\nInstructions here...\n\nUse `{baseDir}` to reference skill folder.",
  metadata: {
    openclaw: {
      requires: { bins: ["node"], env: ["API_KEY"] },
      primaryEnv: "API_KEY"
    }
  }
})
```

### API-based Skill (Legacy)
```typescript
foundry_write_skill({
  name: "my-api",
  description: "API integration",
  baseUrl: "https://api.example.com",
  endpoints: [
    { method: "GET", path: "/users/{id}", description: "Get user by ID" },
    { method: "POST", path: "/users", description: "Create user" }
  ],
  authHeaders: { "Authorization": "Bearer ${API_KEY}" }
})
```

### Skill Frontmatter Options
```yaml
---
name: my-skill
description: What the skill does
homepage: https://example.com
user-invocable: true
disable-model-invocation: false
command-dispatch: tool
command-tool: my_tool
command-arg-mode: raw
metadata: {"openclaw":{"requires":{"bins":["node"],"env":["API_KEY"]},"primaryEnv":"API_KEY"}}
---
```

### Gating (metadata.openclaw.requires)
- `bins` — Required binaries on PATH
- `anyBins` — At least one must be on PATH
- `env` — Required environment variables
- `config` — Required config paths in openclaw.json

## Example: Write a Browser Skill

Browser skills use the OpenClaw `browser` tool for web automation.

```typescript
foundry_write_browser_skill({
  name: "twitter-poster",
  description: "Post tweets via browser automation",
  targetUrl: "https://twitter.com",
  actions: [
    {
      name: "Post Tweet",
      description: "Create and post a new tweet",
      steps: [
        "browser open https://twitter.com/compose/tweet",
        "browser snapshot",
        "browser type ref=tweet_input 'Your tweet content'",
        "browser click ref=post_button"
      ]
    }
  ],
  authMethod: "manual",
  authNotes: "Sign in to Twitter in the openclaw browser profile first"
})
```

Browser skills are automatically gated on `browser.enabled` config.

## Example: Write a Hook

Hooks trigger on OpenClaw events like `command:new`, `gateway:startup`, etc.

```typescript
foundry_write_hook({
  name: "welcome-message",
  description: "Send welcome message on new sessions",
  events: ["command:new"],
  code: `const handler: HookHandler = async (event: HookEvent) => {
  if (event.type !== 'command' || event.action !== 'new') return;
  event.messages.push('Welcome! I am ready to help.');
};`,
  metadata: { openclaw: { emoji: "👋" } }
})
```

Enable with: `openclaw hooks enable welcome-message`

### Available Hook Events
- `command:new` — New session/command started
- `command:reset` — Session reset
- `command:stop` — Session stopped
- `agent:bootstrap` — Before workspace file injection
- `gateway:startup` — After channels load
- `tool_result_persist` — Before tool result is persisted

## Learnings

- Extensions MUST go in `~/.openclaw/extensions/` for openclaw to discover them
- Each extension needs both `index.ts` and `openclaw.plugin.json`
- Tools use `parameters` (not `inputSchema`) with `execute(_toolCallId, params)`
- Extension hooks use `api.on(event, handler)` with async handlers
- Standalone hooks use `HOOK.md` + `handler.ts` pattern in `~/.openclaw/hooks/`
- Gateway restart required to load new extensions
- Skills go in `~/.openclaw/skills/` with proper SKILL.md frontmatter
- Skills use AgentSkills/OpenClaw format with YAML frontmatter (name + description required)
- Metadata must be single-line JSON per OpenClaw spec
- Sandbox validation catches runtime errors before deployment
- Browser skills require `browser.enabled` config
- Use `{baseDir}` in skill content to reference the skill folder
- Plugins can ship skills via `skills` array in openclaw.plugin.json

## Publishing Extensions

- **npm**: bump `package.json` version, `npm publish --access public`. Keep the three
  manifests (`package.json`, `openclaw.plugin.json`, `clawdbot.plugin.json`) in sync per
  the version-drift note above.
- **ClawHub**: `bun add -g clawhub` → `clawhub login` → `clawhub publish skills/<name>
  --slug <name> --name "<Name>" --version <ver> --tags latest` (needs a `SKILL.md` with
  frontmatter under `skills/<name>/`).
- **Nix flake**: optional, uses `buildNpmPackage`; only relevant if distributing via Nix.
- **Manual install**: `git clone <repo> ~/.openclaw/extensions/<name> && npm install`, or
  via `plugins.entries.<name>.source: "github:user/repo"` in config.
