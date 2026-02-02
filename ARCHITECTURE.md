# Clawdbot Architecture Study Guide

This guide helps developers understand Clawdbot's architecture and navigate the codebase efficiently.

## High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           Messaging Channels                             │
│  WhatsApp  Telegram  Discord  Slack  Signal  iMessage  Teams  Matrix    │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                              Gateway                                     │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐    │
│  │   Routing   │  │   Pairing   │  │   Sessions  │  │    Hooks    │    │
│  └─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘    │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                              Agent Core                                  │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐    │
│  │   Agents    │  │  Providers  │  │    Tools    │  │   Memory    │    │
│  └─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘    │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                           AI Providers                                   │
│        Anthropic (Claude)    OpenAI (GPT)    Google (Gemini)            │
└─────────────────────────────────────────────────────────────────────────┘
```

## Directory Structure

| Directory | Purpose |
|-----------|---------|
| `src/` | Main source code |
| `src/cli/` | CLI framework and wiring |
| `src/commands/` | CLI command implementations |
| `src/gateway/` | Gateway server (control plane) |
| `src/agents/` | Agent logic and orchestration |
| `src/providers/` | AI provider integrations |
| `src/routing/` | Message routing logic |
| `src/channels/` | Shared channel abstractions |
| `src/sessions/` | Session management |
| `src/pairing/` | Device/channel pairing |
| `src/config/` | Configuration management |
| `src/hooks/` | Lifecycle hooks |
| `src/memory/` | Memory/context persistence |
| `src/media/` | Media processing pipeline |
| `src/telegram/` | Telegram channel |
| `src/discord/` | Discord channel |
| `src/slack/` | Slack channel |
| `src/signal/` | Signal channel |
| `src/imessage/` | iMessage channel |
| `src/web/` | WhatsApp Web channel |
| `src/whatsapp/` | WhatsApp shared utilities |
| `src/wizard/` | Onboarding wizard |
| `src/infra/` | Infrastructure utilities |
| `src/terminal/` | Terminal UI components |
| `src/tui/` | Text UI framework |
| `src/plugin-sdk/` | Plugin development SDK |
| `src/plugins/` | Plugin loading/management |
| `extensions/` | Channel and feature plugins |
| `apps/` | Native apps (macOS, iOS, Android) |
| `docs/` | Documentation source |
| `scripts/` | Build and utility scripts |

## Key Files to Study

### Entry Points

| File | Description |
|------|-------------|
| `src/entry.ts` | CLI entry point |
| `src/index.ts` | Package exports |
| `src/cli/index.ts` | CLI command registration |
| `src/gateway/index.ts` | Gateway server entry |

### Agent System

| File | Description |
|------|-------------|
| `src/agents/` | Agent orchestration |
| `src/providers/` | LLM provider adapters |
| `src/sessions/` | Conversation sessions |
| `src/memory/` | Long-term memory |

### Gateway & Routing

| File | Description |
|------|-------------|
| `src/gateway/` | Gateway server logic |
| `src/routing/` | Message routing |
| `src/pairing/` | Channel pairing |
| `src/hooks/` | Request/response hooks |

### Channels (Core)

| File | Description |
|------|-------------|
| `src/channels/` | Channel abstractions |
| `src/telegram/` | Telegram integration |
| `src/discord/` | Discord integration |
| `src/slack/` | Slack integration |
| `src/signal/` | Signal integration |
| `src/imessage/` | iMessage integration |
| `src/web/` | WhatsApp Web |

### Configuration

| File | Description |
|------|-------------|
| `src/config/` | Config loading/schema |
| `src/wizard/` | Interactive setup |

## Message Flow

```
┌──────────────┐
│ User Message │
│ (any channel)│
└──────┬───────┘
       │
       ▼
┌──────────────┐     ┌──────────────┐
│   Channel    │────▶│   Gateway    │
│   Adapter    │     │   Server     │
└──────────────┘     └──────┬───────┘
                            │
       ┌────────────────────┼────────────────────┐
       │                    │                    │
       ▼                    ▼                    ▼
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│   Routing    │     │   Session    │     │    Hooks     │
│   (allowlist │     │  (context,   │     │  (pre/post   │
│    DM check) │     │   history)   │     │   process)   │
└──────┬───────┘     └──────┬───────┘     └──────────────┘
       │                    │
       └────────┬───────────┘
                │
                ▼
       ┌──────────────┐
       │    Agent     │
       │  (reasoning, │
       │   tool use)  │
       └──────┬───────┘
              │
              ▼
       ┌──────────────┐
       │  AI Provider │
       │  (Claude,    │
       │   GPT, etc)  │
       └──────┬───────┘
              │
              ▼
       ┌──────────────┐
       │   Response   │
       │  (back thru  │
       │   channel)   │
       └──────────────┘
```

## Key Concepts

### Bindings

Bindings connect channels to the gateway. Each channel type (Telegram, Discord, etc.) has a binding that:
- Establishes connection to the messaging platform
- Translates platform-specific messages to internal format
- Handles platform authentication and webhooks

### Sessions

Sessions maintain conversation context:
- **Conversation history** - Recent messages for context
- **User identity** - Who is talking
- **Channel context** - Where the conversation is happening
- Session data persists in `~/.clawdbot/sessions/`

### Auth Profiles

Auth profiles manage AI provider credentials:
- **OAuth profiles** - Claude Pro/Max, ChatGPT subscriptions
- **API key profiles** - Direct API access
- Supports rotation and failover between profiles
- See: [Model failover docs](https://docs.clawd.bot/concepts/model-failover)

### DM Policies

Control who can DM the bot:
- **allowlist** - Only allowed users
- **denylist** - Everyone except denied users
- **open** - Anyone can DM
- Configured per-channel

### Streaming Modes

How responses are delivered:
- **full** - Wait for complete response
- **streaming** - Stream partial responses (internal UIs only)
- Note: External channels (WhatsApp, Telegram) only receive final responses

## Configuration Files

| Path | Purpose |
|------|---------|
| `~/.clawdbot/config.yaml` | Main configuration |
| `~/.clawdbot/credentials/` | Provider credentials |
| `~/.clawdbot/sessions/` | Session data |
| `~/.clawdbot/agents/` | Agent session logs |
| `~/.clawdbot/workspaces/` | Workspace configs |

## Extension System

Plugins live in `extensions/` as workspace packages:

```
extensions/
├── matrix/          # Matrix protocol
├── msteams/         # Microsoft Teams
├── zalo/            # Zalo messaging
├── voice-call/      # Voice call support
├── memory-lancedb/  # Vector memory
└── ...
```

Each extension:
- Has its own `package.json`
- Runtime deps in `dependencies`
- `clawdbot` in `devDependencies` or `peerDependencies`
- Installed via `npm install --omit=dev`

## Recommended Study Path

1. **Start with CLI**: Read `src/entry.ts` and `src/cli/` to understand command structure
2. **Understand Gateway**: Study `src/gateway/` - the control plane
3. **Learn Routing**: Follow a message through `src/routing/`
4. **Explore a Channel**: Pick one (e.g., `src/telegram/`) and trace the flow
5. **Study Agents**: See how `src/agents/` orchestrates LLM calls
6. **Check Providers**: Look at `src/providers/` for AI integrations
7. **Review Config**: Understand `src/config/` for configuration schema
8. **Explore Extensions**: Pick an extension in `extensions/` to see the plugin pattern

## Development Commands

```bash
# Install dependencies
pnpm install

# Run CLI in dev mode
pnpm clawdbot ...

# Type check and build
pnpm build

# Run tests
pnpm test

# Lint and format
pnpm lint
pnpm format
```

## Further Reading

- [Getting Started](https://docs.clawd.bot/start/getting-started)
- [Models Configuration](https://docs.clawd.bot/concepts/models)
- [Model Failover](https://docs.clawd.bot/concepts/model-failover)
- [Testing Guide](docs/testing.md)
- [AGENTS.md](AGENTS.md) - Repository guidelines for contributors
