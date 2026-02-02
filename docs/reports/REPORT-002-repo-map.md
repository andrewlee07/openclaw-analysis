# OpenClaw Repository Structure Map

**Analysed Commit:** See `ANALYSED_COMMIT.txt`

---

## Overview

OpenClaw is a multi-platform messaging gateway CLI with an AI agent system. It connects to various messaging channels (WhatsApp, Telegram, Discord, Slack, Signal, iMessage, etc.) and provides an AI-powered assistant using multiple LLM providers. The codebase is written primarily in TypeScript (Node.js ESM) with native apps in Swift (macOS/iOS) and Kotlin (Android).

---

## Top-Level Directory Map

### Core Components

| Directory | Purpose | Priority |
|-----------|---------|----------|
| `src/` | Main TypeScript source code (CLI, gateway, agents, channels) | **Start here** |
| `apps/` | Native applications (macOS, iOS, Android) | Platform-specific UIs |
| `extensions/` | Channel plugins and provider integrations (workspace packages) | Extensibility layer |
| `packages/` | Workspace packages (clawdbot, moltbot) | Bot-specific entry points |

### Supporting Components

| Directory | Purpose |
|-----------|---------|
| `docs/` | Mintlify documentation (hosted at docs.openclaw.ai) |
| `skills/` | Built-in skill definitions (52+ skills for various integrations) |
| `scripts/` | Build, test, release, and utility scripts |
| `ui/` | Web-based control UI (Vite + Lit components) |
| `test/` | E2E tests, fixtures, and test helpers |

### Incidental/Experimental

| Directory | Purpose |
|-----------|---------|
| `Swabble/` | Swift wake-word daemon (macOS 26+ Speech.framework) |
| `vendor/` | Vendored dependencies (a2ui specification) |
| `.pi/` | Pi agent prompts and git configuration |
| `.github/` | CI workflows, issue templates, labeler config |
| `patches/` | pnpm dependency patches (currently empty) |
| `assets/` | Static assets (DMG backgrounds, Chrome extension) |
| `git-hooks/` | Git hook scripts for development |

---

## Source Code Structure (`src/`)

### Entry Points and Orchestration

```
src/
├── entry.ts              # CLI entry point (Node.js respawn, profile handling)
├── index.ts              # Library exports and program builder
├── runtime.ts            # Runtime environment abstraction
└── globals.ts            # Global state and utilities
```

**Start reading:** `src/entry.ts` -> `src/cli/run-main.ts` -> `src/cli/program/`

### CLI Layer (`src/cli/`)

The CLI is built with Commander.js and organized into:

- `program/` - Command registration and routing
  - `build-program.ts` - Program construction
  - `register.*.ts` - Command group registrations (agent, onboard, status, etc.)
  - `command-registry.ts` - Dynamic command loading
- `*-cli.ts` - Individual CLI command implementations
- `deps.ts` - Dependency injection for testing

### Gateway Server (`src/gateway/`)

The gateway is a WebSocket/HTTP server that:
- Manages channel connections
- Routes messages to/from agents
- Provides RPC interface for mobile/desktop clients

Key files:
- `server.impl.ts` - Main server implementation
- `server-http.ts` - HTTP endpoints (OpenAI-compatible API)
- `server-chat.ts` - Chat session management
- `session-utils.ts` - Session persistence and utilities
- `protocol/` - Gateway protocol definitions

### Agent System (`src/agents/`)

The core AI agent logic:

| File | Purpose |
|------|---------|
| `pi-embedded-runner.ts` | Main agent execution loop |
| `pi-embedded-subscribe.ts` | Stream subscription and event handling |
| `pi-embedded-helpers.ts` | Utility functions for agent execution |
| `pi-tools.ts` | Tool definitions and execution |
| `bash-tools.exec.ts` | Bash command execution (sandbox-aware) |
| `system-prompt.ts` | System prompt generation |
| `model-selection.ts` | Model picking and fallback logic |
| `skills.ts` | Skill loading and snapshot building |
| `subagent-*.ts` | Multi-agent coordination |

### Commands (`src/commands/`)

High-level command implementations:

| File | Purpose |
|------|---------|
| `agent.ts` | Core `openclaw agent` command |
| `onboard-*.ts` | User onboarding flows |
| `doctor*.ts` | System health checks and migrations |
| `status*.ts` | Status display and scanning |
| `configure*.ts` | Configuration wizards |
| `health.ts` | Gateway health checks |

### Channel Implementations

| Directory | Channel |
|-----------|---------|
| `src/whatsapp/` | WhatsApp (Baileys web client) |
| `src/telegram/` | Telegram (grammy) |
| `src/discord/` | Discord (Carbon library) |
| `src/slack/` | Slack (Bolt SDK) |
| `src/signal/` | Signal (signal-cli wrapper) |
| `src/imessage/` | iMessage (BlueBubbles integration) |
| `src/web/` | Web-based WhatsApp client |
| `src/line/` | LINE messenger |

### Supporting Modules

| Directory | Purpose |
|-----------|---------|
| `src/config/` | Configuration schema, loading, validation |
| `src/infra/` | Infrastructure utilities (bonjour, ports, updates, heartbeat) |
| `src/routing/` | Message routing and session key resolution |
| `src/channels/` | Channel abstraction layer, allowlists, dock |
| `src/plugins/` | Plugin system (discovery, loading, hooks) |
| `src/hooks/` | Lifecycle hooks execution |
| `src/media/` | Media processing |
| `src/media-understanding/` | Image/file analysis |
| `src/tts/` | Text-to-speech |
| `src/tui/` | Terminal UI (pi-tui integration) |
| `src/browser/` | Browser automation (Playwright) |
| `src/sessions/` | Session management utilities |
| `src/memory/` | Memory/embedding system |
| `src/security/` | Security utilities |
| `src/terminal/` | Terminal formatting utilities |

---

## Agent Behavior Definition

Agent behavior is defined through multiple layers:

### 1. System Prompts (`src/agents/system-prompt.ts`)
- Dynamic prompt generation based on configuration
- Includes workspace context, skills, and tool descriptions

### 2. Skills (`skills/`)
Each skill is a directory with:
- `skill.md` - Skill description and instructions
- Optional `config.json` - Skill configuration

Examples: `1password/`, `github/`, `weather/`, `canvas/`, `coding-agent/`

### 3. Tools (`src/agents/pi-tools.ts`)
- File operations (read, write, edit)
- Bash execution (sandboxed)
- Messaging (send, reply)
- Browser automation
- Memory search

### 4. Agent Configuration (`src/config/types.agent-defaults.ts`)
```typescript
{
  agents: {
    defaults: {
      model: { primary: "...", fallbacks: [...] },
      thinkingDefault: "low" | "medium" | "high",
      workspaceRoot: "/path/to/workspace",
      // ...
    },
    "<agent-id>": { /* per-agent overrides */ }
  }
}
```

---

## Extensions/Plugins (`extensions/`)

Channel and provider plugins as workspace packages:

### Channel Plugins
| Plugin | Purpose |
|--------|---------|
| `telegram/` | Telegram channel |
| `discord/` | Discord channel |
| `slack/` | Slack channel |
| `signal/` | Signal channel |
| `whatsapp/` | WhatsApp channel |
| `imessage/` | iMessage channel |
| `msteams/` | Microsoft Teams |
| `matrix/` | Matrix protocol |
| `googlechat/` | Google Chat |
| `mattermost/` | Mattermost |
| `nextcloud-talk/` | Nextcloud Talk |
| `nostr/` | Nostr protocol |
| `twitch/` | Twitch chat |
| `line/` | LINE messenger |
| `zalo/`, `zalouser/` | Zalo messenger |
| `tlon/` | Tlon/Urbit |
| `voice-call/` | Voice call integration |
| `bluebubbles/` | BlueBubbles (iMessage bridge) |

### Provider Plugins
| Plugin | Purpose |
|--------|---------|
| `google-antigravity-auth/` | Google AI Studio OAuth |
| `google-gemini-cli-auth/` | Gemini CLI authentication |
| `qwen-portal-auth/` | Qwen portal OAuth |
| `minimax-portal-auth/` | Minimax portal OAuth |
| `copilot-proxy/` | GitHub Copilot proxy |

### Utility Plugins
| Plugin | Purpose |
|--------|---------|
| `llm-task/` | LLM task execution |
| `open-prose/` | Prose editing |
| `lobster/` | CLI palette/theming |
| `memory-core/`, `memory-lancedb/` | Memory backends |
| `diagnostics-otel/` | OpenTelemetry diagnostics |

---

## Native Apps (`apps/`)

### macOS (`apps/macos/`)
- Swift Package Manager project
- Menu bar app with gateway management
- `Sources/OpenClaw/` - Main app source
- `Sources/OpenClawProtocol/` - Gateway protocol models

### iOS (`apps/ios/`)
- Xcode project (generated via xcodegen)
- `Sources/` - SwiftUI app source
- `project.yml` - xcodegen specification

### Android (`apps/android/`)
- Gradle/Kotlin project
- `app/` - Main application module

### Shared (`apps/shared/`)
- `OpenClawKit/` - Shared Swift code between iOS/macOS

---

## Documentation (`docs/`)

Mintlify documentation structure:
- `docs.json` - Navigation configuration
- `channels/` - Per-channel setup guides
- `cli/` - CLI command reference
- `gateway/` - Gateway configuration
- `plugins/` - Plugin development
- `providers/` - LLM provider setup
- `platforms/` - Platform-specific guides
- `reference/` - API reference, releasing guide
- `start/` - Getting started guides
- `zh-CN/` - Chinese translations

---

## Scripts (`scripts/`)

| Script | Purpose |
|--------|---------|
| `run-node.mjs` | Development runner with tsx |
| `committer` | Git commit helper |
| `package-mac-app.sh` | macOS app packaging |
| `codesign-mac-app.sh` | Code signing |
| `notarize-mac-app.sh` | Apple notarization |
| `test-*.sh` | Various test runners |
| `protocol-gen*.ts` | Protocol schema generation |
| `update-clawtributors.ts` | README contributor list |
| `sync-plugin-versions.ts` | Plugin version synchronization |

---

## Testing

| Location | Type |
|----------|------|
| `src/**/*.test.ts` | Unit tests (colocated) |
| `src/**/*.e2e.test.ts` | E2E tests |
| `src/**/*.live.test.ts` | Live tests (require API keys) |
| `test/` | Shared fixtures and E2E helpers |
| `vitest.*.config.ts` | Test configurations |

---

## Where to Start Reading Code

### For understanding the CLI flow:
1. `src/entry.ts` - Entry point
2. `src/cli/run-main.ts` - CLI initialization
3. `src/cli/program/build-program.ts` - Command registration
4. `src/commands/agent.ts` - Core agent command

### For understanding the agent system:
1. `src/agents/pi-embedded-runner.ts` - Agent execution
2. `src/agents/system-prompt.ts` - Prompt generation
3. `src/agents/pi-tools.ts` - Tool definitions
4. `src/agents/bash-tools.exec.ts` - Command execution

### For understanding the gateway:
1. `src/gateway/server.impl.ts` - Server implementation
2. `src/gateway/server-chat.ts` - Chat handling
3. `src/gateway/session-utils.ts` - Session management

### For understanding channels:
1. `src/channels/registry.ts` - Channel registry
2. `src/channels/dock.ts` - Channel orchestration
3. Pick a specific channel: `src/whatsapp/`, `src/telegram/`, etc.

---

## Surprising/Notable Areas

1. **Swabble** (`Swabble/`) - A separate Swift package for wake-word detection using macOS 26's Speech.framework. Not part of the main build.

2. **vendor/a2ui** - Contains the A2UI (Agent-to-UI) specification, a protocol for agent-UI communication. This is vendored rather than a dependency.

3. **`.pi/prompts/`** - Contains Pi agent prompt templates for code review, changelog generation, and PR workflows.

4. **Multiple vitest configs** - Separate configurations for unit, e2e, live, extensions, and gateway tests.

5. **Dual package manager support** - Both pnpm (primary) and Bun are supported for development.

6. **Skills as directories** - Each skill in `skills/` is a self-contained directory with markdown instructions rather than code modules.

7. **Channel plugins split** - Some channels have both core implementations (`src/<channel>/`) and extension plugins (`extensions/<channel>/`), with the extension providing the plugin manifest.

---

## Unused/Experimental Areas

1. **`patches/`** - Directory exists but is currently empty (placeholder for pnpm patches)

2. **`packages/moltbot/`** - Appears to be a bot variant but minimal content

3. **Some extensions** appear experimental:
   - `nostr/` - Nostr protocol support
   - `tlon/` - Tlon/Urbit integration

---

## Build and Development

```bash
# Install dependencies
pnpm install

# Build
pnpm build

# Run development
pnpm dev <command>

# Run tests
pnpm test

# Type check + lint + format
pnpm check
```

Key files:
- `package.json` - Scripts, dependencies, build config
- `tsconfig.json` - TypeScript configuration
- `vitest.config.ts` - Test configuration
- `.oxlintrc.json` - Linter configuration
- `.oxfmtrc.jsonc` - Formatter configuration
