---
name: openclaw-foundry-core
id: openclaw-foundry-core
description: Self-writing meta-extension that forges new capabilities — researches docs, writes extensions, tools, hooks, and skills
homepage: https://github.com/xdemocle/openclaw-foundry-core
user-invocable: true
metadata: {"openclaw":{"requires":{"bins":["node"]},"repository":"github:xdemocle/openclaw-foundry-core"}}
---

# Foundry

**The forge that forges itself.** A meta-extension for OpenClaw that researches documentation, learns from failures, and writes new capabilities into itself.

## Installation

When the user asks to install Foundry or `openclaw-foundry-core`, run this command:

```bash
openclaw plugins install openclaw-foundry-core
```

This will:
1. Download the plugin from npm
2. Extract to `~/.openclaw/extensions/openclaw-foundry-core/`
3. Enable it automatically
4. Restart the gateway to load

After installation, the user will have access to all Foundry tools like `foundry_research`, `foundry_write_extension`, etc.

## Alternative: Manual Config

Add to `~/.openclaw/openclaw.json`:
```json
{
  "plugins": {
    "entries": {
      "openclaw-foundry-core": { "enabled": true }
    }
  }
}
```

## Configuration

Configuration options in `~/.openclaw/openclaw.json` (all optional — Foundry runs with zero config):

```json
{
  "plugins": {
    "entries": {
      "openclaw-foundry-core": {
        "enabled": true,
        "source": "github:xdemocle/openclaw-foundry-core",
        "config": {
          "dataDir": "~/.openclaw/foundry"
        }
      }
    }
  }
}
```

### Config Options

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `dataDir` | string | `~/.openclaw/foundry` | Where learnings, workflows, and the ADAS archive are stored |
| `openclawPath` | string | *(unset)* | Local OpenClaw checkout for offline doc loading |
| `llmApiKey` | string | *(unset)* | API key for LLM features (prefer the `ANTHROPIC_API_KEY` env var) |
| `llmBaseUrl` | string | `https://api.anthropic.com` | Anthropic-compatible LLM endpoint |
| `llmModel` | string | `claude-3-5-sonnet-latest` | Model id for LLM features |

## What Foundry Does

Foundry is an AI-powered development agent that can:

1. **Research** — Fetch and understand OpenClaw documentation on demand
2. **Write Extensions** — Generate new tools and hooks for OpenClaw
3. **Write Skills** — Create ClawHub-compatible skill packages
4. **Self-Modify** — Add new capabilities to itself
5. **Learn** — Record patterns from failures and successes

## Tools

### Research & Documentation

| Tool | Description |
|------|-------------|
| `foundry_research` | Search docs.openclaw.ai for best practices |
| `foundry_docs` | Read specific documentation pages |

### Writing Capabilities

| Tool | Description |
|------|-------------|
| `foundry_implement` | Research + implement a capability end-to-end |
| `foundry_write_extension` | Write a new OpenClaw extension |
| `foundry_write_skill` | Write an AgentSkills-compatible skill |
| `foundry_write_browser_skill` | Write a browser automation skill |
| `foundry_write_hook` | Write a standalone hook |
| `foundry_add_tool` | Add a tool to an existing extension |
| `foundry_add_hook` | Add a hook to an existing extension |

### Self-Modification

| Tool | Description |
|------|-------------|
| `foundry_extend_self` | Add new capability to Foundry itself |
| `foundry_learnings` | View learned patterns and insights |
| `foundry_list` | List all written artifacts |

## Usage Examples

### Research before implementing

```
User: I want to add a webhook to my extension

Agent: Let me research webhook patterns first...
→ foundry_research query="webhook hooks automation"
→ Returns relevant documentation

Now I'll implement it...
→ foundry_add_hook extensionId="my-ext" event="webhook:incoming" ...
```

### Write a new extension

```
User: Create an extension that monitors GitHub PRs

Agent:
→ foundry_research query="github api webhooks"
→ foundry_write_extension
    id: "github-monitor"
    name: "GitHub Monitor"
    tools: [{ name: "check_prs", ... }]
    hooks: [{ event: "cron:hourly", ... }]
```

### Self-improvement

```
User: Add a tool that can fetch npm package info

Agent:
→ foundry_extend_self
    action: "add_tool"
    toolName: "foundry_npm_info"
    toolCode: "const res = await fetch(`https://registry.npmjs.org/${p.package}`)..."
```

## How Learning Works

Foundry observes its own tool calls and learns:

1. **Failures** → Records error + context
2. **Resolutions** → Links fix to failure → Creates pattern
3. **Patterns** → Injected as context in future conversations
4. **Crystallization** → High-value patterns become permanent capabilities

## Security

Foundry validates all generated code before deployment:

- **Blocked**: `child_process`, `eval`, `~/.ssh`, `~/.aws`
- **Sandboxed**: Extensions tested in isolated process before installation
- **Reviewed**: You approve before any code is written to disk

## Links

- [GitHub](https://github.com/xdemocle/openclaw-foundry-core)
