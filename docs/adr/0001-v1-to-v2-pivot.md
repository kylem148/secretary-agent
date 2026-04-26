# ADR-0001: Rebuild from hardcoded routing to free-reasoning orchestration

**Status:** Accepted

**Date:** 2026-01

**Author:** Kyle Morgan

---

## Context

v1 of secretary-agent shipped as a working multi-agent system. Orchestrator, email reader and sender, calendar read and write, scheduling agents, checkers, three-lane priority queue, full Gmail and Google Calendar integration. End-to-end inbound and outbound scheduling worked.

It was over-hardcoded.

The architecture had Python conditionals throughout the core flow, each one cementing a routing decision in code. `if event.type == "scheduling_request" → inbound_scheduling_agent`. `if sender_tier == "P1" → Lane 1`. `if email contains scheduling language → scheduling_request else general_email`. Every classification, every lane assignment, every spawning decision was hand-coded logic.

This wasn't accidental. It was a deliberate early choice. Hardcoded routing is easier to reason about, easier to test, and easier to extend incrementally. For a small system, it's the right call.

It stopped being the right call as soon as the product vision crystallized: the agent had to make thoughtful scheduling decisions like a human assistant would. That requires holding multiple inputs in mind at once (who's asking, what week is it, what's the priority, what's protected, what does the user actually want) and reasoning about all of them together. Hardcoded routing kept yanking the LLM out of context to make a routing decision, then dropping it back in with less state than it started with. The LLM never got to think.

Two specific problems made this concrete:

**Problem 1: Drafts lost on revision.**

User: "Email Jake about the meeting."
EmailAgent: drafts email, presents `[PROPOSAL]`.
User: "Make it funnier."
EmailAgent: tries to retrieve the draft from session memory to revise it.
EmailAgent: gets back something stale, partially formed, or empty.
EmailAgent: produces a revision that doesn't match what was just shown to the user.

The bug was that the draft was being externally retrieved by the orchestrator, mediated through queues, and rebuilt. The architecture forced retrieval. The retrieval kept losing state. We added more checks, more state synchronization, more defensive code. The bug kept coming back in different forms.

**Problem 2: Rules that should have been context.**

Adding a new behavior meant adding new routing code. The agenda gate was a Python conditional that fired before the LLM thought about the request. Contact tier was a routing decision that decided which agent ran. The weekly schedule template was a constraint that filtered slots before the LLM saw them.

Every one of these should have been context the LLM weighs, not a rule the system enforces. A P1 contact who needs a Tuesday meeting at 3pm during a focus block is exactly the kind of judgment call a thoughtful assistant makes. The system was structured to make that call impossible.

The fix wasn't adding more rules or fixing more bugs. It was rebuilding the core flow.

---

## Decision

Rebuild the entire architecture as v2. Five specific changes:

### 1. Remove all hardcoded routing

The orchestrator (Sonnet 4.7) gets the full event payload, all active `agent_state` rows, and `user_preferences`, and decides what to do. No Python conditionals decide which agent runs. No classification rules pre-sort events into lanes.

New agent types are added by adding one entry to `_AGENT_DESCRIPTIONS`. The orchestrator's system prompt builds the available-agents section from this dict at call time. Adding a Research Agent in Phase 2 will require zero changes to routing code. The orchestrator will see the new entry and start using it.

### 2. Direct A2A, not mediated communication

In v1: orchestrator → main.py → EmailAgent. In v2: orchestrator → EmailAgent directly. Schedulers spawn EmailAgent and CalendarAgent sub-agents directly, also without main.py in the path.

Each sub-agent is a fully isolated black box to its spawner. Input is a `TaskEnvelope`, output is a `ResultEnvelope`. The Scheduler never sees the EmailAgent's `self.messages`. The EmailAgent never reads `agent_state`. Each agent owns its loop end-to-end.

### 3. Draft state lives in `self.messages`

The draft IS the agent's conversation. There is no external draft store, no session memory entry to retrieve, no synchronization to manage. When the user says "make it funnier," the agent reads its own recent conversation history. The last `[PROPOSAL]` block is right there.

The v1 bug is structurally impossible in v2 because there is no retrieval to fail.

### 4. Action Checker as a tool-layer pre-hook

In v1, security ran inside the mediated communication pattern. Every action went orchestrator → main.py → checker → tool. Refactoring routing risked refactoring security. An agent that bypassed the orchestrator (which v2's direct A2A pattern allows) would also bypass the checker.

In v2, the checker is a pre-hook on the MCP tool itself. It fires before execution, regardless of caller. There is no path from agent to side effect that bypasses it.

### 5. Shared `agent_state` table for visibility

Every agent writes status checkpoints to `agent_state` in PostgreSQL. The orchestrator reads, never interrupts. The frontend reads to render task cards on page refresh. Cross-task questions ("what are you working on?") are answered from the DB, not from polling agents.

This combines deep autonomous agents (each owning its full loop) with the visibility of a centralized system (one table, queryable, always reflecting current work).

---

## Consequences

### Positive

**The LLM can think.** Scheduling decisions now happen with full context: contact tier, weekly priorities, schedule template, agenda quality, soft conflicts. The orchestrator weighs all of them together and produces proposals that explain the reasoning.

**No more state loss bugs.** The draft-on-revision class of bug is gone. Whatever the agent has in `self.messages` is what it has. Nothing is retrieved, nothing is synchronized, nothing can fail to load.

**Adding new behavior is cheap.** New agent types require one dict entry. New rules become user_preferences notes the LLM reads. Most product changes are config or content, not code.

**Security has a clean boundary.** The Action Checker pre-hook fires regardless of what's calling. Refactoring the orchestrator can't create a security regression. An agent with a bug, an agent acting on injected content, an agent that bypassed the orchestrator entirely, all hit the same gate.

**Long-running negotiations don't accumulate in the orchestrator.** EmailIngestor's chain matching routes negotiation replies directly to the right Scheduler, bypassing `orch_q`. The orchestrator only reasons about emails that genuinely need fresh judgment.

### Negative

**Harder to predict.** Hardcoded routing is reproducible. Free reasoning isn't. Two requests that look identical can produce different orchestrator decisions if the context differs. Testing this is harder than testing conditionals.

**Higher token cost.** The orchestrator now reasons about every routing decision instead of executing a conditional. For a small system this isn't a problem. For a high-volume system it would be.

**Requires more careful prompt engineering.** The orchestrator's system prompt is doing real work. Mistakes in the prompt cause behavior changes that are hard to trace. v1's bugs were in code. v2's bugs can be in prompts.

**Smoke test suite had to be rewritten.** v1's tests were largely "does the conditional fire correctly." v2's tests are "does the system produce a reasonable decision given this context." The shape of the test suite changed.

### Risks accepted

**The orchestrator could make a wrong decision.** It can. The Action Checker catches the dangerous failure modes (writes to the real world). For non-write decisions, the user sees the proposal before anything is booked, so a wrong call is a recoverable mistake, not a runaway action.

**Prompt regressions are possible.** Mitigated by the security adversarial test suite (15+ injection tests covering social engineering, base64-encoded directives, role reassignment, output overrides, hidden Unicode) and the broader smoke test suite.

---

## Alternatives considered

### A: Keep v1, fix bugs incrementally

Considered and rejected. The draft-loss bug had been "fixed" three times. Each fix added defensive code that handled one symptom. The architecture was producing the bugs faster than we could patch them.

The thoughtful-scheduling product vision was also incompatible with v1's structure. Hardcoded routing prevented the LLM from reasoning about full context, which is exactly what thoughtful scheduling requires. There was no incremental path from v1 to that product.

### B: Hybrid (LLM routing for some events, hardcoded for others)

Considered and rejected. The hybrid approach was the worst of both worlds. The hardcoded path meant the LLM still couldn't think about routing for those events. The LLM path meant maintaining two routing systems in parallel. New features had to decide which path they belonged to.

The complexity tax of two systems was worse than the cost of fully rebuilding one.

### C: Keep mediated communication, fix everything else

Considered and rejected. The mediated pattern (orchestrator → main.py → agents) was contributing to multiple problems simultaneously: state loss in drafts, security tied to routing, orchestrator-as-bottleneck. Fixing the rest without fixing the communication pattern would have required more workarounds, not fewer.

Direct A2A is simpler. Each agent owns its loop. The orchestrator gets out of the way. The architecture matches the actual data flow instead of inventing additional hops.

---

## Verification

The rebuild was verified with three test suites:

- 200+ unit tests covering the queue, models, classifier, calendar rules, checkers, and memory layer
- 30+ end-to-end smoke tests against real Gmail and Google Calendar (live OAuth, real email delivery, real event creation)
- 15+ adversarial security tests against the Injection Checker

The full inbound and outbound scheduling loops walk end-to-end against real services. Page-refresh recovery, server restart recovery for long-lived agents, real email threading via `In-Reply-To` headers, and idempotency on duplicate webhook delivery are all covered by the smoke suite.

---

## References

- v1 source: archived at `kylem148/secretary-agent-v1`
- v2 architecture: [docs/architecture.md](../architecture.md)
- v2 security model: [docs/security-model.md](../security-model.md)
