# AgentGuard — Final Technical Plan
**Android AI Agent Governance SDK**

> Status: Planning / Pre-Implementation
> Version: 0.1 (Living Document)
> Last Updated: 2026-03-06

---

## Table of Contents

1. [Product Definition](#1-product-definition)
2. [Explicit Trust Model](#2-explicit-trust-model)
3. [Threat Model](#3-threat-model)
4. [Observation Architecture (Solving the Core Gap)](#4-observation-architecture)
5. [Audit Log Integrity](#5-audit-log-integrity)
6. [Network Access Governance](#6-network-access-governance)
7. [Mismatch Detection and Enforcement](#7-mismatch-detection-and-enforcement)
8. [Consent System (Full Lifecycle)](#8-consent-system-full-lifecycle)
9. [LLM Runtime Integration](#9-llm-runtime-integration)
10. [Multi-Agent Architecture](#10-multi-agent-architecture)
11. [Background Execution](#11-background-execution)
12. [SDK Architecture and Module Structure](#12-sdk-architecture-and-module-structure)
13. [Core Data Model (Revised)](#13-core-data-model-revised)
14. [Policy Engine (Revised)](#14-policy-engine-revised)
15. [Agent Execution Pipeline (Revised)](#15-agent-execution-pipeline-revised)
16. [Developer Integration Model](#16-developer-integration-model)
17. [Build-Time Instrumentation (Phase 2)](#17-build-time-instrumentation-phase-2)
18. [Developer Experience and Tooling](#18-developer-experience-and-tooling)
19. [OEM and Enterprise Integration](#19-oem-and-enterprise-integration)
20. [MVP Scope (Phase 1)](#20-mvp-scope-phase-1)
21. [Ecosystem Roadmap](#21-ecosystem-roadmap)
22. [What AgentGuard Cannot Do (Honest Boundaries)](#22-what-agentguard-cannot-do)

---

## 1. Product Definition

AgentGuard is an Android SDK that acts as a **cooperative governance and audit layer** between AI agents and the Android environment.

**One-line definition:**
An Android SDK that enables transparent, policy-governed, and auditable execution of AI agents running locally on device — for developers, enterprises, and OEMs.

**Core value:**
- Users see what an AI agent does, why it does it, and can control it
- Developers get a governance layer they do not have to build themselves
- Enterprises get audit trails, policy enforcement, and compliance tooling
- OEMs get a system-level governance foundation for on-device AI

---

## 2. Explicit Trust Model

This is the most important section. Every design decision flows from this.

### AgentGuard governs cooperative agents

AgentGuard is a **transparency and governance layer for agents that integrate honestly**. It is not a security sandbox and does not protect against malicious code.

| Scenario | AgentGuard behavior |
|---|---|
| Developer integrates SDK correctly | Full governance, logging, consent, policy |
| Developer partially integrates | Partial visibility — SDK can only observe what it wraps |
| Developer skips SDK entirely | No visibility — SDK cannot observe agents that bypass it |
| Agent calls Android APIs directly (not through wrappers) | Phase 1: invisible. Phase 2: intercepted via build plugin |
| Malicious app | Out of scope for Phase 1. Addressed partially in Phase 2 via bytecode instrumentation |

### What this means in practice

**Phase 1 (MVP):** Trust-based. Works when the developer integrates honestly. The SDK must be transparent about this in its documentation. The value it provides is still real: it removes the burden of building governance UX, logging, and consent flows from developers who want to do the right thing.

**Phase 2:** Enforcement-assisted. A Gradle plugin instruments the build to intercept direct Android API calls, making cooperation automatic rather than voluntary.

**This framing must appear in all public documentation, marketing, and API contracts.**

---

## 3. Threat Model

Define what AgentGuard protects against and what it does not.

### In scope (Phase 1)
- **Unintentional over-reach:** An agent accidentally accesses more data than intended — SDK prevents this via consent and policy
- **Silent data access:** An agent reads sensitive data without user awareness — SDK surfaces this
- **Missing audit trail:** No record of what the agent did — SDK provides complete event log
- **Inconsistent governance UX:** Every developer builds their own consent dialogs — SDK standardizes this
- **Intent-execution mismatch:** Agent declares one plan but runs different code — SDK detects and flags this

### In scope (Phase 2)
- **Uninstrumented API calls:** Agent bypasses SDK wrappers and calls Android APIs directly — Gradle plugin intercepts at build time
- **Network exfiltration:** Agent silently sends data to external endpoints — OkHttp interceptor + network policy

### Out of scope (explicit)
- **Malicious first-party developers:** A developer who deliberately excludes the SDK cannot be governed
- **Rooted devices:** Root access can bypass all Android security primitives including AgentGuard
- **Kernel-level attacks:** Out of scope for any application-layer SDK
- **Cross-app agent injection:** Phase 3 only

### Attacker personas (who the SDK protects against)
1. **Negligent developer** — doesn't build consent UX, doesn't think about privacy. SDK provides defaults.
2. **Over-reaching agent** — LLM that requests more permissions than its task requires. Policy engine blocks this.
3. **Future-self problem** — developer upgrades an agent's capabilities but forgets to update consent flows. SDK enforces re-consent via capability manifest versioning.

---

## 4. Observation Architecture

### The core problem

All three Phase 1 observation mechanisms (declared steps, tool wrapping, tool metadata) depend on voluntary developer cooperation. This is honest and acceptable for Phase 1, but must be supplemented.

### Phase 1: Cooperative Observation (Three Layers)

**Layer 1 — Declared Intent**
- Agent/app calls `agentGuard.declareGoal(goal)` and `agentGuard.declarePlan(steps[])`
- SDK stores the declared plan as the baseline
- Mismatch detection compares this against actual observed execution

**Layer 2 — SDK-Wrapped Tools**
- All sensitive operations go through SDK-provided wrapper functions
- Each wrapper is registered with metadata (category, sensitivity, permissions required, explanation template)
- The gateway intercepts, logs, checks policy, requests consent if needed, then executes

**Layer 3 — Tool Metadata Registry**
- At agent registration time, the full capability manifest is declared (not just at session start)
- Manifest includes: all tools the agent may call, sensitivity levels, access patterns
- Changes to the manifest trigger re-consent from the user

### Phase 2: Instrumentation-Assisted Observation

**Gradle Plugin — Build-Time Bytecode Instrumentation**
- Intercepts calls to Android platform APIs at compile time using ASM/bytecode transformation
- Target APIs: `ContentResolver`, `SharedPreferences`, `Room` DAO calls, `File` operations, `OkHttpClient`, `HttpURLConnection`
- Wraps these calls with AgentGuard reporting hooks automatically
- Developer does not need to remember to wrap each call manually
- Instrumentation is scoped — only applies within the agent execution context (tracked via session thread context)

**Scope control for instrumentation:**
- Instrumentation only activates inside an active `AgentSession`
- Calls made outside a session are not intercepted (avoids performance overhead on non-agent code)
- Configurable: developers can opt specific classes out of instrumentation

### What cannot be observed even in Phase 2
- JNI calls from native libraries
- Agents using reflection to bypass instrumentation
- Network calls made from WebView or embedded browser engines

This must be documented clearly.

---

## 5. Audit Log Integrity

### Problems with naive encrypted storage
- The app holds the encryption key — the app can decrypt and alter logs
- The app can call `clearDatabase()` — logs can be destroyed
- No way to detect tampering after the fact

### Solution: Hash-Chained Append-Only Audit Log

**Structure of each log entry:**
```
LogEntry {
    entryId: UUID
    timestamp: Long (monotonic clock, not wall clock — cannot be rewound)
    sessionId: UUID
    eventType: String
    payload: JSON
    previousEntryHash: SHA-256  // chain link
    entryHash: SHA-256(previousEntryHash + timestamp + payload)
    signature: Base64            // signed with device key in Android Keystore
}
```

**Properties:**
- **Append-only enforced at API level:** No update or delete operations exposed. Only `append()`.
- **Hash chaining:** Each entry references the hash of the previous entry. Any tampering breaks the chain. Integrity can be verified at any time.
- **Android Keystore signing:** Each entry is signed using a key that never leaves the secure hardware. The app cannot forge signatures even if it reads the database.
- **Monotonic timestamps:** Using `SystemClock.elapsedRealtime()` alongside wall clock. Prevents timestamp manipulation.

**Key ownership:**
- Signing key lives in Android Keystore, hardware-backed where available
- Key is generated per-installation and cannot be exported
- Verification of log integrity is a public operation (signature verification with the device's public key)

**Enterprise extension:**
- Logs can be exported (signed, encrypted) to enterprise MDM or SIEM
- Export is a one-way operation: read-only view of the on-device log

**User data in logs:**
- Tool inputs and outputs are stored in redacted form by default
- Full capture mode is opt-in and requires explicit user consent
- Sensitivity-level-based redaction: `HIGH` sensitivity fields are hashed, not stored in plaintext

---

## 6. Network Access Governance

### The gap
The original document had no mention of network access. An agent can silently exfiltrate data via HTTP while declaring "local-only" operation. This is the most serious unaddressed gap.

### Solution: Network Declaration + Interception

**Phase 1 — Declarative approach:**
- Tool metadata must declare whether a tool makes network calls
- Tools with `networkAccess: true` require explicit consent even if data access seems benign
- "Local-only" agents must have `networkAccess: false` on all tools
- If a tool with `networkAccess: false` attempts an HTTP call (detected via OkHttp interceptor), it is flagged as a mismatch

**Phase 1 OkHttp Interceptor:**
```
AgentGuardNetworkInterceptor implements Interceptor {
    - Checks if the call is made within an active AgentSession
    - Checks if the current tool's metadata declares networkAccess: true
    - If not declared: FLAG_AND_BLOCK (Phase 1) or FLAG_AND_LOG (debug mode)
    - Logs: destination URL (domain only, not full path), request size, response size
    - Does NOT log request/response body (privacy-preserving by default)
}
```

**Registration:**
- Developers who use OkHttp add the interceptor to their client
- Developers using other HTTP clients (Retrofit, Ktor, Volley) are given wrapper factories

**Phase 2 — Bytecode instrumentation** intercepts `HttpURLConnection` and other network primitives automatically

**Policy options for network access:**
- `BLOCK_ALL_NETWORK` — no agent in this app may make network calls
- `ALLOW_DECLARED_ONLY` — only tools explicitly declaring networkAccess may call network
- `ALLOW_WITH_LOG` — allow but log all network calls
- `ENTERPRISE_ALLOWLIST` — only specific domains are permitted

---

## 7. Mismatch Detection and Enforcement

### The gap
Section 8 of the original document said "SDK flags suspicious behavior" but did not define what happens after flagging.

### Enforcement Actions (defined)

Every policy rule and every mismatch handler must specify one of these enforcement actions:

| Action | Behavior |
|---|---|
| `LOG_ONLY` | Record the mismatch, allow execution to continue. Use in debug/monitoring mode. |
| `FLAG_AND_NOTIFY` | Record, allow execution, show user a notification that something unexpected occurred. |
| `FLAG_AND_BLOCK` | Prevent the execution of the mismatched call. Log the attempt. Return an error to the agent. |
| `FLAG_AND_SUSPEND` | Suspend the entire session. Show the user a dialog describing the mismatch. User decides: resume or terminate. |
| `FLAG_AND_TERMINATE` | Immediately end the session. Mark it as `TERMINATED_POLICY_VIOLATION`. |

### Mismatch types

| Mismatch | Default enforcement |
|---|---|
| Agent calls a tool not in its declared plan | `FLAG_AND_NOTIFY` |
| Agent calls a tool not in its capability manifest | `FLAG_AND_BLOCK` |
| Agent calls a tool with `networkAccess: false` that makes a network call | `FLAG_AND_BLOCK` |
| Agent accesses a resource category not declared in manifest | `FLAG_AND_SUSPEND` |
| Hash chain broken in audit log (tamper detected) | `FLAG_AND_TERMINATE` + alert |
| Agent calls tool during a revoked consent scope | `FLAG_AND_BLOCK` |

**All defaults are configurable** per-agent via policy rules. Enterprise policies can override with stricter defaults.

### Important timing note
FLAG_AND_BLOCK acts **before** the actual API call executes — the SDK gateway intercepts the intent, evaluates policy, and only proceeds if approved. This means blocking is genuinely preventive, not just post-hoc logging.

---

## 8. Consent System (Full Lifecycle)

### Consent states
```
ConsentRecord {
    consentId: UUID
    agentId: String
    resource: ResourceCategory
    decision: ALLOW_ONCE | ALLOW_SESSION | ALLOW_ALWAYS | DENY
    scope: FOREGROUND_ONLY | BACKGROUND_ALLOWED
    grantedAt: Long
    expiresAt: Long? (null = no expiry for ALLOW_ALWAYS)
    revokedAt: Long? (null = not revoked)
    revokedReason: String?
}
```

### Consent UI timeout
- When a consent dialog is shown, a configurable timeout applies
- Default timeout: **30 seconds**
- On timeout, the **default is DENY** — the safest option
- Timeout behavior is configurable per-agent: `DENY_ON_TIMEOUT` (default) or `SUSPEND_ON_TIMEOUT`
- A suspended session waits for the next time the user opens the app

### Consent revocation (mid-session)
Users can revoke consent at any time from:
- The live execution timeline view
- The session history view
- The system notification (for background agents)

**What happens when consent is revoked mid-session:**
1. The current in-progress step is allowed to **complete** (cannot interrupt atomic operations safely)
2. The step result is **discarded** — not passed to the agent — and logged as `RESULT_SUPPRESSED_REVOKED_CONSENT`
3. All subsequent steps that require the revoked resource are **blocked**
4. The session status is updated to `SUSPENDED_USER_REVOKED`
5. The agent receives a structured error: `AgentGuardException(CONSENT_REVOKED, resource=X)`
6. The user sees a summary: "You revoked calendar access. 2 remaining steps were cancelled."

**No rollback for completed steps:**
This is explicitly documented — AgentGuard cannot undo data that was already read or written. Rollback is not feasible at the SDK layer for arbitrary operations. This limitation is disclosed to users.

### Consent for background agents
- Background execution requires **BACKGROUND_ALLOWED** scope, which requires its own explicit consent
- If a background agent needs a resource and no consent exists, it queues a notification-based consent request
- The agent's step is held in `PENDING_CONSENT` state until the user responds
- If the user does not respond within a configurable window (default: 24 hours), the step is denied

---

## 9. LLM Runtime Integration

### Supported targets (Phase 1)
| Runtime | Integration approach |
|---|---|
| MediaPipe LLM Inference API | `AgentGuardMediaPipeAdapter` wraps `LlmInferenceTask` |
| ONNX Runtime Mobile | `AgentGuardOnnxAdapter` wraps inference session |
| llama.cpp (via JNI) | `AgentGuardLlamaCppAdapter` wraps JNI inference calls |
| Ollama (local HTTP server) | `AgentGuardOllamaAdapter` wraps HTTP calls via OkHttp interceptor |

### What the adapter does
- Wraps the inference call so the SDK knows when reasoning is happening vs when tool execution is happening
- Captures: model name/version, token counts (input/output), inference duration
- Does NOT capture: the prompt content or response content (privacy-preserving by default)
- Full prompt capture is opt-in enterprise feature with explicit user consent

### Session correlation
- LLM inference events are tagged with the current `sessionId` and `stepId`
- This allows the audit log to show: "During step 3, the model ran inference for 1.2s consuming 847 tokens"

### AgentGuard does not read LLM reasoning
This is a hard constraint — the SDK operates on declared intent and observed tool execution, never on model weights, activations, or prompt content. This makes the system model-agnostic and privacy-preserving.

---

## 10. Multi-Agent Architecture

### Session isolation model
- Each agent runs in its own `AgentSession` with a unique `sessionId`
- Sessions are fully isolated by default: Agent A cannot read Agent B's session data
- Shared resources (e.g., calendar) are governed independently — each agent must obtain its own consent

### Concurrent agent sessions
- Multiple sessions can be active simultaneously
- Each session runs on its own coroutine scope / execution context
- The policy engine evaluates each session independently
- Resource contention is handled: if Agent A holds an exclusive lock on a resource, Agent B's request is queued or denied per policy

### Agent-to-agent communication
- Phase 1: Not supported. Agents cannot call each other.
- Phase 3: Explicit agent-to-agent communication with its own governance layer (declared call graph, consent for data passed between agents)

### Session hierarchy (for chained agents)
- Phase 2: Support for `parentSessionId` — a supervisor agent can spawn sub-agents
- Sub-agent inherits the parent's approved consent scope by default (configurable)
- Sub-agent's actions are attributed to the sub-agent session but linked to the parent in the audit log

---

## 11. Background Execution

### The problem
Consent dialogs cannot be shown when the app is in the background (WorkManager, foreground Service, JobScheduler).

### Solution: Background consent model

**Pre-approval (preferred):**
- Before the agent runs in the background, the app prompts the user to approve background execution with specific resource scope
- Consent record includes `scope: BACKGROUND_ALLOWED`
- The background agent can proceed without interrupting the user

**Deferred consent (fallback):**
- If the agent encounters a resource it has no consent for while backgrounded:
  1. The step enters `PENDING_USER_APPROVAL` state
  2. A persistent notification is shown: "Task Agent is waiting for your approval to read your calendar"
  3. Tapping the notification opens the consent dialog in the app
  4. The step resumes after approval
  5. If the user does not respond within the configured window, the step is denied

**Background-safe policy rules:**
- Developers can configure: `backgroundBehavior: DENY_IF_NO_PRIOR_CONSENT` (safest) or `backgroundBehavior: QUEUE_FOR_USER`
- Resources marked `HIGH` sensitivity always require foreground consent — they cannot be pre-approved for background

**Background audit continuity:**
- All background events are logged with the same guarantee as foreground events
- Background log entries are marked with `executionContext: BACKGROUND`

---

## 12. SDK Architecture and Module Structure

### Modules

```
agentguard-core/
    Session lifecycle management
    Event bus (internal)
    Policy engine
    Consent manager
    Audit log writer (append-only, hash-chained)
    Mismatch detector

agentguard-tools/
    Standard tool wrappers:
        CalendarTool (read events, write events)
        ContactsTool (read contacts)
        FilesTool (read/write local files)
        TasksTool (read/write task data)
        NotesTool (read/write notes)
        NetworkTool (declared HTTP access)
        MediaTool (camera, microphone — declared only in Phase 1)
    Tool registry
    Tool metadata schema

agentguard-ui/
    ConsentDialog (pre-built, customizable)
    ExecutionTimeline (live step progress view)
    StepDetailView (per-step explainer)
    SessionHistoryView (past sessions browser)
    AccessSummaryView (data accessed summary)
    BackgroundConsentNotification

agentguard-llm/
    MediaPipeAdapter
    OnnxAdapter
    LlamaCppAdapter
    OllamaAdapter
    LLMSessionWrapper (base interface)

agentguard-network/
    AgentGuardNetworkInterceptor (OkHttp)
    RetrofitCallAdapterFactory
    NetworkPolicyEvaluator

agentguard-gradle/        [Phase 2]
    BytecodeInstrumentationPlugin
    ContentResolverInterceptor
    SharedPreferencesInterceptor
    RoomInterceptor
    FileApiInterceptor

agentguard-enterprise/    [Phase 2]
    MDMPolicyProvider
    RemotePolicySync
    AuditLogExporter
    SIEMConnector

agentguard-test/
    MockToolRegistry
    SimulatedPolicyEngine
    SessionReplayEngine
    ConsentDialogTestHelper
    AuditLogVerifier
```

### Internal event bus
All modules communicate through a typed internal event bus. No module holds a direct reference to another. This ensures:
- Modules can be included/excluded independently
- Testing any module in isolation is straightforward
- Enterprise modules can add listeners without modifying core

---

## 13. Core Data Model (Revised)

### Agent
```
Agent {
    agentId: String          // stable identifier
    name: String
    version: SemVer
    publisher: String
    capabilityManifest: AgentCapabilityManifest
    trustProfile: TrustProfile
    registeredAt: Long
    manifestHash: SHA-256    // used to detect manifest changes requiring re-consent
}
```

### AgentCapabilityManifest (registered at install/first-run, not just session start)
```
AgentCapabilityManifest {
    agentId: String
    manifestVersion: SemVer
    declaredTools: List<ToolId>
    declaredResourceCategories: List<ResourceCategory>
    networkAccess: Boolean
    backgroundExecution: Boolean
    maxSessionDurationMinutes: Int
    dataRetentionDays: Int
}
```

### Session
```
Session {
    sessionId: UUID
    agentId: String
    parentSessionId: UUID?    // for sub-agent hierarchy
    goal: String
    executionContext: FOREGROUND | BACKGROUND
    startTime: Long
    endTime: Long?
    status: ACTIVE | COMPLETED | SUSPENDED | TERMINATED_USER | TERMINATED_POLICY_VIOLATION | TERMINATED_ERROR
    plannedSteps: List<StepDescriptor>
    consentScope: List<ConsentRecord>
}
```

### StepEvent (revised)
```
StepEvent {
    stepId: UUID
    sessionId: UUID
    sequenceNumber: Int
    title: String
    reason: String
    plannedToolIds: List<ToolId>    // from declared plan
    actualToolId: ToolId?           // from observed execution
    resourceAccessed: ResourceCategory?
    status: PENDING | IN_PROGRESS | COMPLETED | BLOCKED | MISMATCH_FLAGGED
    startTime: Long
    endTime: Long?
    durationMs: Long                // for performance auditing
    policyDecision: PolicyDecision?
    consentDecision: ConsentDecision?
    mismatchType: MismatchType?
    executionContext: FOREGROUND | BACKGROUND
}
```

### Tool
```
Tool {
    toolId: String
    category: ResourceCategory
    sensitivity: LOW | MEDIUM | HIGH | CRITICAL
    description: String
    networkAccess: Boolean
    permissionsRequired: List<AndroidPermission>
    explanationTemplates: ExplanationTemplates  // user / operational / technical
    requiresConsent: Boolean
    backgroundCapable: Boolean
}
```

### PolicyRule (revised)
```
PolicyRule {
    ruleId: String
    condition: PolicyCondition      // when this rule applies
    action: PolicyAction            // LOG_ONLY | FLAG_AND_NOTIFY | FLAG_AND_BLOCK | FLAG_AND_SUSPEND | FLAG_AND_TERMINATE
    priority: Int
    source: DEVELOPER | ENTERPRISE | OEM | USER
    appliesTo: ALL_AGENTS | List<AgentId>
}
```

### AuditLogEntry (tamper-evident)
```
AuditLogEntry {
    entryId: UUID
    timestamp: Long                 // wall clock
    monotonicTimestamp: Long        // SystemClock.elapsedRealtime()
    sessionId: UUID
    agentId: String
    eventType: EventType
    payload: Map<String, Any>       // sensitivity-redacted
    previousEntryHash: String       // SHA-256 of previous entry
    entryHash: String               // SHA-256(this entry without hash fields)
    signature: String               // signed with Android Keystore key
}
```

---

## 14. Policy Engine (Revised)

### Policy actions (expanded)
| Action | Description |
|---|---|
| `LOG_ONLY` | Record the event, allow execution. Used for monitoring without interference. |
| `FLAG_AND_NOTIFY` | Allow execution, show user notification. |
| `FLAG_AND_BLOCK` | Block the specific call. Agent receives structured error. |
| `FLAG_AND_SUSPEND` | Pause session, show user dialog. |
| `FLAG_AND_TERMINATE` | End session immediately. |
| `REQUIRE_CONSENT` | Pause and show consent dialog. Proceed only if user approves. |
| `REDACT_RESULT` | Execute the call but redact sensitive fields from the result returned to the agent. |

### Policy source hierarchy
Priority order (highest to lowest):
1. OEM system policy (cannot be overridden by app)
2. Enterprise MDM policy (cannot be overridden by user)
3. Developer-defined policy (can be overridden by user for some rules)
4. User preferences (applies on top of developer policy)

### Example policy rules

```
// Rule: Calendar access always requires consent
PolicyRule(
    condition = ToolCategory(CALENDAR),
    action = REQUIRE_CONSENT,
    source = DEVELOPER
)

// Rule: No network access for agents in this app (local-only guarantee)
PolicyRule(
    condition = NetworkAccess(any),
    action = FLAG_AND_BLOCK,
    source = DEVELOPER
)

// Rule: Log all HIGH sensitivity tool calls for enterprise audit
PolicyRule(
    condition = Sensitivity(HIGH) AND ExecutionContext(ANY),
    action = LOG_ONLY,   // in addition to normal flow
    source = ENTERPRISE
)

// Rule: Manifest mismatch is a hard block
PolicyRule(
    condition = MismatchType(UNDECLARED_CAPABILITY),
    action = FLAG_AND_TERMINATE,
    source = DEVELOPER
)
```

---

## 15. Agent Execution Pipeline (Revised)

```
Goal
  ↓
Manifest Check (is agent registered? is manifest valid and unchanged?)
  ↓
Session Start → AgentSessionStarted event logged
  ↓
Plan Declared → AgentPlanDeclared event logged (stores baseline)
  ↓
Step Start → AgentStepStarted event logged
  ↓
Tool Request → Tool ID checked against declared plan + capability manifest
  ↓
Mismatch Detection → If tool not in plan or manifest → enforcement action
  ↓
Policy Evaluation → Rules evaluated in priority order
  ↓
[If REQUIRE_CONSENT] → Consent Dialog shown → UserConsentCaptured logged
  ↓
[If blocked] → Tool returns AgentGuardException → step marked BLOCKED → log
  ↓
[If allowed] → Tool Executes → ToolInvocationStarted logged
  ↓
Network Check → If tool makes network call, NetworkInterceptor evaluates
  ↓
Tool Completes → ToolInvocationCompleted logged (duration, records returned)
  ↓
Result → [If REDACT_RESULT policy] → fields redacted before returning to agent
  ↓
[On failure] → Rollback hook called (tool-specific, optional) → step marked FAILED
  ↓
Step Completion → AgentStepCompleted logged
  ↓
Explanation Generated → 3-level explanation stored (user / operational / technical)
  ↓
[Repeat for next step]
  ↓
Session End → AgentSessionCompleted logged → session summary computed
```

---

## 16. Developer Integration Model

### Minimum required integration (Phase 1)

```kotlin
// 1. Register agent at app start (once)
val agent = AgentGuard.registerAgent(
    AgentRegistration(
        agentId = "productivity-assistant-v1",
        name = "Productivity Assistant",
        version = "1.0.0",
        publisher = "YourCompany",
        capabilityManifest = AgentCapabilityManifest(
            declaredTools = listOf("readCalendar", "readTasks", "createNote"),
            networkAccess = false,
            backgroundExecution = false
        )
    )
)

// 2. Start session
val session = agent.startSession(goal = "Prepare my meeting brief")

// 3. Declare plan (recommended but optional)
session.declarePlan(listOf(
    StepDescriptor("Read calendar", toolId = "readCalendar"),
    StepDescriptor("Read task list", toolId = "readTasks"),
    StepDescriptor("Generate summary", toolId = null),  // LLM inference, no tool
    StepDescriptor("Save note", toolId = "createNote")
))

// 4. Execute steps using SDK-wrapped tools
session.step("Read calendar events") {
    val events = tools.readCalendar(date = today)
    // SDK intercepts this call, checks policy, may show consent dialog
    events
}

// 5. End session
session.complete(outputSummary = "Meeting brief created: 3 items")
```

### Using the UI Kit

```kotlin
// Show consent dialog (SDK handles this automatically, but can be customized)
AgentGuardUI.showConsentDialog(
    context = this,
    session = session,
    onDecision = { decision -> /* handle */ }
)

// Embed execution timeline in your UI
AgentGuardUI.ExecutionTimeline(session = session)  // Compose component

// Show session history
AgentGuardUI.SessionHistoryView(agentId = "productivity-assistant-v1")
```

---

## 17. Build-Time Instrumentation (Phase 2)

### Gradle plugin approach

```groovy
// build.gradle
plugins {
    id "io.agentguard.instrumentation" version "2.0.0"
}

agentGuard {
    enabled = true
    interceptors = ["ContentResolver", "SharedPreferences", "Room", "File", "Network"]
    excludePackages = ["com.yourapp.thirdparty"]  // exclude third-party libs
    reportOnlyMode = false  // if true: log but don't block (for migration)
}
```

### How it works
- Plugin applies ASM bytecode transformation at compile time
- Target call sites: `context.contentResolver.query(...)`, `prefs.getString(...)`, etc.
- Each call site is wrapped with: session-context lookup → report to SDK if active session → proceed
- Zero overhead if no active session
- Minimal overhead (single map lookup) if session is active

### Incremental adoption
- Developers can enable `reportOnlyMode = true` first to see what would be intercepted
- Then switch to full enforcement once they've reviewed the reports

---

## 18. Developer Experience and Tooling

### Debug mode
```kotlin
AgentGuard.configure {
    debugMode = BuildConfig.DEBUG  // enables verbose logging
    debugOverlay = true            // floating overlay showing live events in-app
}
```

Debug mode enables:
- Full logcat output of all SDK events (tagged `AgentGuard`)
- In-app floating overlay (like LeakCanary) showing live event stream
- Console warnings when tools are called outside a session (potential integration mistake)
- Mismatch alerts even for `LOG_ONLY` policy (highlighted in logcat)

### Testing utilities

```kotlin
// Mock tool for testing
val mockCalendar = MockTool("readCalendar") {
    CalendarResult(events = listOf(FakeEvent("Meeting at 3pm")))
}

// Simulate policy decisions
AgentGuardTest.simulatePolicy(PolicyDecision.BLOCK, toolId = "readCalendar")

// Replay a past session
val replay = AgentGuardTest.replaySession(sessionId = "abc-123")
replay.step(2).assertToolCalled("readCalendar")
replay.step(2).assertConsentShown()

// Verify audit log integrity
AuditLogVerifier.verify(sessionId = "abc-123")
    .assertChainIntact()
    .assertAllStepsLogged()
```

### Performance budget

Each SDK interceptor must meet these targets:
- Policy evaluation: < 2ms per tool call
- Consent dialog display: < 100ms from request to visible
- Audit log write: < 5ms per entry (async, non-blocking to caller)
- Hash chain computation: < 1ms per entry

These are enforced in CI via benchmark tests. If a change causes a regression beyond 20%, the build fails.

---

## 19. OEM and Enterprise Integration

### OEM integration
- OEMs can embed AgentGuard policy configuration in their system image
- System policies are read from a protected location (`/system/etc/agentguard/policy.json`)
- OEM policies have highest priority — no app can override them
- OEMs can customize: default consent behavior, blocked resource categories, required audit log retention period
- AgentGuard provides an AIDL interface for system-level policy queries (Phase 2)

### Enterprise / MDM integration
- Enterprise admins deploy policy JSON via MDM (Mobile Device Management)
- Policy is fetched from a configurable endpoint and cached on-device
- Policy updates apply to new sessions immediately; active sessions are not interrupted
- Enterprise module adds: remote audit log export, SIEM connectors, compliance reports

### Agent certification (Phase 3)
- Agents can be submitted to an AgentGuard certification process
- Certification verifies: accurate capability manifest, no undeclared API usage, passing test suite
- Certified agents receive a trust score visible to users
- OEMs can require certification as a prerequisite for using certain APIs

---

## 20. MVP Scope (Phase 1)

### In scope
- [x] Agent registration with capability manifest
- [x] Session lifecycle (start, steps, end)
- [x] Declared plan vs actual execution comparison
- [x] SDK-wrapped tool registry (calendar, tasks, notes, files)
- [x] Policy engine (developer-defined rules)
- [x] Consent UI (dialog, allow once / session / always / deny)
- [x] Consent timeout (30s default, deny on timeout)
- [x] Consent revocation (mid-session)
- [x] 3-level explanation engine (user / operational / technical)
- [x] Live execution timeline view
- [x] Session history view
- [x] Hash-chained append-only audit log
- [x] Android Keystore log signing
- [x] OkHttp network interceptor
- [x] Network access policy (block if undeclared)
- [x] Mismatch detection with enforcement actions
- [x] Background execution with deferred consent
- [x] Debug mode and logcat integration
- [x] Testing utilities (mock tools, policy simulation)
- [x] LLM adapter for MediaPipe + ONNX (most common on Android)
- [x] Performance benchmarks in CI

### Out of scope for Phase 1
- [ ] Gradle bytecode instrumentation plugin
- [ ] Enterprise MDM policy sync
- [ ] Remote audit log export
- [ ] Multi-agent session hierarchy
- [ ] Agent-to-agent communication
- [ ] Agent certification / trust scoring
- [ ] Cross-app function calling
- [ ] OEM system-level policy
- [ ] llama.cpp and Ollama adapters

### Demo app
The reference demo is a **Productivity Assistant** inside a single host app:
1. User opens app, starts assistant
2. Assistant declares goal: "Prepare my meeting brief"
3. SDK shows consent dialog: "This agent wants to access your calendar and tasks"
4. User approves
5. Agent reads calendar, reads tasks, generates summary via local LLM, saves note
6. Timeline shows live progress
7. User taps any step to see explanation
8. Session history shows complete audit trail
9. User can revoke and see the session suspended

---

## 21. Ecosystem Roadmap

### Phase 1 — Governance Foundation (current)
Single-app AI agent governance. Trust-based. MVP scope above.
**Target:** Individual developers building AI-powered Android apps.

### Phase 2 — Enforcement Layer
- Gradle bytecode instrumentation plugin
- Network monitoring (full HTTP interception)
- Enterprise MDM integration
- Audit log export and SIEM connectors
- Multi-agent session hierarchy
- LLM adapter expansion (llama.cpp, Ollama)
**Target:** Enterprises deploying AI agents in managed environments.

### Phase 3 — Ecosystem Platform
- Cross-app function calling (apps expose governed functions to agents)
- Agent-to-agent communication governance
- Agent certification and trust marketplace
- OEM system-level integration (AIDL, system policy)
- Private model runtime ecosystem integration
**Target:** OEMs shipping AI-enabled Android devices with trust guarantees.

### Phase 4 — Standards Contribution
- Contribute governance patterns to Android open standards discussions
- Work with OEM partners on platform-level agent governance APIs
- Publish the AgentGuard protocol as an open specification
**Target:** Industry standardization.

---

## 22. What AgentGuard Cannot Do

This section must appear in all public documentation.

AgentGuard **cannot**:
- Govern agents that bypass SDK integration (Phase 1)
- Prevent a malicious developer from excluding the SDK
- Operate on rooted devices where Android security is compromised
- Roll back data that was already read or written before consent was revoked
- Read or analyze LLM reasoning, prompts, or model weights
- Prevent data exfiltration via JNI native code (Phase 2 Gradle plugin covers managed code only)
- Guarantee log integrity if the device is rooted (Keystore can be bypassed on rooted devices)
- Monitor agents in other apps (Phase 1 is single-app only)

AgentGuard **can**:
- Make honest, well-intentioned AI agents fully transparent and auditable
- Remove the governance burden from developers who want to do the right thing
- Provide users with visibility and control over what agents do on their behalf
- Give enterprises a compliant, auditable foundation for deploying AI in managed apps
- Give OEMs a governance layer they can embed in their platform

> "AgentGuard is not a cage for AI agents. It is a flight recorder, control panel, and accountability framework for agents that operate with integrity."

---

*This document is a living specification. Sections will be updated as implementation decisions are made.*
