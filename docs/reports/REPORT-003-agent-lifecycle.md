# REPORT-003: Agent Lifecycle & Execution Model

**Analysis Commit:** `c429ccb64fc319babf4f8adc95df6d658a2d6b2f`
**Analysis Date:** 2026-02-02

## Executive Summary

This report documents the agent lifecycle and execution model in OpenClaw. Unlike simple prompt/response systems, OpenClaw implements a sophisticated multi-layered agent architecture with persistent sessions, nested retry loops, automatic failure recovery, and event-driven streaming. An "agent" is not a single class but a distributed pattern combining configuration, session state, execution context, and identity.

---

## 1. What Constitutes an "Agent" in Code Terms

An agent in OpenClaw is a **composite abstraction** assembled from multiple components at runtime rather than a single unified class.

### 1.1 Agent Identity & Configuration

**Source:** `src/agents/agent-scope.ts`

```typescript
type ResolvedAgentConfig = {
  name?: string;
  workspace?: string;
  agentDir?: string;
  model?: AgentEntry["model"];
  memorySearch?: AgentEntry["memorySearch"];
  humanDelay?: AgentEntry["humanDelay"];
  heartbeat?: AgentEntry["heartbeat"];
  identity?: AgentEntry["identity"];
  groupChat?: AgentEntry["groupChat"];
  subagents?: AgentEntry["subagents"];
  sandbox?: AgentEntry["sandbox"];
  tools?: AgentEntry["tools"];
};
```

Agents are defined in `OpenClawConfig.agents.list` and resolved at runtime via:
- `resolveAgentConfig()` - Retrieves configuration for a specific agent ID
- `resolveAgentWorkspaceDir()` - Determines the agent's working directory
- `resolveAgentDir()` - Locates agent-specific data (skills, settings)

**Key Identifiers:**
- `agentId`: Normalized identifier (e.g., "main", "assistant-2")
- `sessionKey`: Composite key combining agent + channel + sender context
- `sessionId`: UUID for a specific conversation transcript

### 1.2 Session State

**Source:** `src/config/sessions/types.ts:25-96`

The `SessionEntry` type captures all persistent state for an agent session:

```typescript
export type SessionEntry = {
  sessionId: string;               // UUID for transcript file
  updatedAt: number;               // Last activity timestamp
  sessionFile?: string;            // Path to JSONL transcript
  systemSent?: boolean;            // Whether system prompt was sent
  abortedLastRun?: boolean;        // Previous run was interrupted

  // Execution parameters
  thinkingLevel?: string;
  verboseLevel?: string;
  reasoningLevel?: string;
  providerOverride?: string;
  modelOverride?: string;
  authProfileOverride?: string;

  // Queue behavior
  queueMode?: "steer" | "followup" | "collect" | ...;
  queueDebounceMs?: number;
  queueCap?: number;

  // Usage tracking
  inputTokens?: number;
  outputTokens?: number;
  totalTokens?: number;
  contextTokens?: number;
  compactionCount?: number;

  // Delivery context
  channel?: string;
  groupId?: string;
  lastChannel?: SessionChannelId;
  lastTo?: string;

  // Skills snapshot
  skillsSnapshot?: SessionSkillSnapshot;
};
```

### 1.3 Run Context

**Source:** `src/auto-reply/reply/queue/types.ts:21-82`

The `FollowupRun` type encapsulates all parameters needed to execute an agent turn:

```typescript
export type FollowupRun = {
  prompt: string;
  messageId?: string;
  enqueuedAt: number;
  originatingChannel?: OriginatingChannelType;
  run: {
    agentId: string;
    agentDir: string;
    sessionId: string;
    sessionKey?: string;
    sessionFile: string;
    workspaceDir: string;
    config: OpenClawConfig;
    skillsSnapshot?: SkillSnapshot;
    provider: string;
    model: string;
    authProfileId?: string;
    thinkLevel?: ThinkLevel;
    verboseLevel?: VerboseLevel;
    timeoutMs: number;
    ownerNumbers?: string[];
    extraSystemPrompt?: string;
  };
};
```

---

## 2. Agent Initialization and Execution Flow

### 2.1 Entry Points

OpenClaw provides three distinct entry points for agent execution:

#### CLI Command Entry
**Source:** `src/commands/agent.ts:63-526`

```
agentCommand(opts) →
  ├── loadConfig()
  ├── resolveSession() → sessionId, sessionKey, sessionEntry
  ├── buildWorkspaceSkillSnapshot()
  ├── registerAgentRunContext(runId, context)
  ├── runWithModelFallback() →
  │     └── runEmbeddedPiAgent() or runCliAgent()
  ├── updateSessionStoreAfterAgentRun()
  └── deliverAgentCommandResult()
```

The CLI command validates inputs, resolves session state, builds a skills snapshot for new sessions, and delegates to the model fallback system.

#### Gateway RPC Entry
**Source:** `src/gateway/server-methods/agent.ts:44-424`

```
agentHandlers.agent(params) →
  ├── validateAgentParams()
  ├── Dedupe check (idempotencyKey)
  ├── parseMessageWithAttachments()
  ├── injectTimestamp()
  ├── loadSessionEntry()
  ├── resolveSendPolicy() → deny check
  ├── respond(accepted) [immediate ack]
  └── void agentCommand(...) [async execution]
```

The gateway handler provides request/response semantics over WebSocket, handling deduplication via idempotency keys and responding immediately with an "accepted" status before async execution.

#### Auto-Reply Entry
**Source:** `src/auto-reply/reply/agent-runner.ts:47-525`

```
runReplyAgent(params) →
  ├── createTypingSignaler()
  ├── Handle steering (queueEmbeddedPiMessage)
  ├── Handle followup queue (enqueueFollowupRun)
  ├── runMemoryFlushIfNeeded()
  ├── runAgentTurnWithFallback() →
  │     ├── runWithModelFallback() →
  │     │     └── runEmbeddedPiAgent() or runCliAgent()
  │     ├── Handle context overflow → resetSession
  │     └── Handle role ordering conflict → resetSession
  ├── buildReplyPayloads()
  └── finalizeWithFollowup()
```

### 2.2 Session Initialization Sequence

**Source:** `src/agents/pi-embedded-runner/run/attempt.ts:138-905`

The `runEmbeddedAttempt()` function initializes a full agent session:

1. **Workspace Setup** (lines 141-165):
   - Resolve and create workspace directory
   - Apply sandbox context if enabled
   - Load and apply skill environment overrides

2. **Tool Registration** (lines 207-242):
   - Create tools via `createOpenClawCodingTools()`
   - Sanitize for provider compatibility (Google/Gemini)
   - Split into built-in vs custom tools

3. **System Prompt Assembly** (lines 244-393):
   - Build runtime info (host, OS, channel capabilities)
   - Inject skills prompt, docs path, TTS hints
   - Create system prompt report for diagnostics

4. **Session Manager Setup** (lines 395-477):
   - Acquire session write lock
   - Open/create `SessionManager` for transcript
   - Prepare session for run (repair orphaned messages)
   - Create agent session via `createAgentSession()`

5. **History Processing** (lines 530-559):
   - Sanitize session history for provider compatibility
   - Validate turn ordering (Gemini, Anthropic)
   - Limit history to configured bounds

---

## 3. Control Loop and Decision Points

### 3.1 Three-Level Loop Architecture

OpenClaw implements a nested loop structure with distinct responsibilities:

#### Level 1: Reply Agent Loop
**Source:** `src/auto-reply/reply/agent-runner.ts:47-525`

This outer loop handles message queueing and final response assembly:

```
runReplyAgent():
  ├── if (shouldSteer && isStreaming):
  │     queueEmbeddedPiMessage() → return early if steered
  ├── if (isActive && shouldFollowup):
  │     enqueueFollowupRun() → return early (queue for later)
  ├── runMemoryFlushIfNeeded()  # Auto-compaction
  ├── runAgentTurnWithFallback()
  ├── Handle session state updates
  ├── buildReplyPayloads()
  └── finalizeWithFollowup()
```

**Decision Points:**
- Line 162-179: Steer active streaming session
- Line 182-198: Enqueue for followup processing
- Line 202-215: Memory flush before execution
- Line 334-336: Route to finalization on "final" result

#### Level 2: Model Fallback Loop
**Source:** `src/auto-reply/reply/agent-runner-execution.ts:101-602`

This loop handles model-level retries and provider fallback:

```typescript
while (true) {  // Line 101
  try {
    const fallbackResult = await runWithModelFallback({
      run: (provider, model) => {
        if (isCliProvider(provider)) {
          return runCliAgent(...);
        }
        return runEmbeddedPiAgent(...);
      }
    });

    // Check for context overflow in result
    if (isContextOverflowError(embeddedError)) {
      await resetSessionAfterCompactionFailure();
      return { kind: "final", payload: userMessage };
    }

    // Check for role ordering conflict
    if (embeddedError?.kind === "role_ordering") {
      await resetSessionAfterRoleOrderingConflict();
      return { kind: "final", payload: userMessage };
    }

    break;  // Success
  } catch (err) {
    // Error classification and recovery
    if (isCompactionFailure) { reset and return }
    if (isRoleOrderingError) { reset and return }
    if (isSessionCorruption) { cleanup and return }
    // Terminal failure
    return { kind: "final", payload: errorMessage };
  }
}
```

**Decision Points:**
- Line 470-485: Context overflow detection in result
- Line 486-496: Role ordering conflict handling
- Line 506-518: Compaction failure recovery
- Line 519-529: Role ordering error recovery
- Line 532-574: Session corruption cleanup

#### Level 3: Embedded Agent Loop
**Source:** `src/agents/pi-embedded-runner/run.ts:308-686`

The innermost loop handles auth profile rotation and thinking level fallback:

```typescript
while (true) {  // Line 308
  attemptedThinking.add(thinkLevel);

  const attempt = await runEmbeddedAttempt({...});

  if (promptError && !aborted) {
    // Context overflow handling
    if (isContextOverflowError(errorText)) {
      if (!overflowCompactionAttempted) {
        await compactEmbeddedPiSessionDirect();
        if (compactResult.compacted) continue;  // Retry after compaction
      }
      return errorPayload;
    }

    // Auth profile rotation
    if (isFailoverErrorMessage(errorText)) {
      if (await advanceAuthProfile()) continue;
    }

    // Thinking level fallback
    const fallbackThinking = pickFallbackThinkingLevel({...});
    if (fallbackThinking) {
      thinkLevel = fallbackThinking;
      continue;
    }
  }

  // Auth/rate limit rotation on streaming failure
  if (shouldRotate) {
    await markAuthProfileFailure();
    if (await advanceAuthProfile()) continue;
  }

  // Success - mark profile good and return
  await markAuthProfileGood();
  return result;
}
```

**Decision Points:**
- Line 374-410: Context overflow → auto-compaction
- Line 493-498: Auth failure → profile rotation
- Line 500-510: Thinking level fallback
- Line 566-594: Rate limit → profile rotation
- Line 650-662: Success → mark profile good

### 3.2 Core Execution: The Prompt Loop

**Source:** `src/agents/pi-embedded-runner/run/attempt.ts:700-862`

Within a single attempt, the agent enters a prompt/response loop managed by the underlying `@mariozechner/pi-coding-agent` library:

```typescript
// Line 796-802
if (imageResult.images.length > 0) {
  await abortable(activeSession.prompt(effectivePrompt, { images }));
} else {
  await abortable(activeSession.prompt(effectivePrompt));
}
```

The prompt call triggers:
1. Message submission to the LLM
2. Tool execution (if tools are called)
3. Response streaming
4. Transcript persistence

Tool calls create a nested loop where the agent may execute multiple tools before producing a final response.

---

## 4. State Propagation and Mutation

### 4.1 State Flow Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                        INITIAL STATE                             │
│   SessionEntry loaded from sessions.json                        │
└─────────────────────────────────┬───────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────┐
│                     IN-MEMORY COPY                              │
│   activeSessionEntry + activeSessionStore (mutable)             │
└─────────────────────────────────┬───────────────────────────────┘
                                  │
                    ┌─────────────┴─────────────┐
                    │                           │
                    ▼                           ▼
┌──────────────────────────┐    ┌──────────────────────────────┐
│    DURING EXECUTION      │    │      TRANSCRIPT FILE         │
│ • updatedAt timestamp    │    │  SessionManager writes to    │
│ • Token usage            │    │  ~/.openclaw/sessions/*.jsonl│
│ • Compaction count       │    │                              │
│ • Model/provider used    │    │                              │
└───────────────┬──────────┘    └──────────────────────────────┘
                │
                ▼
┌─────────────────────────────────────────────────────────────────┐
│                     PERSISTENCE                                  │
│   updateSessionStore() → sessions.json                          │
│   persistSessionUsageUpdate() → usage fields                    │
└─────────────────────────────────────────────────────────────────┘
```

### 4.2 Key Mutation Points

**Session Entry Updates:**
**Source:** `src/auto-reply/reply/agent-runner.ts:165-176`

```typescript
activeSessionEntry.updatedAt = Date.now();
activeSessionStore[sessionKey] = activeSessionEntry;
```

**Usage Persistence:**
**Source:** `src/auto-reply/reply/agent-runner.ts:387-396`

```typescript
await persistSessionUsageUpdate({
  storePath,
  sessionKey,
  usage,
  modelUsed,
  providerUsed,
  contextTokensUsed,
  systemPromptReport,
  cliSessionId,
});
```

**Compaction Count Increment:**
**Source:** `src/auto-reply/reply/agent-runner.ts:498-503`

```typescript
const count = await incrementCompactionCount({
  sessionEntry: activeSessionEntry,
  sessionStore: activeSessionStore,
  sessionKey,
  storePath,
});
```

### 4.3 Session Reset on Failure

**Source:** `src/auto-reply/reply/agent-runner.ts:235-294`

When a session must be reset (compaction failure, role ordering conflict, corruption):

```typescript
const resetSession = async ({ failureLabel, buildLogMessage, cleanupTranscripts }) => {
  const nextSessionId = crypto.randomUUID();
  const nextEntry: SessionEntry = {
    ...prevEntry,
    sessionId: nextSessionId,
    updatedAt: Date.now(),
    systemSent: false,
    abortedLastRun: false,
  };

  // Update in-memory state
  activeSessionStore[sessionKey] = nextEntry;

  // Persist to disk
  await updateSessionStore(storePath, (store) => {
    store[sessionKey] = nextEntry;
  });

  // Update followup run context
  followupRun.run.sessionId = nextSessionId;
  followupRun.run.sessionFile = nextSessionFile;
  activeSessionEntry = nextEntry;

  // Cleanup old transcript if requested
  if (cleanupTranscripts && prevSessionId) {
    fs.unlinkSync(transcriptPath);
  }
};
```

---

## 5. Failure Handling and Recovery Semantics

### 5.1 Recovery Hierarchy

OpenClaw implements a three-tiered recovery strategy:

#### Tier A: Silent Auto-Recovery (No User Awareness)

| Failure Type | Detection | Recovery Action | Source |
|-------------|-----------|-----------------|--------|
| Context Overflow | `isContextOverflowError()` | Auto-compaction | `run.ts:377-409` |
| Auth/Rate Limit | `isFailoverAssistantError()` | Rotate auth profile | `run.ts:566-594` |
| Thinking Unsupported | `pickFallbackThinkingLevel()` | Downgrade level | `run.ts:500-510` |
| Timeout (potential rate limit) | `timedOut` flag | Rotate auth profile | `run.ts:564-594` |

**Auto-Compaction Flow:**
```typescript
if (isContextOverflowError(errorText)) {
  if (!overflowCompactionAttempted) {
    overflowCompactionAttempted = true;
    const compactResult = await compactEmbeddedPiSessionDirect({...});
    if (compactResult.compacted) {
      log.info(`auto-compaction succeeded; retrying prompt`);
      continue;  // Re-enter main loop
    }
  }
}
```

#### Tier B: Session Reset (Minor User Awareness)

User receives a message explaining the reset:

| Failure Type | User Message | Source |
|-------------|--------------|--------|
| Compaction Failure | "Context limit exceeded. I've reset our conversation..." | `agent-runner-execution.ts:480-484` |
| Role Ordering Conflict | "Message ordering conflict. I've reset the conversation..." | `agent-runner-execution.ts:489-495` |
| Session Corruption | "Session history was corrupted. I've reset the conversation..." | `agent-runner-execution.ts:568-573` |

**Compaction Failure Recovery:**
**Source:** `src/auto-reply/reply/agent-runner-execution.ts:506-518`

```typescript
if (isCompactionFailure && !didResetAfterCompactionFailure) {
  await params.resetSessionAfterCompactionFailure(message);
  didResetAfterCompactionFailure = true;
  return {
    kind: "final",
    payload: {
      text: "⚠️ Context limit exceeded during compaction. I've reset our conversation..."
    },
  };
}
```

#### Tier C: Terminal Failure (Explicit Error)

No recovery possible; user sees error message:

| Failure Type | Handling | Source |
|-------------|----------|--------|
| Image size/dimension error | Return error payload | `run.ts:456-482` |
| Context window too small | Throw `FailoverError` | `run.ts:134-138` |
| All auth profiles exhausted | Throw or return error | `run.ts:596-623` |
| Model not found | Throw error | `run.ts:109-111` |

### 5.2 Auth Profile Management

**Source:** `src/agents/pi-embedded-runner/run.ts:249-274`

The `advanceAuthProfile()` function implements profile rotation:

```typescript
const advanceAuthProfile = async (): Promise<boolean> => {
  if (lockedProfileId) return false;  // User-locked profile

  let nextIndex = profileIndex + 1;
  while (nextIndex < profileCandidates.length) {
    const candidate = profileCandidates[nextIndex];
    if (candidate && isProfileInCooldown(authStore, candidate)) {
      nextIndex += 1;
      continue;  // Skip profiles in cooldown
    }
    try {
      await applyApiKeyInfo(candidate);
      profileIndex = nextIndex;
      thinkLevel = initialThinkLevel;  // Reset thinking level
      attemptedThinking.clear();
      return true;
    } catch (err) {
      nextIndex += 1;
    }
  }
  return false;  // No more profiles available
};
```

### 5.3 Event Emission System

**Source:** `src/infra/agent-events.ts:57-83`

All lifecycle events are emitted via a centralized system:

```typescript
export function emitAgentEvent(event: Omit<AgentEventPayload, "seq" | "ts">) {
  const nextSeq = (seqByRun.get(event.runId) ?? 0) + 1;
  seqByRun.set(event.runId, nextSeq);

  const enriched: AgentEventPayload = {
    ...event,
    sessionKey,
    seq: nextSeq,
    ts: Date.now(),
  };

  for (const listener of listeners) {
    try {
      listener(enriched);
    } catch { /* ignore */ }
  }
}
```

**Event Streams:**
```typescript
type AgentEventStream = "lifecycle" | "tool" | "assistant" | "error" | (string & {});
```

Lifecycle phases: `start`, `end`, `error`

---

## 6. Differences from Simple Prompt/Response Systems

### 6.1 Comparison Table

| Aspect | Simple Prompt/Response | OpenClaw Agent |
|--------|----------------------|----------------|
| **State** | Stateless | Persistent sessions with JSONL transcripts |
| **Response** | Wait for complete response | Streaming with partial replies, block chunking |
| **Failures** | Fail on first error | 3+ nested retry loops with auto-recovery |
| **Tools** | None or isolated calls | Integrated tool loop with result injection |
| **Queueing** | One-off request | Message queue with steer/followup/collect modes |
| **Events** | None | Full lifecycle events to multiple listeners |
| **Auth** | Single credential | Multi-profile rotation with cooldown |
| **Context** | Fixed window | Auto-compaction and cache-TTL pruning |

### 6.2 Multi-Turn State Machine

Simple systems are stateless:
```
Request → Process → Response (done)
```

OpenClaw maintains long-lived sessions:
```
Request → Load Session → Process → Update Session → Response
    ↑                                                   │
    └───────────────────────────────────────────────────┘
                    (next turn)
```

**Session Persistence:**
- Configuration: `~/.openclaw/sessions.json`
- Transcripts: `~/.openclaw/sessions/<sessionId>.jsonl`

### 6.3 Streaming Architecture

**Source:** `src/agents/pi-embedded-subscribe.ts:30-564`

The subscription system provides real-time streaming:

```typescript
export function subscribeEmbeddedPiSession(params) {
  const state: EmbeddedPiSubscribeState = {
    assistantTexts: [],
    toolMetas: [],
    deltaBuffer: "",
    blockBuffer: "",
    blockState: { thinking: false, final: false },
    // ... 40+ state fields
  };

  const unsubscribe = params.session.subscribe(
    createEmbeddedPiSessionEventHandler(ctx)
  );

  return {
    assistantTexts,
    toolMetas,
    unsubscribe,
    isCompacting: () => state.compactionInFlight,
    waitForCompactionRetry: () => state.compactionRetryPromise,
    // ...
  };
}
```

**Block Reply Pipeline:**
- Coalesces streaming chunks
- Strips `<think>` and `<final>` tags
- Deduplicates against messaging tool outputs
- Applies reply directives (media URLs, reply targeting)

### 6.4 Message Queueing

**Source:** `src/auto-reply/reply/queue/types.ts:8-17`

```typescript
type QueueMode = "steer" | "followup" | "collect" | "steer-backlog" | "interrupt" | "queue";

type QueueSettings = {
  mode: QueueMode;
  debounceMs?: number;
  cap?: number;
  dropPolicy?: "old" | "new" | "summarize";
};
```

**Queue Modes:**
- `steer`: Inject into active streaming session
- `followup`: Queue for processing after current reply
- `collect`: Batch messages with debouncing
- `interrupt`: Abort current and process new message

### 6.5 Tool Execution Integration

Unlike simple systems where tool calls are external, OpenClaw integrates tools directly into the agent loop:

```
Agent Prompt → LLM Response (may include tool calls)
      ↓
Tool Execution (bash, file ops, messaging, etc.)
      ↓
Tool Result Injection → Continue LLM Generation
      ↓
Repeat until final response (no tool calls)
```

**Tool Result Streaming:**
**Source:** `src/agents/pi-embedded-subscribe.ts:229-287`

```typescript
const emitToolSummary = (toolName?: string, meta?: string) => {
  const agg = formatToolAggregate(toolName, meta ? [meta] : undefined);
  void params.onToolResult({ text: cleanedText, mediaUrls });
};
```

---

## 7. Key Files Summary

| Component | File | Key Functions |
|-----------|------|---------------|
| Agent Identity | `src/agents/agent-scope.ts` | `resolveAgentConfig`, `resolveAgentWorkspaceDir` |
| Session Types | `src/config/sessions/types.ts` | `SessionEntry`, `mergeSessionEntry` |
| CLI Entry | `src/commands/agent.ts` | `agentCommand` |
| Gateway Entry | `src/gateway/server-methods/agent.ts` | `agentHandlers.agent` |
| Reply Runner | `src/auto-reply/reply/agent-runner.ts` | `runReplyAgent` |
| Execution Loop | `src/auto-reply/reply/agent-runner-execution.ts` | `runAgentTurnWithFallback` |
| Embedded Runner | `src/agents/pi-embedded-runner/run.ts` | `runEmbeddedPiAgent` |
| Attempt Execution | `src/agents/pi-embedded-runner/run/attempt.ts` | `runEmbeddedAttempt` |
| Event System | `src/infra/agent-events.ts` | `emitAgentEvent`, `onAgentEvent` |
| Session Subscription | `src/agents/pi-embedded-subscribe.ts` | `subscribeEmbeddedPiSession` |
| Queue Types | `src/auto-reply/reply/queue/types.ts` | `FollowupRun`, `QueueSettings` |

---

## 8. Architectural Observations

1. **Separation of Concerns**: Agent identity, session state, and execution context are cleanly separated, allowing flexible composition.

2. **Defense in Depth**: Multiple recovery mechanisms (auto-compaction, profile rotation, session reset) ensure robust operation even under adverse conditions.

3. **Observable Execution**: The event system provides full visibility into agent lifecycle, enabling monitoring, debugging, and UI updates.

4. **Provider Abstraction**: The model fallback system and auth profile rotation abstract away provider-specific concerns from core agent logic.

5. **Stateful Streaming**: Unlike simple streaming, OpenClaw maintains state across chunks to handle `<think>` blocks, reply directives, and duplicate detection.

---

*Report generated from commit `c429ccb64fc319babf4f8adc95df6d658a2d6b2f`*
