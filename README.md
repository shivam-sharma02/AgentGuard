# AgentGuard

**Android AI Agent Governance SDK**

> Make AI agents accountable on Android.

---

## What is AgentGuard?

AgentGuard is an Android SDK that acts as a **governance, transparency, and audit layer** between AI agents and the Android environment.

It is designed for developers, enterprises, and OEMs who build or ship AI-powered Android applications and need:

- Users to see what an AI agent is doing — before it happens
- Every agent action to be logged in a tamper-evident audit trail
- Policy-based control over what agents can and cannot do
- A reusable consent and explainability UX they do not have to build themselves

---

## The Core Idea

AI agents running inside Android apps can access files, read calendar data, call contacts, make network requests, and perform actions automatically. There is currently no standard way to make this behavior transparent, auditable, or controllable by the user.

AgentGuard solves this by inserting a **governed execution layer** between the agent and the Android platform.

```
┌─────────────────────────────────────────┐
│              Host App                   │
│                                         │
│   AI Agent (LLM + tool calls)           │
│         │                               │
│   ┌─────▼──────────────────────────┐    │
│   │        AgentGuard SDK          │    │
│   │                                │    │
│   │  Policy Engine                 │    │
│   │  Consent Manager               │    │
│   │  Tool Gateway                  │    │
│   │  Mismatch Detector             │    │
│   │  Audit Log (hash-chained)      │    │
│   │  Explainability Engine         │    │
│   └────────────┬───────────────────┘    │
│                │                        │
└────────────────┼────────────────────────┘
                 │
        Android Platform APIs
```

---

## Trust Model

**AgentGuard governs cooperative agents.**

It is a transparency and governance layer for agents that integrate honestly. It is **not a security sandbox** for malicious code, and it does not claim to be.

In Phase 1, the SDK works when the developer integrates it correctly. In Phase 2, a Gradle bytecode instrumentation plugin extends coverage to agents that make direct Android API calls, reducing reliance on voluntary cooperation.

This is documented explicitly so there are no false security promises.

---

## What it Does

| Capability | Description |
|---|---|
| Consent UI | Shows users what data the agent wants to access before it happens |
| Policy Engine | Developer-defined (and enterprise-defined) rules for what agents can do |
| Mismatch Detection | Compares declared intent against actual tool execution |
| Audit Log | Tamper-evident, hash-chained, Android Keystore-signed log of every agent action |
| Explainability | 3-level explanation (user-friendly / operational / technical) for every step |
| Network Governance | Intercepts and evaluates outbound HTTP calls |
| Session History | Users can review all past agent sessions |
| Background Execution | Governed background agents with deferred consent |
| LLM Adapters | Integration hooks for MediaPipe, ONNX Runtime, llama.cpp, Ollama |

---

## SDK Modules

| Module | Purpose |
|---|---|
| `agentguard-core` | Session lifecycle, event bus, policy engine, audit log, consent manager |
| `agentguard-tools` | Standard SDK-wrapped tool functions (calendar, contacts, files, tasks, notes) |
| `agentguard-ui` | Pre-built consent dialogs, execution timeline, session history, step detail views |
| `agentguard-llm` | Adapters for local LLM runtimes |
| `agentguard-network` | OkHttp interceptor, network policy evaluation |
| `agentguard-gradle` | Build-time bytecode instrumentation plugin (Phase 2) |
| `agentguard-enterprise` | MDM policy sync, audit log export, SIEM connectors (Phase 2) |
| `agentguard-test` | Mock tools, policy simulation, session replay, audit log verification |

---

## Minimum Integration

```kotlin
// Register agent (once, at app start)
val agent = AgentGuard.registerAgent(
    AgentRegistration(
        agentId = "my-agent-v1",
        name = "My Assistant",
        version = "1.0.0",
        publisher = "Your Company",
        capabilityManifest = AgentCapabilityManifest(
            declaredTools = listOf("readCalendar", "readTasks", "createNote"),
            networkAccess = false,
            backgroundExecution = false
        )
    )
)

// Start a session
val session = agent.startSession(goal = "Prepare my meeting brief")

// Execute steps using SDK-wrapped tools
session.step("Read calendar events") {
    tools.readCalendar(date = today)
}

// End session
session.complete(outputSummary = "Meeting brief created")
```

---

## What AgentGuard Cannot Do

We believe honest boundaries are more important than impressive claims.

AgentGuard cannot:
- Govern agents that bypass SDK integration (in Phase 1)
- Operate on rooted devices where Android security is compromised
- Roll back data that was already read before consent was revoked
- Read or analyze LLM reasoning, prompts, or model weights
- Monitor agents in other apps (Phase 1 is single-app only)

See [PLAN.md](./PLAN.md) for the full threat model.

---

## Roadmap

| Phase | Focus | Target |
|---|---|---|
| **Phase 1 (current)** | Governance foundation — single-app, cooperative agents | App developers |
| **Phase 2** | Enforcement layer — Gradle plugin, enterprise MDM, network monitoring | Enterprises |
| **Phase 3** | Ecosystem platform — cross-app function calling, agent certification, OEM integration | OEMs + platform |
| **Phase 4** | Standards contribution — open specification, Android platform collaboration | Industry |

---

## Target Audience

- **Android developers** building AI-powered apps who want a governance layer without building it themselves
- **Enterprises** deploying AI agents in managed Android environments who need audit trails and policy control
- **OEMs** shipping Android devices with on-device AI who need a platform-level governance foundation

---

## Project Status

**Phase 1 — Planning (current)**

The full technical specification is in [PLAN.md](./PLAN.md).

Implementation has not started. The plan is being finalized before development begins.

---

## Design Principles

- **Local-first** — all governance operates on-device; no data sent to cloud
- **Transparent** — users always know what is happening and why
- **Explainable** — every action has a human-readable explanation at three levels of detail
- **Auditable** — tamper-evident logs that can be verified independently
- **Honest** — clear about what the SDK can and cannot do
- **Developer-friendly** — minimize integration burden for developers doing the right thing

---

## License

TBD

---

*For the full technical plan including architecture, data model, threat model, and roadmap, see [PLAN.md](./PLAN.md).*
