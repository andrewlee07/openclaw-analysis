# REPORT-001: OpenClaw System Architecture Overview

**Analysis Date:** 2026-02-02
**Analysed Commit:** `c429ccb64fc319babf4f8adc95df6d658a2d6b2f`
**Report Version:** 1.0

---

## 1. System Purpose and Problem Statement

### What is OpenClaw?

OpenClaw is a **personal AI assistant gateway** that bridges multiple messaging channels to AI agents (primarily the Pi coding agent). It provides a unified, local-first, privacy-focused interface that allows users to interact with AI through messaging applications they already use.

### Problem Statement

The system addresses several key challenges:

1. **Channel Fragmentation**: Users must leave their preferred messaging apps (WhatsApp, Telegram, Discord, etc.) to access AI assistants, creating friction and context-switching overhead.

2. **Privacy Concerns**: Cloud-hosted AI assistants require sending personal data to third-party servers. Users need a local-first solution where they control their data.

3. **Multi-Channel Complexity**: Supporting 13+ messaging platforms with different APIs, protocols, and interaction patterns requires a unified abstraction layer.

4. **Session Management**: Maintaining conversation context across multiple channels while preserving per-peer and per-group isolation.

5. **Always-On Availability**: A personal AI assistant should be constantly available without requiring manual startup each time.

### Design Philosophy

- **Local-first**: Gateway runs on user's machine with loopback-only binding by default
- **Single-user**: Designed for one person, not multi-tenant deployments
- **Privacy-focused**: No required cloud account except for LLM provider API keys
- **Always-on**: Daemon service keeps the gateway running in background
- **Pluggable**: Channels, skills, tools, and providers are extensible via plugins

---

## 2. High-Level Architectural Components

```
+------------------------------------------------------------------+
|                        OpenClaw System                            |
+------------------------------------------------------------------+
|                                                                   |
|  +------------------+     +------------------+                    |
|  |   CLI Layer      |     |   Web UI Layer   |                    |
|  | (src/cli,        |     | (ui/, Control UI)|                    |
|  |  src/commands)   |     +--------+---------+                    |
|  +--------+---------+              |                              |
|           |                        |                              |
|           v                        v                              |
|  +----------------------------------------------------+          |
|  |              Gateway Server (Control Plane)         |          |
|  |   - WebSocket Server (ws://127.0.0.1:18789)        |          |
|  |   - HTTP Server (control UI, webhooks)              |          |
|  |   - Channel Manager                                 |          |
|  |   - Session Manager                                 |          |
|  |   - Plugin Registry                                 |          |
|  |   - Config Reloader                                 |          |
|  +------------------------+---------------------------+          |
|                           |                                       |
|           +---------------+---------------+                       |
|           |               |               |                       |
|           v               v               v                       |
|  +----------------+ +----------------+ +----------------+          |
|  | Channel Plugins| | Agent Runtime  | | Plugin System  |          |
|  | - WhatsApp     | | - Pi Embedded  | | - Tools        |          |
|  | - Telegram     | | - Sessions     | | - Hooks        |          |
|  | - Discord      | | - Auth         | | - Providers    |          |
|  | - Slack        | | - Sandbox      | | - Services     |          |
|  | - Signal       | | - Memory       | +----------------+          |
|  | - iMessage     | +----------------+                             |
|  | - 7+ more      |                                               |
|  +----------------+                                               |
|                                                                   |
|  +------------------+     +------------------+                    |
|  | Config System    |     | State Stores     |                    |
|  | (~/.openclaw/    |     | - sessions.json  |                    |
|  |  config.yaml)    |     | - credentials/   |                    |
|  +------------------+     | - agents/<id>/   |                    |
|                           +------------------+                    |
+------------------------------------------------------------------+
```

### Core Components

| Component | Location | Responsibility |
|-----------|----------|----------------|
| **Gateway Server** | `src/gateway/server.impl.ts` | Central control plane, WebSocket/HTTP server, coordinates all subsystems |
| **CLI** | `src/cli/`, `src/commands/` | Command-line interface, user interaction layer |
| **Channel Manager** | `src/gateway/server-channels.ts` | Manages lifecycle of messaging channel connections |
| **Plugin Registry** | `src/plugins/registry.ts` | Loads and manages plugins (channels, tools, hooks, providers) |
| **Agent Runtime** | `src/agents/` | Pi agent execution, session management, tool invocation |
| **Routing Engine** | `src/routing/` | Routes messages to correct agent based on bindings |
| **Session Store** | `src/config/sessions/store.ts` | Persists session state and metadata |
| **Config System** | `src/config/config.ts` | YAML/JSON configuration loading and validation |

---

## 3. Control Flow Between Major Subsystems

### 3.1 Startup Flow

```
1. CLI Entry (openclaw.mjs → src/entry.ts)
   │
   ├── Parse CLI profile args
   ├── Apply profile environment
   │
   └── Load CLI Router (src/cli/run-main.ts)
       │
       └── Execute Command (e.g., "gateway run")
           │
           └── Start Gateway Server (src/gateway/server.impl.ts)
               │
               ├── Load & validate configuration
               ├── Initialize plugin registry
               ├── Create WebSocket/HTTP servers
               ├── Start channel manager
               ├── Initialize node registry (mobile)
               ├── Start maintenance timers
               ├── Start cron service
               └── Begin config reloader watcher
```

**Code Evidence:** `src/gateway/server.impl.ts:147-590` - The `startGatewayServer` function orchestrates complete startup.

### 3.2 Inbound Message Flow

```
1. User sends message on WhatsApp/Telegram/etc.
   │
2. Channel Plugin receives via protocol (Baileys/grammY/etc.)
   │
3. Channel Plugin → Gateway broadcast
   │
4. Routing Engine resolves agent + session
   │   (src/routing/resolve-route.ts)
   │   ├── Match against bindings (peer, guild, team, account, channel)
   │   └── Build session key (agent:main:channel:peer)
   │
5. Agent Runtime executes
   │   (src/agents/)
   │   ├── Load session context
   │   ├── Run Pi agent with tools
   │   └── Stream response blocks
   │
6. Response flows back through channel
   │
7. User receives reply in same channel
```

**Code Evidence:** `src/routing/resolve-route.ts:167-260` - The `resolveAgentRoute` function implements binding resolution logic.

### 3.3 Agent Event Flow

```
Agent Runtime
     │
     ├── Emits events via onAgentEvent()
     │   (src/infra/agent-events.ts)
     │
     └── createAgentEventHandler()
         (src/gateway/server-chat.ts:140-312)
         │
         ├── broadcast("agent", payload)  → WebSocket clients
         │
         ├── nodeSendToSession()          → Mobile nodes
         │
         └── emitChatDelta/emitChatFinal  → Chat UI updates
```

**Code Evidence:** `src/gateway/server-chat.ts:238-311` - The agent event handler processes lifecycle, assistant, and tool events.

### 3.4 WebSocket Communication Pattern

```
Client                    Gateway Server
  │                            │
  ├───── connect ─────────────>│
  │                            │
  │<───── health.snapshot ─────│
  │                            │
  ├───── method request ──────>│
  │      (JSON-RPC style)      │
  │                            │
  │<───── method response ─────│
  │                            │
  │<───── broadcast events ────│
  │       (agent, chat,        │
  │        health, etc.)       │
  │                            │
```

**Code Evidence:** `src/gateway/server-ws-runtime.ts` - WebSocket handlers attachment and message routing.

---

## 4. Where State Lives and How It Moves

### 4.1 State Locations

| State Type | Location | Format | Persistence |
|------------|----------|--------|-------------|
| **Configuration** | `~/.openclaw/config.yaml` | YAML | File-based, hot-reloadable |
| **Session Store** | `~/.openclaw/sessions.json` | JSON | File-based with locking |
| **Agent Sessions** | `~/.openclaw/agents/<id>/sessions/` | JSONL | Per-agent directories |
| **Credentials** | `~/.openclaw/credentials/` | Various | OAuth tokens, API keys |
| **Channel State** | In-memory (ChannelRuntimeStore) | Objects | Process lifetime |
| **Plugin Registry** | Global singleton | Objects | Process lifetime |

### 4.2 Session Store Architecture

The session store (`src/config/sessions/store.ts`) implements:

1. **File-based persistence** with JSON5 parsing
2. **Cache layer** with 45-second TTL to reduce disk I/O
3. **File locking** via `.lock` files for concurrent access
4. **Atomic writes** using temp files + rename (Unix) or direct write (Windows)

```typescript
// Session key format (src/routing/session-key.ts)
// Format: agent:<agentId>:<scope>

// Examples:
"agent:main:main"                           // Default DM session
"agent:main:telegram:dm:12345"              // Per-peer DM session
"agent:main:discord:group:server123"        // Group session
"agent:main:slack:channel:C12345:thread:ts" // Thread session
```

**Code Evidence:** `src/config/sessions/store.ts:109-175` - Session loading with cache, `src/routing/session-key.ts:126-173` - Session key construction.

### 4.3 State Flow Diagram

```
                    Configuration Layer
                    +------------------+
                    |   config.yaml    |
                    +--------+---------+
                             │
          loadConfig()       │       config reloader watches
                             v
                    +------------------+
                    |  In-Memory Config |
                    +--------+---------+
                             │
                             │
    +------------------------+------------------------+
    │                        │                        │
    v                        v                        v
+--------+            +------------+           +-------------+
|Session |            |  Plugin    |           |   Channel   |
| Store  |            |  Registry  |           |   Manager   |
+---+----+            +-----+------+           +------+------+
    │                       │                         │
    │                       │                         │
    v                       v                         v
+--------+            +------------+           +-------------+
|sessions|            | tools,     |           | runtimes,   |
|.json   |            | channels,  |           | aborts,     |
|        |            | providers  |           | tasks       |
+--------+            +------------+           +-------------+
```

### 4.4 Plugin Registry as Global State

The plugin registry (`src/plugins/runtime.ts`) uses a Symbol-based singleton pattern:

```typescript
const REGISTRY_STATE = Symbol.for("openclaw.pluginRegistryState");
// Survives hot-reloading, ensures single instance per process
```

**Code Evidence:** `src/plugins/runtime.ts:19-37` - Global singleton pattern for plugin registry.

---

## 5. Architectural Style and Core Design Bets

### 5.1 Architectural Style

OpenClaw follows a **modular monolith with plugin architecture**:

- **Monolith Core**: Gateway server, CLI, routing, session management
- **Plugin Extensions**: Channels, tools, hooks, providers loaded dynamically
- **Event-Driven**: Gateway broadcasts events to WebSocket clients
- **Request/Response via RPC**: JSON-RPC style method calls over WebSocket

### 5.2 Core Design Bets

| Design Decision | Rationale | Trade-off |
|-----------------|-----------|-----------|
| **Local-first, loopback binding** | Privacy, security by default | Requires tunneling for remote access |
| **Single-user design** | Simplified auth, no multi-tenancy complexity | Not suitable for shared servers |
| **File-based session store** | Simple, portable, no external DB | Potential scaling limits |
| **Plugin-based channels** | Easy extensibility, clean separation | Plugin loading complexity |
| **Per-command lazy loading** | Fast CLI startup | Delayed errors for invalid configs |
| **WebSocket control plane** | Real-time streaming, bidirectional | Requires persistent connection |
| **In-process Pi agent** | Low latency, no IPC overhead | Memory footprint per session |

### 5.3 Key Patterns

1. **Dependency Injection via `createDefaultDeps()`**
   - Location: `src/cli/deps.ts`
   - Provides testable, swappable dependencies

2. **Channel Docking Pattern**
   - Location: `src/channels/dock.ts`
   - Standardizes channel plugin interface

3. **Gateway Method Handlers**
   - Location: `src/gateway/server-methods.ts`
   - RPC-style request handlers with typed responses

4. **Config Reload with Hot Update**
   - Location: `src/gateway/config-reload.ts`
   - Watches config file, applies changes without restart

5. **Session Key Normalization**
   - Location: `src/routing/session-key.ts`
   - Consistent key format for session isolation

---

## 6. Text-Based Architecture Diagram

```
+============================================================================+
|                           OpenClaw Architecture                             |
+============================================================================+

EXTERNAL INTERFACES
+-------------------+   +-------------------+   +-------------------+
|   Messaging Apps  |   |   Mobile Nodes    |   |   Web Clients     |
| WhatsApp,Telegram |   | iOS / Android     |   | Control UI        |
| Discord,Slack,... |   | via WebSocket     |   | Browser           |
+--------+----------+   +--------+----------+   +--------+----------+
         |                       |                       |
         | (Protocol-specific)   | (WebSocket)           | (WebSocket/HTTP)
         v                       v                       v
+============================================================================+
|                         GATEWAY SERVER (Control Plane)                      |
|                         ws://127.0.0.1:18789                                |
|----------------------------------------------------------------------------|
|                                                                             |
|  +---------------------+    +----------------------+    +----------------+  |
|  |   CHANNEL MANAGER   |    |   WEBSOCKET SERVER   |    |   HTTP SERVER  |  |
|  |   server-channels.ts|    |   server-ws-runtime  |    |   Control UI   |  |
|  +----------+----------+    +----------+-----------+    +-------+--------+  |
|             |                          |                        |           |
|             v                          v                        v           |
|  +-------------------------------------------------------------------+      |
|  |                       MESSAGE ROUTING ENGINE                       |      |
|  |   routing/resolve-route.ts   |   routing/session-key.ts           |      |
|  +-------------------------------------------------------------------+      |
|             |                                                               |
|             v                                                               |
|  +-------------------------------------------------------------------+      |
|  |                         AGENT RUNTIME                              |      |
|  |  +---------------+  +----------------+  +----------------------+   |      |
|  |  | Pi Embedded   |  | Session Mgmt   |  | Tool Execution       |   |      |
|  |  | pi-embedded-* |  | session-*.ts   |  | bash,browser,canvas  |   |      |
|  |  +---------------+  +----------------+  +----------------------+   |      |
|  +-------------------------------------------------------------------+      |
|                                                                             |
+============================================================================+

PLUGIN SYSTEM
+---------------------+   +---------------------+   +---------------------+
|  CHANNEL PLUGINS    |   |    TOOL PLUGINS     |   |  PROVIDER PLUGINS   |
|  - whatsapp         |   |    - browser        |   |  - anthropic        |
|  - telegram         |   |    - bash           |   |  - openai           |
|  - discord          |   |    - canvas         |   |  - gemini           |
|  - slack            |   |    - memory         |   |  - bedrock          |
|  - signal           |   |    - cron           |   |  - etc.             |
|  - imessage         |   +---------------------+   +---------------------+
|  - matrix, msteams  |
|  - etc.             |
+---------------------+

STATE LAYER
+============================================================================+
|                            PERSISTENT STATE                                 |
|----------------------------------------------------------------------------|
|  +---------------------------+    +---------------------------+             |
|  |   ~/.openclaw/config.yaml |    |  ~/.openclaw/sessions.json|             |
|  |   - channels config       |    |  - session entries        |             |
|  |   - agents config         |    |  - delivery context       |             |
|  |   - routing bindings      |    |  - last channel/peer      |             |
|  +---------------------------+    +---------------------------+             |
|                                                                             |
|  +---------------------------+    +---------------------------+             |
|  | ~/.openclaw/credentials/  |    | ~/.openclaw/agents/<id>/  |             |
|  |   - OAuth tokens          |    |   - agent/ (state)        |             |
|  |   - API keys              |    |   - sessions/ (JSONL)     |             |
|  +---------------------------+    +---------------------------+             |
+============================================================================+

DATA FLOW
+--------------------------------------------------------------------------+
|  User Message → Channel Plugin → Routing → Agent → Response → Channel    |
|                                                                          |
|  [WhatsApp] ──> [Baileys] ──> [resolve-route] ──> [Pi Agent] ──>        |
|                                     │                  │                 |
|                                     v                  v                 |
|                              [session-key]     [tool execution]          |
|                                     │                  │                 |
|                                     v                  v                 |
|                              [sessions.json]   [agent events]            |
|                                                      │                   |
|                                                      v                   |
|                              [broadcast] ──> [WebSocket clients]         |
+--------------------------------------------------------------------------+
```

---

## 7. Key Source Files Reference

| Category | File | Purpose |
|----------|------|---------|
| Entry | `src/entry.ts` | CLI entry, Node respawn for warnings |
| Gateway | `src/gateway/server.impl.ts` | Main server startup and coordination |
| Channels | `src/gateway/server-channels.ts` | Channel lifecycle management |
| Routing | `src/routing/resolve-route.ts` | Agent/session routing resolution |
| Sessions | `src/routing/session-key.ts` | Session key construction |
| Store | `src/config/sessions/store.ts` | Session persistence with caching |
| Plugins | `src/plugins/registry.ts` | Plugin registration and API |
| Plugin Runtime | `src/plugins/runtime.ts` | Global plugin registry singleton |
| Events | `src/gateway/server-chat.ts` | Agent event handling and broadcasting |
| Config | `src/config/config.ts` | Configuration loading and validation |
| Agent | `src/agents/agent-scope.ts` | Agent workspace and config resolution |

---

## 8. Conclusion

OpenClaw is a well-structured modular monolith that solves the multi-channel AI gateway problem through:

1. **Clean separation** between gateway control plane and channel plugins
2. **Flexible routing** via configurable bindings that map channels/peers to agents
3. **Robust state management** with file-based persistence and in-memory caching
4. **Extensible plugin system** supporting channels, tools, hooks, and providers
5. **Event-driven architecture** for real-time streaming to multiple clients

The architecture prioritizes **privacy** (local-first), **extensibility** (plugins), and **developer experience** (TypeScript, hot-reload, comprehensive CLI tooling).

---

*Report generated from commit `c429ccb64fc319babf4f8adc95df6d658a2d6b2f`*
