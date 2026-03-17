# Customization Guide

## Changing Your Agent's Personality

Edit `~/.openclaw/workspace/SOUL.md`. This is the agent's core identity file — it defines personality, tone, and values.

## Adding Skills

OpenClaw has a skill marketplace. Browse and install:

```bash
# Search for skills
clawhub search "image generation"

# Install a skill
clawhub install openai-image-gen

# Configure the skill
openclaw config set skills.entries.openai-image-gen.apiKey '${OPENAI_API_KEY}'
```

## Adding More Discord Channels

1. Create the channel in Discord
2. Add it to your config:
```bash
openclaw config set channels.discord.guilds.YOUR_GUILD_ID.channels.NEW_CHANNEL_ID.allow true
```
3. Restart: `openclaw gateway restart`

## Binding an Agent to a Channel

Add a binding in `openclaw.json`:
```json
{
  "bindings": [
    {
      "agentId": "your-agent-id",
      "match": {
        "channel": "discord",
        "peer": {
          "kind": "channel",
          "id": "CHANNEL_ID"
        }
      }
    }
  ]
}
```

## Changing Models

```bash
# Set default model
openclaw config set agents.defaults.model.primary "provider/model-name"

# Set model for a specific agent
openclaw config set agents.list.1.model.primary "provider/model-name"
```

## Using a Custom LLM Provider

Clawdboss supports any OpenAI-compatible API endpoint out of the box. During `setup.sh`, choose option **c** ("Custom LLM provider") and provide:

| Field | Description | Example |
|-------|-------------|---------|
| Provider name | Short identifier | `groq`, `ollama`, `deepseek` |
| Base URL | API endpoint | `https://api.groq.com/openai/v1` |
| API key | Auth token (optional for local) | `gsk_abc123...` |
| API type | Protocol compatibility | `openai-completions` (default) |
| Model ID | Model identifier | `llama-3.3-70b-versatile` |
| Context window | Max input tokens | `128000` |
| Max output tokens | Max generation length | `16384` |

### Tested Providers

| Provider | Base URL | API Type |
|----------|----------|----------|
| Groq | `https://api.groq.com/openai/v1` | openai-completions |
| Together AI | `https://api.together.xyz/v1` | openai-completions |
| Fireworks AI | `https://api.fireworks.ai/inference/v1` | openai-completions |
| DeepSeek | `https://api.deepseek.com/v1` | openai-completions |
| Ollama (local) | `http://localhost:11434/v1` | openai-completions |
| vLLM (local) | `http://localhost:8000/v1` | openai-completions |
| LiteLLM proxy | `http://localhost:4000` | openai-completions |
| Mistral AI | `https://api.mistral.ai/v1` | openai-completions |
| Perplexity | `https://api.perplexity.ai` | openai-completions |
| xAI (Grok) | `https://api.x.ai/v1` | openai-completions |

### Manual Configuration

You can also edit `~/.openclaw/openclaw.json` directly:

```json
{
  "models": {
    "providers": {
      "my-provider": {
        "baseUrl": "https://api.example.com/v1",
        "apiKey": "${CUSTOM_LLM_API_KEY}",
        "api": "openai-completions",
        "models": [
          {
            "id": "model-id",
            "name": "Display Name",
            "input": ["text", "image"],
            "contextWindow": 128000,
            "maxTokens": 16384
          }
        ]
      }
    }
  },
  "agents": {
    "defaults": {
      "model": {
        "primary": "my-provider/model-id"
      }
    }
  }
}
```

Then set the API key in `~/.openclaw/.env`:
```dotenv
CUSTOM_LLM_API_KEY=your-key-here
```

## Memory & Context

Clawdboss uses a three-layer memory architecture:

### The Three Layers

| Layer | Location | Loaded | Purpose |
|-------|----------|--------|---------|
| **L1 (Brain)** | Root workspace files | Every turn (automatic) | Operating system — who the agent is, what's active |
| **L2 (Memory)** | `memory/` directory | Searched semantically | Long-term recall — daily notes + topic breadcrumbs |
| **L3 (Reference)** | `reference/` directory | Opened on demand | Deep context — SOPs, research, playbooks |

**Key rule:** Information flows down, never duplicated across layers. One home per fact.

### L1 Files

- **SOUL.md** — Personality, voice, values (not instructions)
- **AGENTS.md** — Role, rules, lane (not personality)
- **MEMORY.md** — What's active right now (one line per item, present tense)
- **USER.md** — How the user thinks and what they need
- **TOOLS.md** — Machine-specific commands and workarounds
- **IDENTITY.md** — Name, role, quick reference
- **HEARTBEAT.md** — Standing tasks for recurring checks
- **SESSION-STATE.md** — Active working memory (WAL target)

**L1 Budget:** Target 500-1,000 tokens per file, total under 7,000 tokens. Bloated files get skimmed — agents start missing instructions silently.

### L2: Daily Notes + Breadcrumbs

- **Daily notes** (`memory/YYYY-MM-DD.md`): Session history, decisions, completed work. What actually happened.
- **Breadcrumb files** (`memory/[topic].md`): Curated one-liners organized by topic, each pointing to deeper reference docs. Example:

```markdown
# memory/deals.md
- Active deal: 123 Main St, pending inspection → reference/deal-123-main.md
- Compliance check due March 15 → reference/compliance-sop.md
```

Breadcrumbs are the bridge — search finds the breadcrumb, the breadcrumb points to depth. Max 4KB per file.

### L3: Reference

Deep context that agents reach into on demand: SOPs, frameworks, research reports, playbooks. Not searched by `memory_search` by design — you don't want to burn context loading rarely-needed docs.

### WAL Protocol (Write-Ahead Log)

Your agents are pre-configured to use the WAL Protocol. When you tell an agent something important (a correction, a name, a decision), it writes that to `SESSION-STATE.md` before responding. This means the detail survives even if context compacts.

### Working Buffer

When context gets high (~60%), agents start logging every exchange to `memory/working-buffer.md`. After compaction, they read this buffer to recover. You never have to re-explain what you were working on.

### Maintenance Triggers

Two built-in maintenance protocols:

- **`trim`** — Weekly L1 cleanup. Measures all workspace files, moves excess to L2/L3, reports before/after token counts. Nothing gets deleted — everything is archived. Run when MEMORY.md reads like a journal or agents start missing instructions.

- **`recalibrate`** — Drift correction. Forces the agent to re-read all L1 files and compare its recent behavior against them. Reports specific drift examples and corrections. Run weekly or when the agent's personality/behavior feels off.

### Heartbeat Tasks

Add periodic checks to `HEARTBEAT.md`. Keep it lean — each heartbeat burns tokens.

## Queue Tuning

The default config uses `interrupt` mode which is best for Discord (responds to latest message, drops stale ones). If you want to process all messages in order, change to `collect`:

```bash
openclaw config set messages.queue.mode "collect"
```
