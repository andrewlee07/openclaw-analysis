# REPORT-005: Configuration and Extensibility Analysis

**Analysed Commit:** `c429ccb64fc319babf4f8adc95df6d658a2d6b2f`
**Analysis Date:** 2026-02-02
**Scope:** Configuration mechanisms, extension points, modification requirements, maintainability

---

## Executive Summary

OpenClaw demonstrates a mature, deliberately designed extensibility architecture with clear extension points for plugins, tools, channels, and configuration. The system provides **designed extensibility** through a comprehensive plugin API, typed lifecycle hooks, and layered configuration. Most customization needs can be addressed without core modifications.

Key strengths:
- Manifest-based plugin discovery with JSON Schema validation
- 14+ typed lifecycle hooks covering agent, message, tool, session, and gateway events
- Hierarchical tool policy system with profiles, groups, and per-agent overrides
- Multi-layer configuration with file includes, environment substitution, and runtime overrides

Key risks:
- Plugin SDK API stability affects ecosystem health
- Complex configuration layering can produce unexpected interactions
- Channel adapter surface area creates maintenance burden

---

## 1. Configuration Mechanisms

### 1.1 Configuration File Format

**Location:** `~/.openclaw/openclaw.json` (default, overridable via `OPENCLAW_CONFIG_PATH`)

**Format:** JSON5 with extensions:
- Comments allowed (`//` and `/* */`)
- Trailing commas permitted
- Unquoted object keys

**Primary Type:** `OpenClawConfig` defined in `src/config/types.openclaw.ts`

### 1.2 Configuration Layers (Precedence Order)

| Layer | Description | Override Mechanism |
|-------|-------------|-------------------|
| 1. Defaults | Hardcoded in `src/config/defaults.ts` | Always applied first |
| 2. File Includes | `$include` directive for modular configs | Recursive merge |
| 3. Environment Variables | `config.env.vars` and `${VAR}` substitution | Before validation |
| 4. Shell Environment | Login shell env loading (`env.shellEnv.enabled`) | Fallback for missing vars |
| 5. Runtime Overrides | `setConfigOverride()` API | In-memory, highest priority |

### 1.3 Include System

```json5
{
  "$include": "./base.json5",                    // Single file
  "$include": ["./auth.json5", "./channels.json5"]  // Multiple files
}
```

**Features:**
- Relative paths resolved from config directory
- Circular dependency detection
- Max depth: 10 levels
- Deep merge semantics (arrays concatenate, objects merge)

**Source:** `src/config/includes.ts` (250 lines)

### 1.4 Environment Variable Substitution

```json5
{
  "models": {
    "providers": {
      "custom": {
        "apiKey": "${CUSTOM_API_KEY}"   // Substituted at load time
      }
    }
  },
  "env": {
    "vars": {
      "MY_VAR": "value"                 // Inline env var definition
    }
  }
}
```

**Rules:**
- Pattern: `[A-Z_][A-Z0-9_]*` only (uppercase)
- Escape: `$${VAR}` produces literal `${VAR}`
- Missing vars throw `MissingEnvVarError` with JSON path context

**Source:** `src/config/env-substitution.ts` (135 lines)

### 1.5 CLI Configuration Commands

| Command | Purpose |
|---------|---------|
| `openclaw config` | Interactive wizard |
| `openclaw config get <path>` | Read value |
| `openclaw config set <path> <value>` | Write value |
| `openclaw config unset <path>` | Remove value |

**Path Notation:** Dot notation (`gateway.port`), bracket notation (`agents.list[0]`), mixed

**Source:** `src/cli/config-cli.ts` (345 lines)

### 1.6 Configuration Backup System

- 5 rotating backups on write
- Atomic write via temp file + rename
- File permissions: `0o600` (owner read/write only)
- Pattern: `openclaw.json.bak`, `openclaw.json.bak.1`, etc.

---

## 2. Extension Points (Designed Extensibility)

### 2.1 Plugin System Architecture

**Discovery Locations (precedence order):**
1. Workspace `node_modules/`
2. Global npm `node_modules/`
3. Config-specified `plugins.loadPaths`
4. Bundled plugins in `extensions/`

**Manifest File:** `openclaw.plugin.json` (required)

```json
{
  "id": "twitch",
  "channels": ["twitch"],
  "configSchema": {
    "type": "object",
    "additionalProperties": false,
    "properties": {}
  }
}
```

**Source:** `src/plugins/loader.ts` (452 lines), `src/plugins/discovery.ts`

### 2.2 Plugin API Surface

The `OpenClawPluginApi` provides these registration methods:

| Method | Purpose | Example Use Case |
|--------|---------|-----------------|
| `registerTool()` | Agent tools | Custom MCP integrations |
| `registerHook()` | Lifecycle hooks | Analytics, logging |
| `registerChannel()` | Messaging channels | Twitch, Matrix, MS Teams |
| `registerProvider()` | Model providers | Custom LLM backends |
| `registerGatewayMethod()` | Gateway RPC | Custom protocol extensions |
| `registerHttpHandler()` | HTTP middleware | Auth, metrics |
| `registerHttpRoute()` | HTTP endpoints | Webhooks, callbacks |
| `registerCli()` | CLI commands | Plugin-specific commands |
| `registerService()` | Background services | Monitors, pollers |
| `registerCommand()` | Chat commands | `/tts`, `/status` |
| `on()` | Typed lifecycle hooks | Programmatic hook binding |

**Source:** `src/plugins/types.ts` (527 lines)

### 2.3 Typed Lifecycle Hooks

OpenClaw provides 14 typed lifecycle hooks with priority ordering:

**Agent Lifecycle:**
- `before_agent_start` - Inject context into system prompt (sequential, returns result)
- `agent_end` - Analyze completed conversations (parallel, fire-and-forget)

**Message Lifecycle:**
- `message_received` - Process incoming messages (parallel)
- `message_sending` - Modify/cancel outgoing messages (sequential, returns result)
- `message_sent` - Post-send analytics (parallel)

**Tool Lifecycle:**
- `before_tool_call` - Modify params or block calls (sequential, returns result)
- `after_tool_call` - Tool telemetry (parallel)
- `tool_result_persist` - Modify session transcript (synchronous, sequential)

**Session Lifecycle:**
- `session_start` - Session initialization (parallel)
- `session_end` - Session cleanup (parallel)

**Compaction Lifecycle:**
- `before_compaction` - Pre-compaction notification (parallel)
- `after_compaction` - Post-compaction notification (parallel)

**Gateway Lifecycle:**
- `gateway_start` - Gateway initialization (parallel)
- `gateway_stop` - Gateway shutdown (parallel)

**Hook Registration:**
```typescript
api.on("before_tool_call", (event, ctx) => {
  if (event.toolName === "exec") {
    return { block: true, blockReason: "Exec disabled" };
  }
}, { priority: 100 });
```

**Source:** `src/plugins/hooks.ts` (471 lines)

### 2.4 Tool Policy System

**Profiles (built-in presets):**
- `minimal` - Only `session_status`
- `coding` - File I/O, runtime, sessions, memory, image
- `messaging` - Messaging surface, session tools, status
- `full` - All tools, no restrictions

**Tool Groups (expandable via `group:` prefix):**
- `group:memory` - memory_search, memory_get
- `group:web` - web_search, web_fetch
- `group:fs` - read, write, edit, apply_patch
- `group:runtime` - exec, process
- `group:sessions` - session management tools
- `group:ui` - browser, canvas
- `group:automation` - cron, gateway
- `group:messaging` - message
- `group:nodes` - nodes tool
- `group:openclaw` - all native OpenClaw tools
- `group:plugins` - all plugin-provided tools

**Policy Hierarchy:**
1. Profile policy (`tools.profile`)
2. Provider-specific profile (`tools.byProvider[provider].profile`)
3. Global allow/deny
4. Provider allow/deny
5. Agent allow/deny
6. Agent provider allow/deny
7. Group (channel) allow/deny
8. Sandbox allow/deny
9. Subagent default deny

**Configuration Example:**
```json5
{
  "tools": {
    "profile": "coding",
    "allow": ["web_search"],
    "deny": ["browser"]
  },
  "agents": {
    "list": [{
      "id": "restricted",
      "tools": {
        "deny": ["exec", "process"]
      }
    }]
  }
}
```

**Source:** `src/agents/pi-tools.policy.ts`, `src/agents/tool-policy.ts`

### 2.5 Channel Plugin Architecture

Channels implement the `ChannelPlugin` interface with adapter-based composition:

| Adapter | Responsibility |
|---------|---------------|
| `config` | Configuration parsing/resolution |
| `configSchema` | JSON Schema for channel config |
| `setup` | Initial setup flow |
| `pairing` | QR/code pairing flows |
| `security` | DM policy, account validation |
| `groups` | Group/conversation management |
| `mentions` | @mention parsing |
| `outbound` | Message sending |
| `status` | Health/connection status |
| `gateway` | Gateway protocol handlers |
| `auth` | Authentication adapters |
| `elevated` | Admin/elevated permissions |
| `commands` | Channel-specific commands |
| `streaming` | Streaming message support |
| `threading` | Thread/reply handling |
| `messaging` | Message parsing/formatting |
| `agentPrompt` | Agent context hints |
| `directory` | Contact/group directory |
| `resolver` | Target resolution |
| `actions` | Message actions (react, delete) |
| `heartbeat` | Connection heartbeat |
| `agentTools` | Channel-specific agent tools |
| `onboarding` | CLI onboarding wizard |

**Example (Twitch channel):**
```typescript
// extensions/twitch/index.ts
export default {
  id: "twitch",
  name: "Twitch",
  register(api: OpenClawPluginApi) {
    api.registerChannel({ plugin: twitchPlugin });
  },
};
```

**Source:** `src/channels/plugins/types.plugin.ts` (85 lines)

### 2.6 Skills System

Skills are markdown-based prompt extensions with metadata:

**Skill Loading Sources (precedence order):**
1. Extra directories (`skills.load.extraDirs`)
2. Bundled skills
3. Managed skills (`~/.openclaw/skills/`)
4. Workspace skills (`<workspace>/skills/`)

**Skill Metadata (frontmatter):**
```yaml
---
always: true
skillKey: git
primaryEnv: GIT_PATH
os: [darwin, linux]
requires:
  bins: [git]
  config: [gateway.enabled]
install:
  - kind: brew
    formula: git
---
```

**Source:** `src/agents/skills/types.ts`, `src/agents/skills/workspace.ts`

### 2.7 Provider Plugin System

Custom model providers can register via plugins:

```typescript
api.registerProvider({
  id: "custom-llm",
  label: "Custom LLM",
  aliases: ["custom"],
  envVars: ["CUSTOM_API_KEY"],
  models: {
    default: "custom-model-v1",
    aliases: { fast: "custom-model-fast" }
  },
  auth: [{
    id: "api_key",
    label: "API Key",
    kind: "api_key",
    run: async (ctx) => { /* ... */ }
  }]
});
```

**Source:** `src/plugins/providers.ts`, `src/plugins/types.ts` (lines 84-124)

---

## 3. Areas Requiring Core Code Modification

### 3.1 Definite Core Modifications Required

| Feature | Location | Reason |
|---------|----------|--------|
| New hook types | `src/plugins/types.ts`, `src/plugins/hooks.ts` | TypeScript types + runner methods |
| New tool groups | `src/agents/tool-policy.ts` | Hardcoded group definitions |
| Core tool changes | `src/agents/pi-tools.ts` | Tool factories in core |
| Agent loop behavior | `src/agents/pi-embedded-runner.ts` | Core agent execution logic |
| New config sections | `src/config/zod-schema.ts`, `types.*.ts` | Schema validation |
| Gateway protocol | `src/gateway/server-methods/` | Core RPC handling |
| Compaction algorithms | `src/agents/compaction.ts` | Session management core |

### 3.2 Potentially Extensible Without Core Changes

| Feature | Extension Mechanism |
|---------|-------------------|
| Custom tools | Plugin `registerTool()` |
| Tool behavior modification | `before_tool_call` / `after_tool_call` hooks |
| System prompt injection | `before_agent_start` hook |
| Message interception | `message_sending` / `message_received` hooks |
| New messaging channels | Plugin `registerChannel()` |
| New model providers | Plugin `registerProvider()` |
| Custom CLI commands | Plugin `registerCli()` |
| HTTP endpoints | Plugin `registerHttpRoute()` |
| Background services | Plugin `registerService()` |
| Chat commands | Plugin `registerCommand()` |

### 3.3 Grey Areas (Partial Extension Support)

| Feature | Current State | Limitation |
|---------|--------------|------------|
| Tool result transformation | `tool_result_persist` hook exists | Synchronous only, limited context |
| Session persistence | No hooks | Core code modification required |
| Model routing | Config-based only | No runtime plugin hooks |
| Sandbox policy | Config + defaults | Cannot add new sandbox modes |

---

## 4. Long-Term Maintainability and Scaling Risks

### 4.1 API Stability Risks

**Risk: Plugin SDK Breaking Changes**
- **Impact:** All plugins must update for incompatible changes
- **Current Mitigation:** TypeScript types provide compile-time checks
- **Gap:** No versioned SDK, no deprecation warnings
- **Recommendation:** Introduce SDK versioning with compatibility matrix

**Risk: Hook Contract Changes**
- **Impact:** Priority ordering changes can break plugins
- **Current Mitigation:** Priority-based ordering is deterministic
- **Gap:** No documentation of expected hook execution order
- **Recommendation:** Document and test hook ordering guarantees

### 4.2 Configuration Complexity Risks

**Risk: Layer Interaction Surprises**
- 5 configuration layers can interact unexpectedly
- Environment variable substitution happens before validation
- Runtime overrides bypass file-based config
- **Recommendation:** Add `openclaw config debug` to show resolved config sources

**Risk: Schema Migration Burden**
- Zod schema at 563 lines, growing
- Legacy migration code spans 3 files
- **Recommendation:** Automated schema migration generators

### 4.3 Channel Adapter Surface Area

**Risk: Large Interface Contract**
- `ChannelPlugin` has 24+ optional adapters
- Each adapter has its own contract and types
- **Recommendation:** Consider adapter composition patterns, reduce mandatory surface

**Risk: Channel-Specific Complexity**
- Channels like WhatsApp, Signal have platform-specific behaviors
- Testing matrix grows exponentially with channels
- **Recommendation:** Abstract common patterns, platform adapters for specifics

### 4.4 Scaling Considerations

**Horizontal Scaling Limitations:**
- Single gateway process model
- Session state stored locally
- Plugin registry is process-global

**Plugin Ecosystem Growth:**
- No plugin marketplace/registry
- Installation via npm only
- No plugin dependency resolution

---

## 5. Extensibility Classification Matrix

| Extension Type | Designed | Accidental | Requires Core |
|---------------|----------|------------|---------------|
| Custom tools | Yes | - | - |
| Tool blocking/modification | Yes | - | - |
| System prompt injection | Yes | - | - |
| Message interception | Yes | - | - |
| New channels | Yes | - | - |
| New providers | Yes | - | - |
| CLI commands | Yes | - | - |
| HTTP endpoints | Yes | - | - |
| Background services | Yes | - | - |
| Chat commands | Yes | - | - |
| New hook types | - | - | Yes |
| Tool groups | - | - | Yes |
| Agent loop changes | - | - | Yes |
| Config schema | - | - | Yes |
| Gateway protocol | - | - | Yes |
| jiti TypeScript loading | - | Yes | - |
| Global registry access | - | Yes | - |

---

## 6. Lessons for Jarvis v2

### 6.1 Architecture Patterns to Adopt

1. **Manifest-Based Plugin Discovery**
   - JSON Schema validation before runtime execution
   - Clear metadata: id, version, capabilities, config schema
   - Enable/disable via config without code changes

2. **Typed Lifecycle Hooks**
   - Discriminate void vs result-returning hooks
   - Priority ordering for deterministic execution
   - Sequential vs parallel execution modes

3. **Hierarchical Policy System**
   - Profiles for common presets
   - Groups for logical tool collections
   - Per-context overrides (agent, provider, sandbox)

4. **Multi-Layer Configuration**
   - File includes for modularity
   - Environment variable substitution
   - Runtime overrides for programmatic control

### 6.2 Patterns to Improve

1. **SDK Versioning**
   - Declare minimum SDK version in manifests
   - Deprecation warnings for breaking changes
   - Compatibility matrix documentation

2. **Extension Discovery**
   - Plugin marketplace/registry for discoverability
   - Dependency resolution between plugins
   - Version conflict detection

3. **Configuration Debugging**
   - Debug command showing resolved config with source attribution
   - Validation warnings for common mistakes
   - Schema diff tools for migration

4. **Testing Infrastructure**
   - Plugin test harness with mock API
   - Channel adapter test fixtures
   - Hook execution order tests

### 6.3 Anti-Patterns to Avoid

1. **Hardcoded Expansion Lists**
   - Tool groups defined in code, not config
   - Requires core changes to add groups

2. **Large Interface Contracts**
   - 24+ adapter methods in ChannelPlugin
   - Prefer composition over inheritance

3. **Synchronous Hook Limitations**
   - `tool_result_persist` is synchronous only
   - Plan async-first for hot path hooks

4. **Global Singleton State**
   - Plugin registry via `Symbol.for` global
   - Complicates testing and multi-instance

### 6.4 Recommended Extension Points for Jarvis v2

Based on OpenClaw's proven extension points:

| Extension Point | Priority | Notes |
|-----------------|----------|-------|
| Tool registration | Must have | Core extensibility requirement |
| Lifecycle hooks (typed) | Must have | before_agent_start, before_tool_call essential |
| Channel plugins | Must have | Multi-platform messaging support |
| Provider plugins | Must have | LLM abstraction layer |
| Config includes | Should have | Modular configuration |
| CLI extensions | Should have | Plugin-specific commands |
| HTTP endpoints | Should have | Webhooks, callbacks |
| Background services | Nice to have | Monitors, scheduled tasks |
| Chat commands | Nice to have | User-facing shortcuts |

### 6.5 Recommended Hook Set for Jarvis v2

| Hook | Type | Purpose |
|------|------|---------|
| `agent:before_start` | Modifying | Inject context |
| `agent:after_end` | Void | Analytics |
| `tool:before_call` | Modifying | Block/modify calls |
| `tool:after_call` | Void | Telemetry |
| `message:before_send` | Modifying | Transform/cancel |
| `message:after_receive` | Void | Logging |
| `session:start` | Void | Initialization |
| `session:end` | Void | Cleanup |
| `config:changed` | Void | React to config updates |

---

## 7. Conclusion

OpenClaw provides a well-architected extensibility system with clear separation between designed extension points and core functionality. The plugin API, hook system, and configuration layers enable substantial customization without forking.

**Key Takeaways:**
1. Most practical extensions are achievable through the plugin system
2. The hook system is comprehensive but could benefit from better documentation
3. Configuration complexity is the primary maintainability concern
4. Channel adapter surface area should be reduced for long-term health

**Recommendation for Jarvis v2:**
Adopt OpenClaw's manifest-based plugin discovery, typed hook system, and hierarchical policy model while addressing the versioning, debugging, and interface size concerns identified in this analysis.

---

*Reference: ANALYSED_COMMIT.txt (`c429ccb64fc319babf4f8adc95df6d658a2d6b2f`)*
