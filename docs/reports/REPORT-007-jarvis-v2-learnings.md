# REPORT-007: Jarvis v2 Comparative Analysis

**Analysis Target:** OpenClaw codebase
**Analysed Commit:** `c429ccb64fc319babf4f8adc95df6d658a2d6b2f`
**Report Date:** 2026-02-02
**Purpose:** Translate OpenClaw investigation findings into actionable design guidance for Jarvis v2

---

## Executive Summary

This report distills architectural patterns from the OpenClaw codebase into concrete recommendations for Jarvis v2. The analysis identifies patterns to adopt, avoid, and evolve beyond, with specific implications for agent model design, tool governance, and state handling.

---

## 1. Patterns Jarvis v2 Should Adopt

### 1.1 Double-Queue Lane System for Concurrency Control

**Source:** `src/agents/pi-embedded-runner/lanes.ts`, `src/agents/pi-embedded-runner/run.ts:74-91`

OpenClaw implements a two-tier queuing system:
- **Session-level queue:** Prevents concurrent operations on the same session
- **Global queue:** Prevents system-wide resource contention

```
Session Lane: session:chat-123 → Session Queue
                                      ↓
Global Lane: main               → Global Queue → Execute
```

**Jarvis v2 Recommendation:**
- Adopt hierarchical queuing with session-scope and global-scope lanes
- Session isolation prevents transcript corruption from concurrent writes
- Global coordination prevents provider rate-limit avalanche effects
- This pattern enables safe multi-agent operation without explicit distributed locking

### 1.2 Exponential Backoff with Failure Classification

**Source:** `src/agents/auth-profiles/usage.ts:78-84`, `src/agents/failover-error.ts`

OpenClaw classifies failures by type and applies differentiated backoff:

| Failure Type | Backoff Sequence | Max Duration |
|-------------|------------------|--------------|
| Rate limit | 1m → 5m → 25m → 1h | 1 hour |
| Billing | 5h base, 2x exponential | 24 hours |
| Auth | Standard cooldown | 1 hour |
| Timeout | Treated as rate limit | 1 hour |

**Jarvis v2 Recommendation:**
- Implement typed failure classification (not just generic error handling)
- Billing errors warrant longer backoff than transient failures
- Track per-profile failure counts within a sliding window (default: 24h)
- Reset failure state on successful operation, not on timeout expiry

### 1.3 Auth Profile Rotation with Proactive Health Tracking

**Source:** `src/agents/auth-profiles/usage.ts`, `src/agents/pi-embedded-runner/run.ts:249-274`

OpenClaw maintains multiple auth profiles per provider with usage statistics:
- `lastUsed`: Timestamp of last successful use
- `errorCount`: Consecutive failures within window
- `cooldownUntil`: When profile becomes available
- `failureCounts`: Per-reason failure breakdown

**Jarvis v2 Recommendation:**
- Support multiple credentials per model provider
- Track credential health independently (not just availability)
- Implement `advanceAuthProfile()` pattern: attempt rotation before giving up
- Mark profiles "good" on success to enable rapid recovery

### 1.4 Tool Policy Framework with Group Abstraction

**Source:** `src/agents/tool-policy.ts`

OpenClaw defines tool access through composable groups:

```typescript
TOOL_GROUPS = {
  "group:fs": ["read", "write", "edit", "apply_patch"],
  "group:runtime": ["exec", "process"],
  "group:sessions": ["sessions_list", "sessions_history", ...],
  // ...
}

TOOL_PROFILES = {
  minimal: { allow: ["session_status"] },
  coding: { allow: ["group:fs", "group:runtime", "group:sessions", ...] },
  messaging: { allow: ["group:messaging", "sessions_list", ...] },
  full: {},  // Empty = all allowed
}
```

**Jarvis v2 Recommendation:**
- Define tools as first-class entities with group membership
- Policies reference groups, not individual tools (reduces config churn)
- Support `allow` and `deny` lists with group expansion
- Profile presets (`minimal`, `coding`, `full`) simplify common configurations

### 1.5 Session Store with Write-Lock and Atomic Operations

**Source:** `src/config/sessions/store.ts:285-355`

OpenClaw implements robust session persistence:
- File-based locking via `.lock` file with exclusive create (`wx` mode)
- Stale lock eviction (30s default) for crash recovery
- Atomic writes via temp file + rename (Unix) or direct write with lock (Windows)
- Cache invalidation on write to prevent stale reads

**Jarvis v2 Recommendation:**
- Implement pessimistic locking for session writes (not optimistic/CAS)
- Stale lock detection prevents indefinite blocking from crashed processes
- Always re-read inside lock to avoid overwriting concurrent changes
- Use `structuredClone()` for cache entries to prevent mutation leakage

### 1.6 Context Window Guard with Hard Limits

**Source:** `src/agents/context-window-guard.ts`, `src/agents/pi-embedded-runner/run.ts:113-138`

OpenClaw validates context window before execution:
- Hard minimum (1024 tokens): Block model if below
- Warning threshold: Log but proceed
- Per-model context window configuration

**Jarvis v2 Recommendation:**
- Pre-flight context validation prevents wasted API calls
- Fail fast with clear error messages, not mid-execution
- Allow per-model context overrides for fine-tuning
- Auto-compaction on overflow (see section 1.7)

### 1.7 Automatic Session Compaction on Overflow

**Source:** `src/agents/pi-embedded-runner/run.ts:372-409`, `src/agents/pi-embedded-runner/compact.ts`

When context overflow occurs, OpenClaw:
1. Detects overflow error via `isContextOverflowError()`
2. Attempts compaction (removes older turns while preserving recent context)
3. Retries original prompt if compaction succeeds
4. Falls back to user-friendly error if compaction fails

**Jarvis v2 Recommendation:**
- Implement transparent compaction as first-response to overflow
- Track `overflowCompactionAttempted` to prevent infinite loops
- Distinguish `context_overflow` from `compaction_failure` in error responses
- Compaction should preserve tool call/result pairs atomically

---

## 2. Patterns Jarvis v2 Should Avoid

### 2.1 In-Memory Cache Without Invalidation Hooks

**Source:** `src/config/sessions/store.ts:27-53`

OpenClaw uses TTL-based cache (45s default) with mtime checking. While functional, this pattern has issues:
- Race window between mtime check and cache use
- No invalidation on external file modification (e.g., manual edit)
- Cache TTL is a tuning parameter that varies by deployment

**Jarvis v2 Anti-Pattern:**
- Avoid relying solely on TTL for cache coherence
- Do not assume single-writer access to session files
- Avoid cache layers that lack explicit invalidation primitives

### 2.2 Global Registry Singletons for Plugin State

**Source:** `src/plugins/registry.ts`

OpenClaw accumulates plugin registrations into a global mutable registry:

```typescript
const registry: PluginRegistry = {
  plugins: [],
  tools: [],
  hooks: [],
  channels: [],
  // ... accumulated during startup
}
```

**Jarvis v2 Anti-Pattern:**
- Global mutable state complicates testing and hot-reload
- Plugin load order becomes implicit dependency
- Avoid accumulator patterns; prefer immutable snapshots or dependency injection
- Plugin state should be scoped to execution context, not process lifetime

### 2.3 Mixed Sync/Async File Operations

**Source:** `src/infra/exec-approvals.ts:189-219`

OpenClaw mixes `fs.existsSync()` with `fs.readFileSync()` and async patterns:

```typescript
function readExecApprovalsSnapshot(): ExecApprovalsSnapshot {
  if (!fs.existsSync(filePath)) { ... }  // Sync check
  const raw = fs.readFileSync(filePath, "utf8");  // Sync read
  // ...
}
```

**Jarvis v2 Anti-Pattern:**
- Sync file operations block event loop
- TOCTOU race between `existsSync` and `readFile`
- Prefer fully async I/O with proper error handling for ENOENT

### 2.4 String-Based Error Classification via Regex

**Source:** `src/agents/pi-embedded-helpers.ts`, `src/agents/failover-error.ts:115-124`

OpenClaw classifies errors by matching message strings:

```typescript
const TIMEOUT_HINT_RE = /timeout|timed out|deadline exceeded/i;
const ABORT_TIMEOUT_RE = /request was aborted|request aborted/i;
```

**Jarvis v2 Anti-Pattern:**
- Fragile to provider message changes
- Locale/language sensitive
- Prefer structured error codes when available
- If regex matching is unavoidable, centralize patterns with version tracking

### 2.5 Magic String Scrubbing as Security Measure

**Source:** `src/agents/pi-embedded-runner/run.ts:57-69`

OpenClaw scrubs specific strings to prevent model behavior manipulation:

```typescript
const ANTHROPIC_MAGIC_STRING_TRIGGER_REFUSAL = "ANTHROPIC_MAGIC_STRING_TRIGGER_REFUSAL";
// ...
function scrubAnthropicRefusalMagic(prompt: string): string {
  return prompt.replaceAll(MAGIC_STRING, REPLACEMENT);
}
```

**Jarvis v2 Anti-Pattern:**
- Blocklist-based scrubbing is inherently incomplete
- Security through string matching is easily bypassed
- Prefer allowlist validation and content policy layers
- Do not rely on input sanitization for security-critical decisions

---

## 3. Patterns Jarvis v2 Should Evolve Beyond

### 3.1 From File-Based Sessions to Structured Event Log

**Current Pattern:** OpenClaw stores sessions as JSON files with transcript arrays
**Source:** `src/config/sessions/store.ts`, `src/config/sessions/types.ts`

**Evolution for Jarvis v2:**
- Replace monolithic session files with append-only event logs
- Events: `message_received`, `tool_invoked`, `tool_completed`, `response_sent`
- Benefits:
  - Crash recovery via event replay
  - Partial session inspection without loading entire transcript
  - Natural audit trail for compliance
  - Enables streaming persistence (write events, not snapshots)

### 3.2 From Exec Approvals to Capability-Based Authorization

**Current Pattern:** OpenClaw uses path-based allowlists with glob matching
**Source:** `src/infra/exec-approvals.ts:550-572`

```typescript
function matchAllowlist(entries, resolution): ExecAllowlistEntry | null {
  for (const entry of entries) {
    if (matchesPattern(entry.pattern, resolution.resolvedPath)) {
      return entry;
    }
  }
  return null;
}
```

**Evolution for Jarvis v2:**
- Move from "what executable" to "what capability" model
- Define capabilities: `fs.read`, `fs.write`, `net.http`, `exec.shell`
- Tools declare required capabilities; authorization checks capabilities
- Benefits:
  - Portable across platforms (no path dependencies)
  - Composable capability bundles
  - Clear security audit (what can agent X do?)

### 3.3 From Plugin Registry to Hot-Loadable Modules

**Current Pattern:** Plugins register at startup; changes require restart
**Source:** `src/plugins/registry.ts`, `src/plugins/loader.ts`

**Evolution for Jarvis v2:**
- Support hot-loading of tool modules without full restart
- Version-pinned module references for reproducibility
- Sandboxed module execution (separate V8 isolate or subprocess)
- Benefits:
  - Development iteration without restart
  - Runtime capability augmentation
  - Fault isolation (crashed module doesn't crash agent)

### 3.4 From Embedded Compaction to External Memory Service

**Current Pattern:** Compaction runs inline during agent execution
**Source:** `src/agents/pi-embedded-runner/compact.ts`

**Evolution for Jarvis v2:**
- Extract memory management to dedicated service
- Service handles: compaction, summarization, retrieval
- Agent communicates with memory service via well-defined API
- Benefits:
  - Compaction doesn't block agent execution
  - Memory strategies can evolve independently
  - Enables shared memory across agent instances

### 3.5 From Hierarchical Session Keys to Content-Addressable References

**Current Pattern:** Session keys encode routing hierarchy
**Source:** `src/sessions/session-key-utils.ts`

```
agent:main:direct-chat
agent:main:subagent:child:task-123
agent:main:telegram:user-456:thread:789
```

**Evolution for Jarvis v2:**
- Content-addressable session references (hash of initial context)
- Metadata (agent, channel, thread) stored separately
- Session key becomes opaque identifier
- Benefits:
  - Decouples session identity from routing
  - Enables session migration across agents/channels
  - Simplifies key parsing (no format assumptions)

---

## 4. Implications for Agent Model

### 4.1 Execution Model

| Aspect | OpenClaw Pattern | Jarvis v2 Recommendation |
|--------|------------------|--------------------------|
| Concurrency | Double-queue lanes | Adopt: Session + Global queues |
| Retry | In-loop with profile rotation | Adopt: Rotation before failure |
| Context | Guard + auto-compact | Evolve: External memory service |
| Isolation | Subagent spawning | Adopt: First-class subagent model |

### 4.2 Session Lifecycle

OpenClaw session lifecycle:
1. Resolve session key from routing context
2. Load session (with cache, lock on write)
3. Run agent turn (queued)
4. Persist updated transcript
5. Update metadata (lastChannel, lastTo, etc.)

**Jarvis v2 Recommendation:**
- Adopt lifecycle but replace step 4 with event append
- Add explicit session states: `active`, `compacting`, `archived`
- Support session branching for "what-if" exploration

### 4.3 Multi-Agent Coordination

OpenClaw supports multi-agent via:
- Session key encoding (`agent:parentId:subagent:childId:key`)
- Spawned agent tools (`sessions_spawn`)
- Lane isolation per agent

**Jarvis v2 Recommendation:**
- Formalize parent-child agent relationships
- Define message passing protocol between agents
- Support agent supervision (parent monitors child health)
- Consider actor model semantics for agent communication

---

## 5. Implications for Tool Governance

### 5.1 Authorization Model

| Layer | OpenClaw Implementation | Jarvis v2 Recommendation |
|-------|------------------------|--------------------------|
| Global | Tool allowlist/denylist | Adopt with capability mapping |
| Agent | Per-agent policy override | Adopt |
| Session | Tool profile binding | Evolve to dynamic capability |
| Tool | Approval queue for exec | Adopt for sensitive operations |

### 5.2 Tool Registration

OpenClaw pattern:
```typescript
registerTool(factory, { name, optional })
```

**Jarvis v2 Recommendation:**
- Add capability declarations: `registerTool(factory, { capabilities: ["fs.read"] })`
- Add schema validation: `registerTool(factory, { inputSchema, outputSchema })`
- Support tool versioning for backwards compatibility

### 5.3 Exec Approvals

OpenClaw implements:
- Security modes: `deny`, `allowlist`, `full`
- Ask modes: `off`, `on-miss`, `always`
- Socket-based approval flow (external UI integration)

**Jarvis v2 Recommendation:**
- Adopt socket-based approval for human-in-the-loop
- Add approval timeout with configurable default action
- Log all approval decisions for audit
- Support approval delegation (auto-approve if parent approved)

---

## 6. Implications for State Handling

### 6.1 Persistence Strategy

| State Type | OpenClaw Approach | Jarvis v2 Recommendation |
|------------|------------------|--------------------------|
| Session transcript | JSON file | Evolve: Event log |
| Auth profiles | JSON with lock | Adopt |
| Exec approvals | JSON + socket | Adopt |
| Plugin config | Config file | Evolve: Versioned config store |

### 6.2 Cache Coherence

OpenClaw uses TTL + mtime checking. For Jarvis v2:
- Implement write-through caching (invalidate on write)
- Use generation numbers for optimistic reads
- Consider event sourcing for cache invalidation

### 6.3 Crash Recovery

OpenClaw provides:
- Stale lock eviction (30s timeout)
- Atomic file writes (temp + rename)
- Session normalization on load (migration)

**Jarvis v2 Recommendation:**
- Adopt all three patterns
- Add crash recovery for in-flight agent turns
- Implement session repair for truncated events

---

## 7. Non-Obvious Insights

### 7.1 Timeout as Rate Limit Signal

**Source:** `src/agents/pi-embedded-runner/run.ts:563-579`

OpenClaw treats timeouts as potential rate limits:
```typescript
// Treat timeout as potential rate limit (Antigravity hangs on rate limit)
const shouldRotate = (!aborted && failoverFailure) || timedOut;
```

**Insight:** Some providers silently rate-limit by hanging requests rather than returning 429. Jarvis v2 should treat timeout as a rotation trigger, not just an error.

### 7.2 Tool Result Format Varies by Channel

**Source:** `src/agents/pi-embedded-runner/run.ts:81-87`

```typescript
const resolvedToolResultFormat =
  params.toolResultFormat ??
  (channelHint
    ? isMarkdownCapableMessageChannel(channelHint)
      ? "markdown"
      : "plain"
    : "markdown");
```

**Insight:** Tool output formatting must be channel-aware. Jarvis v2 should support per-channel output transformers.

### 7.3 Plugin-Only Allowlists Get Stripped

**Source:** `src/agents/tool-policy.ts:201-245`

When a tool allowlist contains only plugin tools (no core tools), OpenClaw strips it entirely:
```typescript
// When an allowlist contains only plugin tools, we strip it to avoid accidentally
// disabling core tools.
```

**Insight:** Additive tool configuration is safer than restrictive. Jarvis v2 should prefer `alsoAllow` over `allow` for extension-provided tools.

### 7.4 Billing Failures Require Extended Backoff

**Source:** `src/agents/auth-profiles/usage.ts:106-117`

Billing errors get special treatment:
- Separate `billingBackoffHours` configuration
- Provider-specific overrides (`billingBackoffHoursByProvider`)
- Much longer default (5h base vs 1min for transient errors)

**Insight:** Billing-exhausted credentials are unlikely to recover quickly. Jarvis v2 should implement provider-specific billing failure handling.

### 7.5 Shell Command Analysis Prevents Injection

**Source:** `src/infra/exec-approvals.ts:679-716`

OpenClaw parses shell commands to prevent:
- Pipeline injection (`|`, `||`, `|&`)
- Redirect injection (`>`, `<`)
- Subshell injection (`$()`, backticks)
- Chain operators (`&&`, `;`)

**Insight:** Command allowlisting requires understanding shell semantics, not just string matching. Jarvis v2 should implement semantic command parsing if allowing shell execution.

### 7.6 Session Key Contains Routing Information

**Source:** `src/sessions/session-key-utils.ts`

Session keys encode complete routing context:
```
agent:agentId:channel:to:thread:threadId
```

**Insight:** This couples identity to routing. Useful for simplicity but problematic for session migration. Jarvis v2 should consider separating session identity from routing metadata.

### 7.7 Probe Sessions Are Treated Specially

**Source:** `src/agents/pi-embedded-runner/run.ts:88`

```typescript
const isProbeSession = params.sessionId?.startsWith("probe-") ?? false;
```

Sessions starting with "probe-" have different logging behavior.

**Insight:** System/health-check sessions need different treatment than user sessions. Jarvis v2 should have first-class session categories (user, system, probe).

---

## 8. Summary Recommendations

### Must Adopt
1. Double-queue lane system for concurrency
2. Typed failure classification with differentiated backoff
3. Auth profile rotation with health tracking
4. Tool policy groups and profiles
5. Session write locking with stale eviction
6. Context window guards with auto-compaction

### Must Avoid
1. TTL-only cache invalidation
2. Global mutable plugin registries
3. Mixed sync/async file I/O
4. Regex-based error classification without fallbacks
5. String scrubbing as security measure

### Should Evolve
1. File sessions → Event logs
2. Path allowlists → Capability model
3. Static plugins → Hot-loadable modules
4. Embedded compaction → Memory service
5. Hierarchical keys → Content-addressable references

---

## Appendix: Key Source Files Referenced

| File | Purpose |
|------|---------|
| `src/agents/pi-embedded-runner/run.ts` | Main agent execution loop |
| `src/agents/pi-embedded-runner/lanes.ts` | Queue/lane system |
| `src/agents/tool-policy.ts` | Tool governance framework |
| `src/agents/failover-error.ts` | Error classification |
| `src/agents/auth-profiles/usage.ts` | Credential health tracking |
| `src/config/sessions/store.ts` | Session persistence |
| `src/infra/exec-approvals.ts` | Command authorization |
| `src/plugins/registry.ts` | Plugin registration |

---

*This report is grounded in analysis of commit `c429ccb64fc319babf4f8adc95df6d658a2d6b2f` and references the patterns documented in the ANALYSED_COMMIT.txt file.*
