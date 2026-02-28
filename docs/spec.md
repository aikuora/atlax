# ATLAX SDK — Technical Specification v0.1

> **Framework for building Agent-First Operating Systems**
> License: MIT | Primary Language: TypeScript | Platforms: Web (PWA/SPA) + Mobile (React Native/Expo)

---

## Table of Contents

1. [Vision and Philosophy](#1-vision-and-philosophy)
2. [High-Level Architecture](#2-high-level-architecture)
3. [Agent Model](#3-agent-model)
4. [AXP: Atlax Exchange Protocol](#4-axp-atlax-exchange-protocol)
5. [Memory System](#5-memory-system)
6. [LLM Router](#6-llm-router)
7. [Micro-Apps and Declarative Schema](#7-micro-apps-and-declarative-schema)
8. [UI Runtime and Ambient Dashboard](#8-ui-runtime-and-ambient-dashboard)
9. [Security](#9-security)
10. [Marketplace and Agent Registry](#10-marketplace-and-agent-registry)
11. [Developer Experience](#11-developer-experience)
12. [Observability](#12-observability)
13. [Implementation Roadmap](#13-implementation-roadmap)
14. [Technical Decisions and Trade-offs](#14-technical-decisions-and-trade-offs)
15. [Glossary](#15-glossary)

---

## 1. Vision and Philosophy

### 1.1 What is Atlax

Atlax is an **open-source framework** (MIT) that enables any developer to build **agent-first operating systems**: digital environments where specialized intelligent agents accompany the user throughout their day, understand their interests, activities, and preferences, and deliver dynamically generated interfaces (micro-apps) to manage their entire digital life.

Atlax **is not** an app. It's the SDK for building next-generation apps.

### 1.2 Fundamental Principles

| # | Principle | Implication |
|---|-----------|-------------|
| 1 | **Intent over navigation** | The user expresses what they want, not where to find it. No menus, just conversation + ambient surface. |
| 2 | **Agents as first-class citizens** | Each agent has its own identity, memory, permissions, and reputation. They are autonomous and auditable entities. |
| 3 | **UI as agent output** | The interface is not statically designed — it's declaratively composed at runtime from schemas the agent emits. |
| 4 | **Local-first, privacy by design** | User data lives on their device. Cloud sync is opt-in and end-to-end encrypted. |
| 5 | **Convention over configuration** | The developer defines domain and business rules. Atlax handles orchestration, memory, routing, UI, and security. |
| 6 | **Open protocols** | MCP for external tools. AXP (proprietary protocol) for inter-agent communication. No lock-in. |
| 7 | **Innate observability** | Every decision of every agent is traceable, auditable, and measurable. It's not opt-in, it's part of the core. |

### 1.3 Analogy

If Rails is "convention over configuration for web apps", **Atlax is "convention over configuration for agentic operating systems"**. The developer writes:

```typescript
// This is all you need for a finance agent
@Agent({
  name: 'finance',
  description: 'Manages user expenses, budgets, and investments',
  permissions: ['memory.read', 'memory.write', 'tools.payments', 'ui.render'],
})
class FinanceAgent {
  @Intent('track expense')
  async trackExpense(ctx: AgentContext, params: { amount: number; category: string }) {
    await ctx.memory.store('expenses', { ...params, date: Date.now() });
    return ctx.ui.render('ExpenseCard', { expense: params, balance: await this.getBalance(ctx) });
  }
}
```

Atlax handles: intent routing to the correct agent, encrypted local memory management, optimal LLM selection, micro-app rendering, execution sandboxing, audit log, and more.

---

## 2. High-Level Architecture

### 2.1 System Layers

```
┌─────────────────────────────────────────────────────────────────┐
│                     INTERACTION SURFACE                          │
│   Ambient Dashboard │ Chat/Voice │ Notifications │ Widgets       │
├─────────────────────────────────────────────────────────────────┤
│                      UI RUNTIME (Neo Renderer)                   │
│   Schema Interpreter │ Component Registry │ Layout Engine        │
├─────────────────────────────────────────────────────────────────┤
│                      AGENT ORCHESTRATOR (Cortex)                 │
│   Intent Router │ Agent Lifecycle │ Permission Gate │ AXP Bus    │
├─────────┬───────────┬──────────────┬────────────────────────────┤
│  AGENTS │  MEMORY   │  LLM ROUTER  │  TOOL GATEWAY             │
│  (User  │  (Vault)  │  (Synapse)   │  (MCP Bridge)             │
│  defined│           │              │                            │
│  agents)│  Local DB │  Multi-model │  MCP Servers               │
│         │  E2E Enc. │  Cost/Perf   │  External APIs             │
│         │  Vec+KV   │  routing     │  Code exec                 │
├─────────┴───────────┴──────────────┴────────────────────────────┤
│                      SECURITY LAYER (Bastion)                    │
│   Sandbox │ Crypto Identity │ Permissions │ Audit Log            │
├─────────────────────────────────────────────────────────────────┤
│                      PERSISTENCE (local-first)                   │
│   SQLite/OPFS │ Vector Store │ Encrypted Blob Store             │
└─────────────────────────────────────────────────────────────────┘
```

### 2.2 Internal Components (Naming)

Each Atlax subsystem has an internal name for easier communication:

| Component | Internal Name | Responsibility |
|-----------|---------------|----------------|
| UI Runtime | **Neo** | Interprets schemas, renders micro-apps, manages dashboard layout |
| Agent Orchestrator | **Cortex** | Intent routing, agent lifecycle, AXP message bus |
| Memory System | **Vault** | Encrypted local storage, episodic/semantic/working memory |
| LLM Router | **Synapse** | Model selection, cost/complexity routing, fallback chains |
| Tool Gateway | **Bridge** | MCP adapter, tool registry, sandboxed execution |
| Security Layer | **Bastion** | Cryptographic identity, permissions, sandbox, immutable audit log |
| Marketplace Client | **Zion** | Agent discovery, installation, and verification from the marketplace |

### 2.3 Typical Interaction Flow

```
User: "How much did I spend at restaurants this month?"

1. [Neo]     → Captures input (text/voice) from the ambient dashboard
2. [Cortex]  → Analyzes intent → detects "finance" domain → selects FinanceAgent
3. [Synapse] → Chooses an economical model (simple query task)
4. [Cortex]  → Verifies FinanceAgent permissions: ✓ memory.read
5. [Vault]   → FinanceAgent queries local memory: month's expenses with category="restaurants"
6. [Bastion] → Logs the action: query(expenses, {month: current, category: restaurant})
7. [Agent]   → Processes data, generates response schema
8. [Neo]     → Renders micro-app: card with expense chart + total
9. [Neo]     → Inserts micro-app into the user's ambient dashboard
```

---

## 3. Agent Model

### 3.1 Agent Definition

In Atlax, an **agent** is an autonomous entity with:

| Property | Description |
|----------|-------------|
| **Identity** | Ed25519 key pair. The agent signs every action. Cryptographically verifiable. |
| **Manifest** | Immutable document declaring: name, description, capabilities, required permissions, version, author. |
| **Own memory** | Isolated namespace within Vault. An agent CANNOT read another's memory without explicit permission. |
| **Lifecycle** | `idle → activated → executing → waiting → idle`. Managed by Cortex. |
| **Reputation** | Score based on: success rate, human interventions, errors, user feedback. |

### 3.2 Agent Anatomy (TypeScript API)

```typescript
import { Agent, Intent, Hook, Tool, Guard, AgentContext } from '@atlax/core';

@Agent({
  name: 'calendar',
  version: '1.0.0',
  description: 'Manages user events, reminders, and availability',
  permissions: [
    'memory.read:calendar.*',      // Reads its own namespace
    'memory.write:calendar.*',     // Writes to its own namespace
    'memory.read:contacts.names',  // Reads only contact names (cross-agent)
    'tools.google_calendar',       // External tool via MCP
    'ui.render',                   // Can generate micro-apps
    'notify.push',                 // Can send notifications
  ],
  triggers: {
    schedule: '0 8 * * *',        // Cron: activates every day at 8am
    events: ['morning_briefing'],  // Activates when another agent emits this event
  },
})
class CalendarAgent {

  // --- INTENTS ---
  // Atlax detects the user's intent and routes it here

  @Intent({
    patterns: ['schedule meeting', 'create event', 'set appointment'],
    params: {
      title: { type: 'string', required: true },
      date: { type: 'datetime', required: true },
      participants: { type: 'string[]', required: false },
    },
  })
  async createEvent(ctx: AgentContext, params: CreateEventParams) {
    // Guard: check for conflicts
    const conflicts = await ctx.memory.query('calendar.events', {
      date: params.date,
      overlap: true,
    });

    if (conflicts.length > 0) {
      // Ask for user confirmation via UI
      return ctx.ui.confirm({
        schema: 'ConflictResolution',
        data: { conflicts, proposed: params },
        onConfirm: () => this.executeCreateEvent(ctx, params),
        onCancel: () => ctx.ui.render('EventCancelled', {}),
      });
    }

    return this.executeCreateEvent(ctx, params);
  }

  private async executeCreateEvent(ctx: AgentContext, params: CreateEventParams) {
    // Execute external tool via MCP
    const result = await ctx.tools.invoke('google_calendar', 'createEvent', params);

    // Save to local memory
    await ctx.memory.store('calendar.events', {
      ...params,
      externalId: result.id,
      createdAt: Date.now(),
    });

    // Render confirmation
    return ctx.ui.render('EventCreated', {
      event: params,
      link: result.htmlLink,
    });
  }

  // --- HOOKS ---
  // React to system or other agent events

  @Hook('morning_briefing')
  async onMorningBriefing(ctx: AgentContext) {
    const todayEvents = await ctx.memory.query('calendar.events', {
      date: { $gte: today(), $lt: tomorrow() },
    });

    return ctx.ui.render('DailyAgenda', {
      events: todayEvents,
      freeSlots: this.calculateFreeSlots(todayEvents),
    });
  }

  // --- GUARDS ---
  // Validations that run before any action

  @Guard()
  async validatePermissions(ctx: AgentContext, action: string) {
    if (action.startsWith('memory.write') && ctx.agent.reputation < 0.5) {
      throw new AgentError('REPUTATION_TOO_LOW', 'Agent reputation below threshold');
    }
  }

  // --- INTER-AGENT COMMUNICATION ---

  @Intent({ patterns: ['do I have time for...', 'am I free...'] })
  async checkAvailability(ctx: AgentContext, params: { datetime: string }) {
    // Respond to queries from other agents about availability
    const busy = await ctx.memory.query('calendar.events', { date: params.datetime });
    return { available: busy.length === 0, slots: this.getAlternatives(busy) };
  }
}
```

### 3.3 Multi-Agent Topology

```
                    ┌──────────────┐
                    │    CORTEX    │
                    │ (Orchestrator)│
                    └──────┬───────┘
                           │ AXP Bus
          ┌────────────────┼────────────────┐
          │                │                │
    ┌─────┴─────┐   ┌─────┴─────┐   ┌─────┴─────┐
    │ Calendar  │   │  Finance  │   │  Health   │
    │  Agent    │◄──►│  Agent    │◄──►│  Agent    │
    └─────┬─────┘   └─────┬─────┘   └─────┬─────┘
          │               │               │
     ┌────┴────┐    ┌────┴────┐    ┌────┴────┐
     │   Own   │    │   Own   │    │   Own   │
     │ Memory  │    │ Memory  │    │ Memory  │
     └─────────┘    └─────────┘    └─────────┘
```

**Inter-agent communication rules:**

1. Agents **never** communicate directly. Everything goes through Cortex's AXP bus.
2. An agent can **request data** from another, but the receiving agent decides whether to share (based on permissions declared in its manifest).
3. Inter-agent communications are recorded in the audit log.
4. An agent can **emit events** that other agents listen to (decoupled pub/sub).

### 3.4 Agent Lifecycle

```
     install          activate           intent/hook/schedule
  ┌─────────┐     ┌──────────┐     ┌───────────┐     ┌─────────┐
  │INSTALLED│────►│  IDLE    │────►│ ACTIVATED │────►│EXECUTING│
  └─────────┘     └──────────┘     └───────────┘     └────┬────┘
       │               ▲                                    │
       │               │              ┌─────────┐          │
       │               └──────────────│ WAITING │◄─────────┘
       │                              │(user/tool)│         │
       │                              └─────────┘          │
       │          ┌──────────┐                             │
       └─────────►│ DISABLED │◄────────────────────────────┘
                  │(by user) │         (error/revoked)
                  └──────────┘
```

**States:**

| State | Description |
|-------|-------------|
| `INSTALLED` | Agent downloaded/registered. Manifest verified. Permissions not yet granted. |
| `IDLE` | Permissions granted. Waiting for activation by intent, hook, or schedule. |
| `ACTIVATED` | Cortex has selected this agent. Its context and memory are loaded. |
| `EXECUTING` | The agent is processing. Has access to tools and can emit UI. |
| `WAITING` | Paused: waiting for user confirmation or tool response. |
| `DISABLED` | Deactivated by the user or the system (permissions revoked, low reputation). |

---

## 4. AXP: Atlax Exchange Protocol

### 4.1 Purpose

AXP (Atlax Exchange Protocol) is Atlax's proprietary protocol for **communication between agents within the same operating system**. It does not replace MCP (which is for tools), but complements it.

**Why not use Google's A2A?** A2A is designed for communication between agents from different vendors and frameworks across the network. AXP is optimized for **intra-system** communication, local-first, with security and performance guarantees that A2A cannot offer in that context. However, Atlax can expose an A2A bridge so external agents can communicate with Atlax agents.

### 4.2 Protocol Design

#### 4.2.1 Message Types

```typescript
// All AXP messages follow this base structure
interface AXPMessage {
  id: string;                    // UUID v7 (timestamp-ordered)
  type: AXPMessageType;
  source: AgentIdentity;         // Signed with the sender agent's Ed25519
  target?: AgentIdentity | '*';  // Specific destination or broadcast
  timestamp: number;             // Unix timestamp ms
  payload: unknown;
  signature: string;             // Ed25519 signature of the complete message
  ttl?: number;                  // Time-to-live in ms (default: 30000)
  correlationId?: string;        // For request-response
}

enum AXPMessageType {
  // Request-Response (synchronous)
  REQUEST = 'axp.request',          // Request data or action from another agent
  RESPONSE = 'axp.response',        // Response to a request
  ERROR = 'axp.error',              // Processing error

  // Pub-Sub (asynchronous)
  EVENT = 'axp.event',              // Broadcast event (other agents listen if they choose)

  // Delegation
  DELEGATE = 'axp.delegate',        // Delegate a sub-task to another agent
  DELEGATE_RESULT = 'axp.delegate_result',

  // Negotiation
  CAPABILITY_QUERY = 'axp.cap_query',   // "Can you do X?"
  CAPABILITY_RESPONSE = 'axp.cap_resp', // "Yes, with these parameters"

  // Control
  HEARTBEAT = 'axp.heartbeat',      // Keep-alive for active agents
  CANCEL = 'axp.cancel',            // Cancel a pending request/delegation
}
```

#### 4.2.2 Transport

AXP operates on an **in-memory message bus** managed by Cortex. No network is involved in intra-system communication.

```typescript
// Internal bus implementation (not exposed to developers)
interface AXPBus {
  // Send point-to-point message
  send(message: AXPMessage): Promise<AXPMessage>;     // Awaits response

  // Publish broadcast event
  emit(event: AXPEvent): void;                          // Fire-and-forget

  // Subscribe to events
  on(eventType: string, handler: AXPEventHandler): Unsubscribe;

  // Query capabilities
  queryCapabilities(agentName: string): Promise<AgentManifest>;
}
```

#### 4.2.3 Request-Response Flow

```
CalendarAgent                    Cortex (AXP Bus)                FinanceAgent
     │                                │                              │
     │  AXP.REQUEST                   │                              │
     │  "What's the remaining         │                              │
     │   travel budget?"              │                              │
     │───────────────────────────────►│                              │
     │                                │  1. Verify permissions       │
     │                                │  2. Verify signature         │
     │                                │  3. Record in audit log      │
     │                                │                              │
     │                                │  AXP.REQUEST (forwarded)     │
     │                                │─────────────────────────────►│
     │                                │                              │
     │                                │          AXP.RESPONSE        │
     │                                │  { remaining: 1500, cur: USD }│
     │                                │◄─────────────────────────────│
     │                                │                              │
     │         AXP.RESPONSE           │                              │
     │◄───────────────────────────────│                              │
     │                                │                              │
```

#### 4.2.4 Delegation Pattern

When an agent needs another to execute a complete sub-task:

```typescript
// In the productivity agent
@Intent({ patterns: ['plan trip to *'] })
async planTrip(ctx: AgentContext, params: { destination: string }) {
  // Delegate flight search to the travel agent
  const flights = await ctx.axp.delegate('travel', {
    action: 'searchFlights',
    params: { destination: params.destination, dates: await this.getFreeDates(ctx) },
    timeout: 15000,
  });

  // Delegate budget check to the finance agent
  const budget = await ctx.axp.delegate('finance', {
    action: 'checkBudget',
    params: { category: 'travel', amount: flights.cheapest.price },
  });

  // Compose result
  return ctx.ui.render('TripPlan', { flights, budget, destination: params.destination });
}
```

#### 4.2.5 Serialization

AXP uses **MessagePack** for internal serialization (more compact and faster than JSON for local communication). Messages are signed before serialization.

```
┌──────────┬──────────┬──────────┬──────────────────┐
│ Header   │ Metadata │ Payload  │ Signature (64B)  │
│ (8 bytes)│ (var)    │ (var)    │ Ed25519          │
└──────────┴──────────┴──────────┴──────────────────┘
```

### 4.3 Comparison: AXP vs MCP vs A2A

| Aspect | AXP | MCP | A2A |
|--------|-----|-----|-----|
| **Purpose** | Agent ↔ Agent (intra-system) | Agent ↔ Tool | Agent ↔ Agent (inter-system, cross-vendor) |
| **Transport** | In-memory bus (local) | HTTP/SSE/stdio | HTTP + JSON-RPC |
| **Serialization** | MessagePack + Ed25519 | JSON | JSON |
| **Latency** | Sub-millisecond | Depends on server | Network-bound |
| **Security** | Cryptographic signing + Cortex permissions | OAuth/API keys | Agent Cards + mTLS |
| **Discovery** | Manifests registered in Cortex | MCP Server list | Agent Cards |
| **Scope** | Only within Atlax | Universal (tools) | Universal (agents) |

---

## 5. Memory System (Vault)

### 5.1 Principles

1. **Local-first**: All memory lives on the user's device by default.
2. **Encrypted at rest**: AES-256-GCM. The key is derived from a user passphrase via Argon2id.
3. **Namespaced**: Each agent has its own namespace. Cross-namespace access requires explicit permission.
4. **Multi-tier**: Different memory types for different needs.

### 5.2 Memory Tiers

```typescript
interface VaultConfig {
  tiers: {
    // Tier 1: Working memory (current conversation)
    working: {
      backend: 'in-memory';
      maxSize: '50MB';
      ttl: 'session';           // Lost on session close
    };

    // Tier 2: Episodic memory (specific remembered events)
    episodic: {
      backend: 'sqlite';        // SQLite via OPFS (web) or filesystem (mobile)
      encrypted: true;
      maxSize: '500MB';
      retention: '1y';          // Configurable by the user
    };

    // Tier 3: Semantic memory (general knowledge, embeddings)
    semantic: {
      backend: 'vectorstore';   // hnswlib-wasm (web) or hnswlib-node (mobile)
      encrypted: true;
      dimensions: 384;          // all-MiniLM-L6-v2 embeddings (local)
      maxVectors: 100000;
    };

    // Tier 4: Preference memory (learned configuration)
    preferences: {
      backend: 'kv';            // Encrypted key-value store
      encrypted: true;
      syncable: true;           // Can sync to cloud (opt-in)
    };
  };
}
```

### 5.3 Memory API for Agents

```typescript
// Available via ctx.memory inside any agent
interface AgentMemory {
  // --- Working Memory ---
  // Current conversation/session context
  working: {
    get(key: string): Promise<unknown>;
    set(key: string, value: unknown): Promise<void>;
    clear(): Promise<void>;
  };

  // --- Episodic Memory ---
  // Specific events and facts the agent remembers
  store(collection: string, document: Record<string, unknown>): Promise<string>;  // returns doc ID
  query(collection: string, filter: QueryFilter): Promise<Document[]>;
  update(collection: string, id: string, patch: Partial<Document>): Promise<void>;
  delete(collection: string, id: string): Promise<void>;

  // --- Semantic Memory ---
  // Similarity-based semantic search
  embed(text: string): Promise<Float32Array>;           // Generates local embedding
  semanticStore(namespace: string, text: string, metadata?: Record<string, unknown>): Promise<string>;
  semanticSearch(namespace: string, query: string, topK?: number): Promise<SemanticResult[]>;

  // --- Preferences ---
  // Key-value for learned user configuration
  preferences: {
    get(key: string): Promise<unknown>;
    set(key: string, value: unknown): Promise<void>;
    observe(key: string, handler: (value: unknown) => void): Unsubscribe;
  };

  // --- Cross-Agent (requires explicit permission) ---
  // Read data from another agent (only if permission is declared in manifest)
  foreign(agentName: string): {
    query(collection: string, filter: QueryFilter): Promise<Document[]>;
    semanticSearch(namespace: string, query: string, topK?: number): Promise<SemanticResult[]>;
  };
}
```

### 5.4 Local Embeddings Model

To maintain the local-first principle, Atlax includes an embeddings model that runs on-device:

| Platform | Model | Runtime | Dimensions |
|----------|-------|---------|------------|
| Web | all-MiniLM-L6-v2 | ONNX Runtime Web (WASM) | 384 |
| Mobile (RN) | all-MiniLM-L6-v2 | ONNX Runtime Mobile | 384 |

The model is **downloadable on demand** (~25MB). Once downloaded, all embeddings are generated locally without sending data to any server.

### 5.5 Optional Cloud Sync

```typescript
// Sync configuration (opt-in by the user)
@Agent({
  sync: {
    enabled: false,                  // User activates manually
    strategy: 'selective',           // Only syncs what the user chooses
    encryption: 'e2e',              // End-to-end: the server never sees data in the clear
    provider: 'atlax-cloud',        // Or self-hosted
    conflict: 'last-write-wins',    // Conflict strategy
  }
})
```

**E2E sync flow:**
```
Device A                          Cloud (opaque)                    Device B
     │                                │                                │
     │ encrypt(data, userKey)         │                                │
     │───────────────────────────────►│                                │
     │                                │ stores encrypted blob          │
     │                                │───────────────────────────────►│
     │                                │                   decrypt(blob, userKey)
     │                                │                                │
```

The sync server **never** has access to the user's key. It only stores encrypted blobs.

---

## 6. LLM Router (Synapse)

### 6.1 Purpose

Synapse is the component that **selects the optimal LLM** for each task, optimizing the balance between cost, latency, and quality. It's not a simple switch — it's an intelligent router with fallback strategies.

### 6.2 Configuration

```typescript
import { defineRouter } from '@atlax/synapse';

export const router = defineRouter({
  providers: {
    anthropic: {
      models: {
        'claude-sonnet': { costPer1kTokens: 0.003, latencyP50: 800, capabilities: ['reasoning', 'tools', 'vision'] },
        'claude-haiku': { costPer1kTokens: 0.0008, latencyP50: 300, capabilities: ['tools', 'classification'] },
      },
      apiKey: { env: 'ANTHROPIC_API_KEY' },  // Or stored encrypted in Vault
    },
    openai: {
      models: {
        'gpt-4o': { costPer1kTokens: 0.005, latencyP50: 900, capabilities: ['reasoning', 'tools', 'vision'] },
        'gpt-4o-mini': { costPer1kTokens: 0.0006, latencyP50: 250, capabilities: ['tools', 'classification'] },
      },
      apiKey: { env: 'OPENAI_API_KEY' },
    },
    local: {
      models: {
        'phi-3-mini': { costPer1kTokens: 0, latencyP50: 150, capabilities: ['classification', 'extraction'] },
      },
      runtime: 'webllm',  // Or 'ollama' for mobile/desktop
    },
  },

  // Routing strategies
  strategies: {
    // Simple task (classification, extraction) → local or cheap model
    simple: {
      prefer: ['local.phi-3-mini', 'openai.gpt-4o-mini', 'anthropic.claude-haiku'],
      maxCost: 0.001,
      maxLatency: 500,
    },
    // Complex task (reasoning, planning) → powerful model
    complex: {
      prefer: ['anthropic.claude-sonnet', 'openai.gpt-4o'],
      maxLatency: 5000,
    },
    // Vision task → only models with 'vision' capability
    vision: {
      require: ['vision'],
      prefer: ['anthropic.claude-sonnet', 'openai.gpt-4o'],
    },
  },

  // Global fallback chain
  fallback: {
    maxRetries: 2,
    backoff: 'exponential',
    onAllFail: 'queue',  // Queue the task for later retry
  },

  // Budget
  budget: {
    daily: 5.00,          // Max USD per day
    monthly: 100.00,
    alertAt: 0.8,         // Alert at 80% of budget
    onExhausted: 'degrade-to-local',  // Use only local models if budget is exhausted
  },
});
```

### 6.3 Automatic Complexity Classification

Synapse analyzes each request before sending it to an LLM and classifies it:

```typescript
// Synapse internal classifier (not exposed to developers)
interface TaskClassification {
  complexity: 'trivial' | 'simple' | 'moderate' | 'complex';
  estimatedTokens: { input: number; output: number };
  requiredCapabilities: string[];
  strategy: string;  // Name of the strategy to use
}

// Heuristic classification examples:
// - "What time is it?" → trivial (no LLM needed, respond directly)
// - "Classify this expense" → simple (local model)
// - "Summary of my monthly expenses" → moderate (mid-tier model)
// - "Analyze my spending patterns and suggest a budget" → complex (powerful model)
```

### 6.4 API for Agents

```typescript
// Agents do NOT choose the model directly — Synapse does
// But they can give hints about complexity

// Inside an agent:
async processTask(ctx: AgentContext) {
  // Option 1: Let Synapse decide (recommended)
  const result = await ctx.llm.complete({
    messages: [{ role: 'user', content: 'Classify this expense: "Uber $15"' }],
    tools: [this.classifyTool],
  });

  // Option 2: Give complexity hint
  const analysis = await ctx.llm.complete({
    messages: [{ role: 'user', content: 'Analyze spending patterns...' }],
    hint: 'complex',  // Synapse will prioritize powerful models
    stream: true,      // Stream the response
  });
}
```

---

## 7. Micro-Apps and Declarative Schema

### 7.1 Concept

A **micro-app** is a unit of interface dynamically generated by an agent. It's not a pre-designed "screen" — it's a declarative schema that describes what to show and how, which the Neo Renderer converts into native components (React on web, React Native on mobile).

### 7.2 The AXUI Schema (Atlax UI Schema)

AXUI is a declarative JSON format that describes user interfaces. Inspired by JSON Schema + SwiftUI declarative + Server-Driven UI patterns.

#### 7.2.1 Base Structure

```typescript
interface AXUISchema {
  // Metadata
  $schema: 'axui/1.0';
  id: string;                      // Unique identifier for this micro-app
  agent: string;                   // Agent that generated it
  intent: string;                  // Intent that originated it
  timestamp: number;

  // Layout
  surface: 'card' | 'sheet' | 'fullscreen' | 'widget' | 'notification' | 'inline';
  priority: 'low' | 'medium' | 'high' | 'urgent';

  // Content
  root: AXUINode;

  // Interactivity
  actions?: AXUIAction[];

  // Local micro-app state
  state?: Record<string, unknown>;

  // Real-time data binding
  bindings?: AXUIBinding[];

  // Lifecycle
  lifecycle?: {
    ttl?: number;                  // Auto-dismiss after N ms
    refresh?: number;              // Auto-refresh every N ms
    persistent?: boolean;          // Survives session change
  };
}
```

#### 7.2.2 UI Nodes (AXUINode)

```typescript
type AXUINode =
  | AXUIText
  | AXUIStack
  | AXUICard
  | AXUIChart
  | AXUIForm
  | AXUITable
  | AXUIImage
  | AXUIMap
  | AXUIList
  | AXUIProgress
  | AXUIMetric
  | AXUICodeBlock
  | AXUIMarkdown
  | AXUIDivider
  | AXUISpacer
  | AXUIConditional
  | AXUILoop;

// Example: Text node
interface AXUIText {
  type: 'text';
  content: string;
  variant?: 'title' | 'subtitle' | 'body' | 'caption' | 'label' | 'code';
  style?: AXUIStyle;
  actions?: AXUIAction[];   // Tap/click actions
}

// Example: Stack (layout container)
interface AXUIStack {
  type: 'stack';
  direction: 'horizontal' | 'vertical';
  gap?: number;
  align?: 'start' | 'center' | 'end' | 'stretch';
  justify?: 'start' | 'center' | 'end' | 'between' | 'around';
  children: AXUINode[];
  style?: AXUIStyle;
}

// Example: Chart
interface AXUIChart {
  type: 'chart';
  chartType: 'line' | 'bar' | 'pie' | 'donut' | 'area' | 'scatter';
  data: {
    labels?: string[];
    datasets: Array<{
      label: string;
      values: number[];
      color?: string;
    }>;
  };
  interactive?: boolean;
  style?: AXUIStyle;
}

// Example: Form
interface AXUIForm {
  type: 'form';
  fields: Array<{
    name: string;
    label: string;
    inputType: 'text' | 'number' | 'date' | 'select' | 'toggle' | 'slider' | 'textarea';
    value?: unknown;
    options?: Array<{ label: string; value: string }>;  // For select
    validation?: {
      required?: boolean;
      min?: number;
      max?: number;
      pattern?: string;
    };
  }>;
  submitAction: AXUIAction;
  style?: AXUIStyle;
}

// Example: Conditional rendering
interface AXUIConditional {
  type: 'conditional';
  condition: string;       // Expression evaluated against state: "state.balance > 1000"
  then: AXUINode;
  else?: AXUINode;
}

// Example: Loop
interface AXUILoop {
  type: 'loop';
  items: string;           // Path to array in state: "state.expenses"
  as: string;              // Variable name: "expense"
  template: AXUINode;      // Template repeated for each item
  empty?: AXUINode;        // UI when the array is empty
}
```

#### 7.2.3 Actions

```typescript
interface AXUIAction {
  type:
    | 'agent.invoke'       // Invoke an agent's intent
    | 'agent.confirm'      // Confirm a pending action
    | 'agent.cancel'       // Cancel a pending action
    | 'navigate'           // Navigate to another view/micro-app
    | 'link'               // Open external URL
    | 'copy'               // Copy text to clipboard
    | 'share'              // Native OS share
    | 'state.update'       // Update micro-app local state
    | 'dismiss';           // Close the micro-app

  // For agent.invoke:
  agent?: string;
  intent?: string;
  params?: Record<string, unknown>;

  // For navigate:
  target?: string;

  // For link:
  url?: string;

  // For state.update:
  path?: string;
  value?: unknown;

  // Visual
  label: string;
  icon?: string;
  variant?: 'primary' | 'secondary' | 'danger' | 'ghost';
  confirm?: {
    title: string;
    message: string;
  };
}
```

#### 7.2.4 Styles

```typescript
interface AXUIStyle {
  // Spacing
  padding?: number | { top?: number; right?: number; bottom?: number; left?: number };
  margin?: number | { top?: number; right?: number; bottom?: number; left?: number };

  // Sizing
  width?: number | 'full' | 'auto';
  height?: number | 'auto';
  minHeight?: number;
  maxWidth?: number;

  // Colors (semantic tokens, not raw values)
  background?: SemanticColor;
  foreground?: SemanticColor;
  border?: { color: SemanticColor; width: number; radius?: number };

  // Typography
  fontSize?: 'xs' | 'sm' | 'md' | 'lg' | 'xl' | '2xl';
  fontWeight?: 'normal' | 'medium' | 'semibold' | 'bold';

  // Effects
  shadow?: 'none' | 'sm' | 'md' | 'lg';
  opacity?: number;
}

// Colors are semantic — the theme resolves them
type SemanticColor =
  | 'surface' | 'surface.secondary' | 'surface.elevated'
  | 'text' | 'text.secondary' | 'text.muted'
  | 'primary' | 'primary.soft'
  | 'success' | 'warning' | 'danger' | 'info'
  | 'accent';
```

### 7.3 Complete Example: Expense Summary Micro-App

```json
{
  "$schema": "axui/1.0",
  "id": "expense-summary-2026-02",
  "agent": "finance",
  "intent": "monthly_summary",
  "timestamp": 1740700800000,
  "surface": "card",
  "priority": "medium",

  "state": {
    "month": "February 2026",
    "total": 2847.50,
    "budget": 3500,
    "categories": [
      { "name": "Restaurants", "amount": 680, "color": "primary" },
      { "name": "Transport", "amount": 420, "color": "info" },
      { "name": "Entertainment", "amount": 350, "color": "accent" },
      { "name": "Groceries", "amount": 890, "color": "success" },
      { "name": "Other", "amount": 507.50, "color": "text.secondary" }
    ],
    "trend": "down",
    "vsPrevMonth": -12.3
  },

  "root": {
    "type": "stack",
    "direction": "vertical",
    "gap": 16,
    "children": [
      {
        "type": "stack",
        "direction": "horizontal",
        "justify": "between",
        "align": "center",
        "children": [
          { "type": "text", "content": "Expenses for {state.month}", "variant": "title" },
          {
            "type": "text",
            "content": "{state.vsPrevMonth}% vs previous month",
            "variant": "caption",
            "style": { "foreground": "success" }
          }
        ]
      },

      {
        "type": "stack",
        "direction": "horizontal",
        "gap": 24,
        "children": [
          {
            "type": "metric",
            "label": "Total spent",
            "value": "${state.total}",
            "format": "currency",
            "size": "large"
          },
          {
            "type": "progress",
            "value": "{state.total}",
            "max": "{state.budget}",
            "label": "of budget",
            "variant": "circular"
          }
        ]
      },

      {
        "type": "chart",
        "chartType": "donut",
        "data": {
          "labels": "{state.categories.*.name}",
          "datasets": [{
            "label": "Expenses by category",
            "values": "{state.categories.*.amount}"
          }]
        },
        "interactive": true
      },

      {
        "type": "stack",
        "direction": "horizontal",
        "gap": 8,
        "justify": "end",
        "children": [
          {
            "type": "text",
            "content": "View details",
            "variant": "label",
            "style": { "foreground": "primary" },
            "actions": [{
              "type": "agent.invoke",
              "agent": "finance",
              "intent": "expense_detail",
              "params": { "month": "{state.month}" },
              "label": "View details"
            }]
          },
          {
            "type": "text",
            "content": "Adjust budget",
            "variant": "label",
            "style": { "foreground": "primary" },
            "actions": [{
              "type": "agent.invoke",
              "agent": "finance",
              "intent": "edit_budget",
              "params": {},
              "label": "Adjust budget"
            }]
          }
        ]
      }
    ]
  }
}
```

### 7.4 Neo Renderer: From Schema to Component

The Neo Renderer is the component that converts AXUI schemas into React (web) and React Native (mobile) components.

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│  AXUI Schema │────►│     Neo      │────►│   React /    │
│  (JSON)      │     │   Renderer   │     │   React      │
│              │     │              │     │   Native     │
│              │     │  - Validate  │     │   Component  │
│              │     │  - Resolve   │     │              │
│              │     │    theme     │     │              │
│              │     │  - Map nodes │     │              │
│              │     │  - Bind data │     │              │
│              │     │  - Wire      │     │              │
│              │     │    actions   │     │              │
└──────────────┘     └──────────────┘     └──────────────┘
```

**Custom component registration:**

```typescript
import { registerComponent } from '@atlax/neo';

// Developers can register custom components for AXUI
registerComponent('crypto-ticker', {
  // Mapping from AXUI schema to React props
  propsFromSchema: (node: AXUINode) => ({
    symbol: node.symbol,
    refreshInterval: node.refresh || 5000,
  }),
  // React/RN component
  web: React.lazy(() => import('./components/CryptoTicker.web')),
  mobile: React.lazy(() => import('./components/CryptoTicker.native')),
});
```

---

## 8. UI Runtime and Ambient Dashboard

### 8.1 Ambient Dashboard

The dashboard is the **primary surface** of the agent-first OS. It's not a list of apps — it's a living space that adapts to the user's context.

```
┌─────────────────────────────────────────────────────────┐
│  🕐 8:32 AM | Good morning, Carlos           [🎤] [⚙️]  │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  ┌─────────────────────┐  ┌─────────────────────┐      │
│  │  📅 Today            │  │  💰 Feb Expenses    │      │
│  │  ─────────────────  │  │  ─────────────────  │      │
│  │  9:00 Standup       │  │  $2,847 / $3,500   │      │
│  │  11:30 Dentist      │  │  [====------] 81%  │      │
│  │  14:00 Lunch María  │  │                     │      │
│  │  [+ Add event]      │  │  [View details →]  │      │
│  └─────────────────────┘  └─────────────────────┘      │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │  🏃 Activity                                     │   │
│  │  You have 6,234 steps today. 3,766 to go.       │   │
│  │  [████████░░░░░] 62%                             │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │  💡 Finance agent suggestion                     │   │
│  │  "It's been 3 days since you logged expenses.   │   │
│  │   Want me to check your bank transactions?"     │   │
│  │  [Yes, check]  [Not now]  [Don't ask again]    │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
├─────────────────────────────────────────────────────────┤
│  [💬 Type or speak...]                                   │
└─────────────────────────────────────────────────────────┘
```

### 8.2 Dashboard Layout Engine

The dashboard uses an **adaptive grid** system where micro-apps are positioned based on priority, context, and user preferences.

```typescript
import { defineDashboard } from '@atlax/neo';

export const dashboard = defineDashboard({
  // Base layout
  layout: {
    type: 'masonry',           // masonry | grid | stack
    columns: { mobile: 1, tablet: 2, desktop: 3 },
    gap: 16,
  },

  // Always-visible header
  header: {
    greeting: true,            // Contextual greeting (time of day + name)
    voiceInput: true,          // Voice input button
    settings: true,            // Settings access
  },

  // Always-visible footer
  footer: {
    chatInput: true,           // Text input as secondary channel
    quickActions: true,        // User's frequent actions
  },

  // Micro-app prioritization rules
  prioritization: {
    rules: [
      // Urgent always at top
      { match: { priority: 'urgent' }, position: 'top', animate: 'pulse' },
      // Calendar in the morning, finance in the evening
      { match: { agent: 'calendar' }, boost: { hours: [6, 12], factor: 2.0 } },
      { match: { agent: 'finance' }, boost: { hours: [18, 23], factor: 1.5 } },
      // What the user uses most, higher up
      { match: { any: true }, boost: { source: 'user-interaction-frequency' } },
    ],
  },

  // Persistent widgets (always visible as mini-cards)
  pinnedWidgets: ['calendar.today', 'health.steps', 'finance.budget'],
});
```

### 8.3 Interaction Channels

| Channel | When to use | Implementation |
|---------|-------------|----------------|
| **Dashboard** (visual) | Primary surface. Visible micro-apps and widgets. | React (web) / RN (mobile) |
| **Chat** (text) | Secondary input in footer. For complex or conversational intents. | Embedded input + streaming response |
| **Voice** | Header button. Ideal for hands-free or mobile. | Web Speech API (web) / expo-speech (mobile) |
| **Notifications** | Proactive from agent. Suggestions, alerts, reminders. | Push notifications + in-app toasts |
| **Widgets** | Persistent mini-cards. At-a-glance info (clock, steps, balance). | AXUI schema with `surface: 'widget'` |

---

## 9. Security (Bastion)

### 9.1 Non-Negotiable Principles (v1)

1. **Per-agent sandbox**: Each agent runs in an isolated context. Cannot access the DOM, filesystem, or other agents' memory without permission.
2. **Granular permissions**: Declared in manifest, granted by user, verified on every operation.
3. **E2E encryption**: All data in Vault is encrypted with AES-256-GCM. The key never leaves the device.
4. **Immutable audit log**: Every action of every agent is recorded in a signed append-only log.

### 9.2 Agent Cryptographic Identity

Each agent has an **Ed25519** key pair generated at creation or installation time.

```typescript
interface AgentIdentity {
  // Public key (agent identifier)
  publicKey: Uint8Array;         // 32 bytes Ed25519

  // Human-readable fingerprint (derived from public key)
  fingerprint: string;           // e.g.: "axag:7f3a:b2c1:d4e5:f6a7"

  // Signed manifest
  manifest: SignedManifest;

  // Trust chain
  signedBy?: AgentIdentity;      // For marketplace agents: signed by the publisher
}

interface SignedManifest {
  payload: {
    name: string;
    version: string;
    description: string;
    author: string;
    permissions: Permission[];
    capabilities: string[];
    hash: string;                // SHA-256 of the agent's code
  };
  signature: string;             // Ed25519 signature of the payload
}
```

### 9.3 Permission System

```typescript
// Permissions are granular and hierarchical
type Permission =
  // Memory
  | 'memory.read:*'                    // Read everything (system agents only)
  | `memory.read:${string}.*`          // Read its own namespace
  | `memory.read:${string}.${string}`  // Read specific collection from another agent
  | `memory.write:${string}.*`         // Write to its namespace

  // Tools
  | `tools.${string}`                  // Use specific tool via MCP

  // UI
  | 'ui.render'                        // Can generate micro-apps
  | 'ui.notify'                        // Can send notifications
  | 'ui.fullscreen'                    // Can render in fullscreen

  // Network
  | `network.${string}`               // Can make requests to specific domain
  | 'network.*'                        // Can make requests to any domain

  // Inter-agent communication
  | `axp.send:${string}`              // Can send messages to specific agent
  | 'axp.broadcast'                    // Can emit broadcast events

  // System
  | 'schedule.cron'                    // Can register scheduled tasks
  | 'clipboard.read'                   // Can read clipboard
  | 'clipboard.write'                  // Can write to clipboard
  | 'location.read'                    // Can read user location
  | 'camera.read'                      // Can access camera
  | 'microphone.read';                 // Can access microphone

// Permission flow:
// 1. Agent declares permissions in manifest
// 2. On install, user sees and approves permissions
// 3. On every operation, Bastion verifies permission at runtime
// 4. User can revoke permissions at any time
```

### 9.4 Execution Sandbox

| Platform | Sandbox Mechanism | Isolation |
|----------|-------------------|-----------|
| Web | **Web Workers** + API Proxy | Each agent runs in its own Worker. No access to the main DOM. Communicates via `postMessage` controlled by Bastion. |
| Mobile (RN) | **JSC/Hermes isolates** + controlled bridge | Isolated JS contexts per agent. Only approved APIs available. |

```typescript
// Internal example: how Bastion wraps agent execution
class AgentSandbox {
  private worker: Worker;
  private permissions: Set<Permission>;

  async execute(action: AgentAction): Promise<AgentResult> {
    // 1. Verify permission
    if (!this.permissions.has(action.requiredPermission)) {
      throw new PermissionDenied(action.requiredPermission);
    }

    // 2. Execute in sandbox
    const result = await this.worker.postMessage({
      type: 'execute',
      action,
      timeout: 30000,  // Kill after 30s
    });

    // 3. Record in audit log
    await this.auditLog.append({
      agent: this.identity,
      action: action.type,
      params: action.params,
      result: result.status,
      timestamp: Date.now(),
      signature: this.sign(result),
    });

    return result;
  }
}
```

### 9.5 Immutable Audit Log

The audit log is an **append-only** log where each entry is signed and chained (similar to a simplified blockchain).

```typescript
interface AuditEntry {
  id: string;                    // UUID v7
  previousHash: string;          // SHA-256 of the previous entry (chain)
  timestamp: number;
  agent: string;                 // Agent fingerprint
  action: string;                // What it did
  params: Record<string, unknown>; // With what parameters
  result: 'success' | 'error' | 'denied' | 'cancelled';
  metadata?: {
    llmModel?: string;           // Which model was used
    tokensUsed?: number;         // Tokens consumed
    latency?: number;            // Latency in ms
    cost?: number;               // Cost in USD
  };
  hash: string;                  // SHA-256 of the entire entry (including previousHash)
  signature: string;             // Ed25519 from the agent
}
```

The user can **export** their complete audit log and independently verify the chain's integrity.

### 9.6 Prompt Injection Protection

```typescript
// Bastion includes an input sanitizer before passing to the LLM
interface PromptGuard {
  // Sanitize user inputs before passing to the agent
  sanitize(input: string): string;

  // Detect injection attempts in tool responses
  validateToolResponse(response: unknown): ValidationResult;

  // Isolate untrusted data in the prompt
  wrapUntrusted(data: string): string;  // Wraps in delimiters the LLM recognizes
}
```

---

## 10. Marketplace and Agent Registry (Zion)

### 10.1 Marketplace Model

Zion is a **decentralized registry** (inspired by npm) with cryptographic verification and a reputation system.

```
┌──────────────────────────────────────────────┐
│                    ZION                        │
│                                              │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  │
│  │ Registry │  │ Reputation│  │ Verify   │  │
│  │          │  │  Engine   │  │  Service │  │
│  │ Manifests│  │          │  │          │  │
│  │ + Code   │  │ Scores   │  │ Signatures│ │
│  │ + Reviews│  │ Reviews  │  │ Hashes   │  │
│  └──────────┘  └──────────┘  └──────────┘  │
└──────────────────────────────────────────────┘
```

### 10.2 Publishing an Agent

```bash
# The developer publishes their agent
atlax publish ./my-agent

# Internally:
# 1. Generates SHA-256 hash of the code
# 2. Signs the manifest with the developer's key
# 3. Uploads to registry with metadata
# 4. Runs static security analysis (excessive permissions, suspicious network calls)
```

### 10.3 Installation and Verification

```typescript
// When a user installs an agent from the marketplace
async function installAgent(agentId: string) {
  // 1. Download manifest and code
  const pkg = await zion.download(agentId);

  // 2. Verify publisher signature
  const verified = await crypto.verify(pkg.manifest, pkg.signature, pkg.publisherKey);
  if (!verified) throw new SecurityError('Invalid publisher signature');

  // 3. Verify code hash
  const codeHash = await crypto.sha256(pkg.code);
  if (codeHash !== pkg.manifest.payload.hash) throw new SecurityError('Code hash mismatch');

  // 4. Show permissions to user for approval
  const approved = await ui.showPermissionDialog(pkg.manifest.payload.permissions);
  if (!approved) return;

  // 5. Generate unique key pair for this agent instance
  const identity = await crypto.generateEd25519KeyPair();

  // 6. Register in Cortex
  await cortex.registerAgent(pkg, identity);
}
```

### 10.4 Reputation System

```typescript
interface AgentReputation {
  agentId: string;

  // Automatically calculated metrics
  metrics: {
    intentSuccessRate: number;     // % of intents completed correctly
    humanInterventionRate: number; // % of times user had to correct
    errorRate: number;             // % of errors/crashes
    avgResponseTime: number;       // Average latency in ms
    uptime: number;                // % of time available
  };

  // User feedback (local, not sent to marketplace without consent)
  userFeedback: {
    rating: number;               // 1-5
    lastInteraction: number;      // timestamp
    totalInteractions: number;
  };

  // Aggregated marketplace score (anonymous, opt-in)
  marketplaceScore?: {
    downloads: number;
    avgRating: number;
    securityAudits: number;       // Number of audits passed
    verifiedPublisher: boolean;
  };

  // Combined score (0-1)
  score: number;
}

// The score affects behavior:
// score > 0.8 → Agent can act with fewer confirmations
// score 0.5-0.8 → Normal behavior
// score < 0.5 → More confirmations required, warnings to user
// score < 0.2 → Agent automatically disabled
```

---

## 11. Developer Experience

### 11.1 CLI

```bash
# Create a new Atlax project
npx create-atlax my-os

# Generated structure:
my-os/
├── atlax.config.ts          # Global configuration
├── agents/                  # Agent directory
│   └── example/
│       ├── agent.ts         # Agent definition
│       ├── intents.ts       # Intents
│       ├── tools.ts         # MCP tools
│       └── schemas/         # AXUI schemas
│           └── ExampleCard.axui.json
├── components/              # Custom React components for AXUI
│   └── CustomChart.tsx
├── themes/                  # Visual themes
│   └── default.ts
├── dashboard.ts             # Dashboard configuration
├── router.ts                # LLM router configuration
└── package.json
```

```bash
# CLI commands
atlax dev              # Dev server with hot reload
atlax build            # Production build
atlax agent create     # Scaffold a new agent
atlax agent test       # Test agents (simulate intents)
atlax agent audit      # Audit permissions and security
atlax publish          # Publish agent to Zion
atlax doctor           # Diagnose configuration issues
```

### 11.2 Global Configuration

```typescript
// atlax.config.ts
import { defineConfig } from '@atlax/core';
import { synapse } from './router';
import { dashboard } from './dashboard';

export default defineConfig({
  // OS name
  name: 'My Personal OS',
  version: '1.0.0',

  // Platforms
  platforms: {
    web: { framework: 'react', bundler: 'vite' },
    mobile: { framework: 'expo', router: 'expo-router' },
  },

  // LLM Router
  llm: synapse,

  // Dashboard
  dashboard,

  // Memory
  memory: {
    encryption: {
      algorithm: 'aes-256-gcm',
      keyDerivation: 'argon2id',
    },
    embeddings: {
      model: 'all-MiniLM-L6-v2',
      runtime: 'onnx',
    },
    sync: {
      enabled: false,  // Opt-in by the user
    },
  },

  // MCP Tools
  mcp: {
    servers: [
      { name: 'google-calendar', url: 'npx @anthropic/mcp-google-calendar' },
      { name: 'github', url: 'npx @anthropic/mcp-github' },
    ],
  },

  // Security
  security: {
    sandbox: 'webworker',  // 'webworker' | 'isolate' | 'wasm'
    auditLog: true,
    promptGuard: true,
  },

  // Theme
  theme: {
    mode: 'system',  // 'light' | 'dark' | 'system'
    colors: {
      primary: '#6366f1',
      accent: '#f59e0b',
    },
  },

  // Observability
  observability: {
    tracing: true,
    metrics: true,
    costTracking: true,
  },
});
```

### 11.3 Agent Testing

```typescript
// agents/finance/__tests__/finance.test.ts
import { testAgent, mockMemory, mockLLM, mockTool } from '@atlax/testing';
import { FinanceAgent } from '../agent';

describe('FinanceAgent', () => {
  it('should track an expense correctly', async () => {
    const { agent, ctx } = await testAgent(FinanceAgent, {
      memory: mockMemory({
        'finance.expenses': [
          { amount: 100, category: 'food', date: '2026-02-01' },
        ],
      }),
      llm: mockLLM({
        classify: () => ({ category: 'restaurant' }),
      }),
    });

    const result = await agent.handleIntent('track expense', {
      amount: 50,
      description: 'Lunch at La Trattoria',
    });

    expect(result.ui.schema).toBe('ExpenseCard');
    expect(ctx.memory.stored).toContainEqual(
      expect.objectContaining({ amount: 50, category: 'restaurant' })
    );
    expect(ctx.auditLog).toHaveLength(1);
  });

  it('should respect budget limits', async () => {
    const { agent } = await testAgent(FinanceAgent, {
      memory: mockMemory({
        'finance.budget': { monthly: 3000, spent: 2900 },
      }),
    });

    const result = await agent.handleIntent('track expense', { amount: 200 });

    // Should warn about exceeding budget
    expect(result.ui.schema).toBe('BudgetWarning');
    expect(result.ui.actions).toContainEqual(
      expect.objectContaining({ type: 'agent.confirm' })
    );
  });
});
```

---

## 12. Observability

### 12.1 End-to-End Traceability

Each interaction generates a **trace** that connects all operations:

```
Trace: usr_intent_abc123
├── [Cortex] IntentRouter.classify (2ms, local model)
├── [Cortex] AgentSelector.select → FinanceAgent (0.1ms)
├── [Synapse] ModelRouter.select → claude-haiku (0.5ms)
├── [Bastion] PermissionCheck: memory.read ✓ (0.05ms)
├── [Vault] Query: finance.expenses (3ms)
├── [Synapse] LLM.complete: 142 input tokens, 89 output tokens ($0.0002)
├── [Neo] Render: ExpenseCard (12ms)
├── [Bastion] AuditLog.append (1ms)
└── Total: 18.65ms | Cost: $0.0002
```

### 12.2 Exported Metrics

```typescript
// Atlax exports OpenTelemetry-compatible metrics
interface AtlaxMetrics {
  // Per agent
  agent_intent_total: Counter;           // Intents processed
  agent_intent_success_rate: Gauge;      // Success rate
  agent_execution_duration: Histogram;   // Execution duration
  agent_error_total: Counter;            // Errors

  // Per LLM
  llm_tokens_total: Counter;             // Tokens consumed
  llm_cost_total: Counter;               // Accumulated cost
  llm_latency: Histogram;               // Latency per model
  llm_fallback_total: Counter;           // Fallbacks triggered

  // Per memory
  vault_operations_total: Counter;       // Read/write operations
  vault_storage_bytes: Gauge;            // Space used

  // Per UI
  neo_render_duration: Histogram;        // Render time
  neo_user_interaction: Counter;         // User interactions with micro-apps
}
```

### 12.3 Developer Dashboard

Atlax includes a **development dashboard** (only in `atlax dev`) that shows in real time:

- Traces for each interaction
- Accumulated LLM costs
- State of each agent (idle/executing/error)
- Navigable audit log
- Micro-app rendering performance
- Errors and warnings

---

## 13. Implementation Roadmap

### Phase 0: Foundations (Weeks 1–6)

| Deliverable | Description | Priority |
|-------------|-------------|----------|
| `@atlax/core` | TypeScript types, `@Agent`, `@Intent`, `@Hook` decorators | P0 |
| `@atlax/vault` | Encrypted local memory (SQLite/OPFS + KV + in-memory) | P0 |
| `@atlax/bastion` | Basic permissions + Web Worker sandbox | P0 |
| `@atlax/cortex` | Basic intent router + agent lifecycle | P0 |
| AXUI Schema v0.1 | Schema spec + JSON Schema validator | P0 |
| CLI: `create-atlax` | Basic project scaffolding | P0 |

**Milestone**: An agent can receive an intent, query memory, and return an AXUI schema that renders.

### Phase 1: Complete Core (Weeks 7–14)

| Deliverable | Description | Priority |
|-------------|-------------|----------|
| `@atlax/synapse` | Multi-model LLM Router with complexity classification | P0 |
| `@atlax/neo` | AXUI → React + React Native renderer | P0 |
| `@atlax/bridge` | MCP adapter for external tools | P0 |
| AXP v0.1 | Inter-agent protocol: request-response + pub-sub | P0 |
| Ambient Dashboard | Layout engine + micro-app prioritization | P1 |
| Audit Log | Append-only log with hash chaining | P1 |
| CLI: `atlax dev` | Dev server with hot reload and devtools | P1 |

**Milestone**: Multiple agents collaborate, use external tools, and the dashboard shows dynamic micro-apps.

### Phase 2: Complete Experience (Weeks 15–22)

| Deliverable | Description | Priority |
|-------------|-------------|----------|
| Voice | Voice input (Web Speech API + expo-speech) | P1 |
| Proactive notifications | Agents that suggest actions without user asking | P1 |
| Cryptographic identity | Ed25519 key pairs + signed manifest per agent | P1 |
| Local embeddings | ONNX Runtime Web/Mobile for all-MiniLM-L6-v2 | P1 |
| `@atlax/testing` | Agent testing framework | P1 |
| Themes and customization | Theme system + semantic tokens | P2 |
| CLI: `atlax build` | Production build (web + mobile) | P1 |

**Milestone**: The OS is usable end-to-end with voice, notifications, local embeddings, and testing.

### Phase 3: Marketplace and Ecosystem (Weeks 23–30)

| Deliverable | Description | Priority |
|-------------|-------------|----------|
| `@atlax/zion` | Marketplace client: publish, install, verify | P1 |
| Zion Registry | Registry server (self-hostable) | P1 |
| Reputation system | Automatic scoring + user feedback | P2 |
| E2E Sync | Optional encrypted cloud synchronization | P2 |
| A2A Bridge | Bridge for Atlax agents to communicate with external A2A agents | P2 |
| CLI: `atlax publish` | Publish agents to Zion | P1 |
| Complete documentation | Guides, API reference, tutorials | P0 |

**Milestone**: Functional ecosystem with marketplace, reputation, and sync.

### Phase 4: Advanced (Weeks 31+)

| Deliverable | Description | Priority |
|-------------|-------------|----------|
| Custom AXUI components | Registration and distribution of custom UI components | P2 |
| Multi-modal input | Camera + location as contextual inputs | P2 |
| Agent devtools | Visual inspector for agents, memory, and AXP | P2 |
| Performance optimization | Lazy loading of agents, AXUI schema caching | P2 |
| Desktop (Tauri) | Desktop support via Tauri | P3 |
| Plugin system | Extensions for Cortex, Neo, Vault | P3 |

---

## 14. Technical Decisions and Trade-offs

### 14.1 Why TypeScript code-first and not a DSL?

**Decision**: Everything is defined in TypeScript with decorators and functions.

**Reason**: A DSL creates a barrier to entry. TypeScript is already the language of the React/RN ecosystem. Decorators provide DSL-like ergonomics without the cost of learning a new language. Additionally, the developer has access to the entire npm ecosystem.

**Trade-off**: Fewer compile-time restrictions than a pure DSL. Mitigated with runtime validation and strong types.

### 14.2 Why declarative AXUI and not open-ended generative UI?

**Decision**: Agents emit JSON schemas (AXUI) that the runtime renders, not React/HTML code.

**Reason**:
- **Security**: A JSON schema cannot execute arbitrary code. Generated HTML/JS can.
- **Consistency**: The schema goes through the theme and layout engine. Generated UI can break the design.
- **Performance**: Parsing JSON is orders of magnitude faster than evaluating generated JSX.
- **Portability**: The same schema renders on web and mobile without changes.

**Trade-off**: Less flexibility than generating HTML/React directly. Mitigated with the registrable custom component system.

### 14.3 Why a proprietary protocol (AXP) and not just A2A?

**Decision**: Proprietary protocol for intra-system communication, A2A as a bridge for external communication.

**Reason**:
- A2A is HTTP-based and designed for cross-network. For local communication between agents on the same device, it's unnecessary overhead.
- AXP uses MessagePack over an in-memory bus: sub-millisecond latency.
- AXP integrates cryptographic signing and permissions natively with Bastion.
- A2A doesn't handle the sandbox/permissions model that Atlax needs.

**Trade-off**: Another protocol to maintain. Mitigated with an A2A bridge for interoperability.

### 14.4 Why local-first and not cloud-first?

**Decision**: Data on-device by default. Cloud is opt-in.

**Reason**:
- Privacy as a differentiating feature. In a world of agents that know everything about you, having your data not on a server is a huge selling point.
- Latency: querying a local DB is instantaneous vs an API call.
- Offline-capable by design.
- The user has total control of their data.

**Trade-off**: Sync between devices is more complex. Mitigated with CRDTs (Conflict-free Replicated Data Types) for E2E sync.

### 14.5 Why Web Workers for sandboxing and not WASM?

**Decision**: Web Workers as the sandbox mechanism on web. WASM as a future option.

**Reason**:
- Web Workers provide real memory and execution context isolation.
- Available in all modern browsers. WASM sandbox (e.g., Wasmer) is not yet mature enough on web.
- Allows running agent TypeScript code without additional compilation.

**Trade-off**: Workers have communication overhead via `postMessage` (serialization). Mitigated with `SharedArrayBuffer` + `Atomics` where available, and transferables for large data.

---

## 15. Glossary

| Term | Definition |
|------|-----------|
| **Agent** | Autonomous entity with identity, memory, and permissions that processes user intents. |
| **Agent-First** | Paradigm where the agent is the entry point, not the visual interface. |
| **Zion** | Atlax agent marketplace. |
| **Neo** | UI Runtime that renders AXUI schemas into React/RN components. |
| **AXUI** | Atlax UI Schema. Declarative JSON format for describing interfaces. |
| **AXP** | Atlax Exchange Protocol. Intra-system inter-agent communication protocol. |
| **Bastion** | Security layer: sandbox, permissions, audit log, crypto identity. |
| **Bridge** | MCP adapter for connecting agents with external tools. |
| **Cortex** | Central orchestrator: intent routing, lifecycle, AXP bus. |
| **Ambient Dashboard** | OS primary surface: adaptive micro-app grid. |
| **Intent** | User intent expressed in natural language. |
| **Micro-App** | Unit of interface dynamically generated by an agent via AXUI schema. |
| **MCP** | Model Context Protocol. Open standard (Anthropic) for connecting LLMs with tools. |
| **Synapse** | LLM Router: selects optimal model per task. |
| **Vault** | Encrypted local-first memory system: working, episodic, semantic, preferences. |

---

## Appendix A: AXUI Schema — Complete JSON Schema

The formal AXUI JSON Schema will be distributed as `@atlax/axui-schema` and available at:
```
https://schema.atlax.dev/axui/1.0.json
```

This will enable IDE validation, autocompletion, and CI/CD verification.

---

## Appendix B: Complete Minimal Project Example

```typescript
// atlax.config.ts
import { defineConfig } from '@atlax/core';

export default defineConfig({
  name: 'My First OS',
  platforms: { web: { framework: 'react', bundler: 'vite' } },
  llm: {
    providers: {
      anthropic: {
        models: { 'claude-haiku': { costPer1kTokens: 0.0008 } },
        apiKey: { env: 'ANTHROPIC_API_KEY' },
      },
    },
    strategies: { default: { prefer: ['anthropic.claude-haiku'] } },
  },
  memory: { encryption: { algorithm: 'aes-256-gcm', keyDerivation: 'argon2id' } },
  security: { sandbox: 'webworker', auditLog: true },
});

// agents/notes/agent.ts
import { Agent, Intent, AgentContext } from '@atlax/core';

@Agent({
  name: 'notes',
  version: '1.0.0',
  description: 'Takes and searches user notes',
  permissions: ['memory.read:notes.*', 'memory.write:notes.*', 'ui.render'],
})
export class NotesAgent {
  @Intent({ patterns: ['take a note', 'write down', 'remember that'] })
  async createNote(ctx: AgentContext, params: { content: string }) {
    const id = await ctx.memory.store('notes.entries', {
      content: params.content,
      createdAt: Date.now(),
    });
    await ctx.memory.semanticStore('notes.search', params.content, { noteId: id });
    return ctx.ui.render('NoteCreated', { content: params.content });
  }

  @Intent({ patterns: ['search notes', 'what did I note about'] })
  async searchNotes(ctx: AgentContext, params: { query: string }) {
    const results = await ctx.memory.semanticSearch('notes.search', params.query, 5);
    return ctx.ui.render('NotesList', { notes: results });
  }
}
```

---

*Atlax SDK — Technical Specification v0.1*
*February 2026*
*License: MIT*