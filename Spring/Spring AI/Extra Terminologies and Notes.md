---
share_link: https://share.note.sx/s9m9l2rw#F6dKMDN2ivLWIocLlfAhBg
share_updated: 2026-08-01T19:22:42+03:00
---

### With multiple agents, how do you handle per-agent dependencies while staying "agent-agnostic"?

The core idea is to **separate the orchestration/harness layer from agent-specific configuration**, so no agent's specific model, tools, or dependencies are hardcoded into the system that runs them. A few concrete patterns:

- **Common interface/contract for every agent.** Every agent, regardless of what model or framework powers it, exposes the same shape of input/output (e.g., "receive a task + context, return a result or a tool call request"). The orchestrator only ever talks to this interface — it doesn't know or care whether Agent A is Claude-based and Agent B is GPT-based.
    
- **Config-driven agent definitions**, not code branching. Each agent's model, system prompt, tool list, and permissions are declared in configuration (YAML/JSON/DB record) rather than baked into if/else logic. Adding a new agent means adding a new config entry, not modifying the orchestrator's code.
    
- **Dependency injection per agent.** Each agent instance gets its own tool registry, credentials, and configuration injected at construction/runtime, rather than sharing a single global set of dependencies. This avoids one agent's tool set leaking into another's, and makes it easy to swap what an agent has access to.
    
- **Standardized tool-connectivity protocol** (e.g., MCP — Model Context Protocol). Tools are exposed through a standard protocol rather than agent-specific SDK calls, so any agent that speaks the protocol can use any tool, regardless of which model/framework is behind that agent.
    
- **Isolated environments per agent** (containers, sandboxes, or separate processes). If agents have conflicting library versions or need different runtime dependencies, isolating them (e.g., one container per agent, or per agent "role") avoids dependency collisions — similar to why you'd use separate virtual environments for different Python projects.
    
- **An orchestrator/coordinator pattern.** A central component routes tasks to the appropriate agent, aggregates results, and manages handoffs between agents — but it delegates all agent-specific concerns (which tools, which model, which prompt) to that agent's own configuration, keeping the orchestrator itself generic.
    

The overall goal: the system's _control plane_ (task routing, memory, permissions, observability) stays agent-agnostic and uniform, while each agent's _specific_ dependencies (its model, tools, prompt) are encapsulated and swappable behind a common interface.

---

### What is "harnessing" (an agent harness)?

An **agent harness** is the software scaffolding built around a language model that turns a raw model into a working agent. The model itself only does one thing well — reasoning over text and producing a response — it can't actually read files, run code, remember past turns, or safely take real-world actions on its own. The harness is the code that:

- Runs the loop that calls the model, parses its output, and decides what to do next
- Executes tool calls (file access, code execution, web search, API calls) on the model's behalf and feeds results back
- Manages memory/context — what the agent remembers across turns, how it summarizes or evicts old context
- Enforces permissions and safety boundaries — deciding what the agent is actually allowed to touch, rather than trusting the model to self-police
- Provides sandboxing — isolating where agent actions actually execute
- Handles observability — logging, tracing, and auditing what the agent did and why

A useful distinction: **prompt engineering** is about crafting better inputs to the model; **harness engineering** is about designing the entire system around the model — tools, memory, execution environment, and safety controls. As models get more capable, the quality of the harness increasingly determines how well an agent performs and how safely it behaves, since the harness — not the model — is what actually touches your filesystem, credentials, or production systems.

Examples of harnesses in the wild: Claude Code's built-in agent loop, the Claude Agent SDK, LangChain's Deep Agents, OpenAI's Agents SDK, and various minimal open-source harnesses (e.g., "Pi") that implement just the core loop and leave the rest to you.

---

### What is an "agent loop"?

The agent loop is the repeating cycle at the heart of every AI agent — often described as **perceive → reason → act**, or more concretely:

1. **Call the model** with the current context (task, conversation history, available tools)
2. **Parse the model's response** — is it a final answer, or does it want to call a tool?
3. **If it's a tool call**: execute the tool (via the harness, not the model directly) and capture the result
4. **Feed the tool result back** into the model's context
5. **Repeat** until the model produces a final answer, a stopping condition is met, or a hard exit/limit is hit (e.g., max iterations, budget, or timeout)

This pattern is closely related to the **ReAct** approach (Reasoning + Acting) from the research literature, where the model alternates between explicit reasoning steps and concrete actions. In practice, a minimal agent loop can be written in a small amount of code — the complexity in real systems comes from everything _around_ the loop: context management, tool permissioning, error handling, and knowing when to stop.

---

### What is PII, and what is a sandbox (in an AI agent context)?

**PII (Personally Identifiable Information)** is any data that can be used to identify a specific individual — names, email addresses, phone numbers, physical addresses, government ID numbers (SSNs, passport numbers), IP addresses, and similar. In AI agent systems, PII matters because:

- Agents often process user data (documents, emails, support tickets, form submissions) that may contain PII
- That data might get sent to a third-party model provider, logged, stored in memory/context, or exposed in outputs — creating privacy and compliance risk (e.g., GDPR, HIPAA depending on the domain)
- Modern agent harnesses increasingly include **PII detection middleware** — a step in the pipeline that scans inputs/outputs and redacts or flags sensitive data before it's sent to a model, logged, or persisted

**A sandbox** is an isolated execution environment where an agent's actual actions (running code, executing shell commands, modifying files) take place, separated from your real systems. The reasoning behind sandboxing:

- Agent-generated code or tool calls can be wrong, unpredictable, or (in adversarial cases) manipulated via prompt injection — running that directly against production systems is risky
- A sandbox contains the blast radius: if something goes wrong, only the disposable, isolated environment is affected, not real infrastructure, data, or credentials
- Sandboxes are often disposable/stateless — spun up per task, torn down afterward — and credentials/secrets are typically injected at a controlled boundary rather than stored inside the sandbox itself, so agent-generated code can't directly access them
- This ties back to the "brain vs. hands" framing some teams use: the model + harness (the "brain") decide _what_ to do; the sandbox (the "hands") is where that decision is actually _executed_, kept deliberately separate so a mistake in reasoning can't directly cause irreversible damage

Together, PII handling and sandboxing are two of the main **safety mechanisms** that live in the harness layer rather than in the model itself — the model can be told not to do something, but the harness is what actually enforces it.

---
