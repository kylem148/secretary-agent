# Architecture

The full v2 architecture for secretary-agent. The README has the high-level picture. This doc is the depth.

If you're skimming, the table of contents tells you what's here:

- [Core pattern: deep autonomous agents + shared state](#core-pattern)
- [Communication model: direct A2A, not mediated](#communication-model)
- [The draft is the agent's conversation](#the-draft-is-the-agents-conversation)
- [Negotiation routing: bypassing the orchestrator](#negotiation-routing)
- [Data layer](#data-layer)
- [Restart recovery](#restart-recovery)
- [Streaming and frontend integration](#streaming)

---

## Core pattern

The system uses **deep autonomous agents** combined with a **shared state table in PostgreSQL**. This combines the best of two patterns without the failure modes of either.

**Deep autonomous agents (what agents do):**
Each agent owns its complete loop end-to-end. The draft lives in the agent's own `self.messages`, never retrieved externally. The agent runs independently until done or cancelled, reports back to the orchestrator only on completion. Errors are isolated. One agent failing doesn't affect others.

**Shared state table (what the DB provides):**
Every agent writes key checkpoints to `agent_state`. The orchestrator reads `agent_state` to know what every agent is doing without interrupting any of them. Frontend reads `agent_state` to reconstruct task cards on page refresh. Cross-task instructions ("what are you working on?") get answered from the DB.

**Why not shallow check-in agents:**
When everything routes through the orchestrator at every step, context accumulates, the orchestrator becomes a bottleneck, and the re-invoke loop creates context bleed and state loss. These were the exact bugs from v1.

**Why not pure deep agents without shared state:**
Without a state table, the orchestrator is blind during active agent loops. Cross-task instructions have nowhere to route. The shared state table solves this without coupling agents to each other.

---

## Communication model

The orchestrator holds direct references to all active agents and reads `agent_state` for all of them regardless of who spawned them. Schedulers spawn EmailAgent and CalendarAgent instances directly as internal implementation details.

This is a **flat orchestrator-worker pattern**, not a strict deep hierarchy. Strict hierarchical opacity (orchestrator sees only direct children) exists to prevent bottlenecks at 50+ agent enterprise scale. This system peaks at ~8 simultaneous agents. The complexity of relay hops adds no value at this scale.

**What the orchestrator can do:**
- Read `agent_state` for all active agents (passive, no interruption)
- Send targeted instructions via `task_id` routing
- Spawn agents directly
- Cancel agents via `active_tasks[task_id].cancel()`

**What the orchestrator cannot do:**
- Read any agent's `self.messages` (private, never exposed)
- Interrupt any agent's running loop (only `task_id` routing allowed)
- Write to any agent's context (impossible by design)

**True A2A pattern:**
- Orchestrator directly spawns EmailAgent, CalendarAgent, Schedulers
- Schedulers directly spawn EmailAgent and CalendarAgent sub-agents
- Each agent calls its MCP tools directly
- Action Checker runs as a pre-hook at the MCP tool layer

**Mediated patterns we don't use:**
- Orchestrator → main.py → EmailAgent (unnecessary middleman)
- Agent → main.py → Action Checker → MCP (wrong layer for security)

Each sub-agent is a fully isolated black box to its spawner. Input is a `TaskEnvelope`, output is a `ResultEnvelope`. The Scheduler never sees the EmailAgent's `self.messages`. The EmailAgent never reads `agent_state`.

---

## The draft is the agent's conversation

The draft lives entirely inside the agent's own conversation loop. There is no structured ProposalCard widget. The agent speaks the draft as a plain chat message. The user responds in plain text. The agent understands intent naturally.

Example:

```
Agent (chat turn):
    "Here's a draft for Jake:

    [PROPOSAL]
    to: jake@example.com
    subject: Following up on bananas
    body: Hey Jake, just wanted to follow up on our banana bread
    conversation. Let me know if you still want to share the recipe!
    [/PROPOSAL]

    Let me know if you'd like any changes."

User: "make it funnier"

Agent (next turn):
    "Updated:

    [PROPOSAL]
    to: jake@example.com
    subject: Banana bread emergency
    body: Jake, I need to know, are we still doing this banana bread
    thing or did you ghost me AND the bananas?!
    [/PROPOSAL]"

User: "perfect, send it"

Agent: "Sent."
```

**Why this works:**

The draft never needs to be retrieved. It's always the last `[PROPOSAL]` block in `self.messages`. When the user says "make it funnier," the agent reads its own recent conversation history. There is no external lookup. No state loss. The context window IS the draft store.

The v1 bug (draft lost on revision) is structurally impossible here because the agent never fetches the draft from outside itself.

**Tagged output for frontend rendering:**

The agent produces tagged blocks inside its natural conversation. The frontend intercepts these tags and renders them as styled cards. The agent is completely UI-agnostic.

| Tag | Direction | Purpose |
|---|---|---|
| `[PROPOSAL]` | Agent → User | Email draft awaiting approval |
| `[CALENDAR_PROPOSAL]` | Agent → User | Calendar event awaiting approval |
| `[RESULT]` | Agent → Orchestrator | Plain text report on completion |
| `[STATUS]` | Agent → User | Negotiation progress update |
| `[DONE]` | Agent internal | Loop exit signal, stripped before streaming |

Anything outside a tag renders as normal chat text. The frontend owns all rendering logic. Adding a new card type costs one tag definition. No agent code changes needed.

---

## Negotiation routing

Long-running negotiations would normally accumulate in the orchestrator's context. This system bypasses the orchestrator for negotiation replies entirely.

When an inbound email arrives:

1. Gmail Pub/Sub webhook fires. Body lands in the Redis inbound queue.
2. EmailIngestor pulls from the queue. Strips HTML, runs the Injection Checker, dedupes against `provider_message_id`.
3. EmailIngestor resolves the email's chain (Gmail thread ID) against `email_chains`.
4. EmailIngestor queries `negotiations.chain_ids[]` for an active match:
   ```sql
   SELECT task_id FROM negotiations
   WHERE chain_id = ANY(chain_ids)
   AND current_state NOT IN ('CONFIRMED', 'CANCELLED')
   ```
5. If a match exists: deliver the body directly to that Scheduler's `_inbox`. The orchestrator is never involved.
6. If no match: put on `orch_q` for the orchestrator to handle as a new task.

This means once a Scheduler is spawned for "Tom, Tuesday meeting," every email Tom sends about that thread goes straight to the Scheduler. The orchestrator sees the Scheduler spawn and the eventual ResultEnvelope. Nothing in between.

**Why this matters:**

The orchestrator only reasons about emails that genuinely need fresh judgment. A multi-day negotiation with 8 back-and-forth replies costs the orchestrator nothing. The Scheduler holds the negotiation context inside its own loop, where it belongs.

This pattern also handles the cross-chain edge case. If Tom replies in a new Gmail thread mid-negotiation, the new chain has no match, so EmailIngestor sends it to the orchestrator. The orchestrator surfaces it to the user: "I got an email from Tom that looks like part of an ongoing conversation but I don't have the thread context." User confirms, orchestrator writes the new chain_id to `negotiations.chain_ids[]`, and from then on EmailIngestor routes correctly. One-time correction, permanent self-healing.

---

## Data layer

PostgreSQL holds the durable state. Redis is the fast path for active session work. In-memory holds per-process state that doesn't need to survive restarts.

### Operational data (PostgreSQL)

| Store | Written by | Read by | Why it exists |
|---|---|---|---|
| `email_chains` | EmailIngestor | EmailAgent | One row per Gmail thread. Mirrors Gmail's thread model. `provider_thread_id` is the natural key. EmailIngestor resolves or creates this row on every ingest before writing the email row. Chain always exists before message. |
| `emails` | EmailIngestor | EmailAgent, Scheduler (restart only) | One row per individual message. Full plain-text body stored, HTML stripped at ingest. `ai_summary` stored ONLY for Scheduler restart recovery. NOT used for routing or orchestrator decisions. `provider_message_id UNIQUE` enforces dedup on webhook retry. |
| `negotiations` | Schedulers | EmailIngestor (chain_id lookup), Schedulers (restart) | One row per active multi-day negotiation. `chain_ids UUID[]` supports multiple Gmail threads per negotiation. State machine: round, proposed slots, current_state, owner agent. Deleted when negotiation completes or is cancelled. |
| `agent_state` | Each agent | Orchestrator, frontend | One row per running agent. Written at lifecycle checkpoints. Deleted on completion or cancellation. The table always shows active work only. Orchestrator reads `context_summary` to answer "what are you working on?" Frontend reads to render task card tabs. No draft content stored. Drafts live only in `self.messages`. |
| `actions_log` | Action Checker | Audit / user queries | Append-only. One row per `gmail_send`, `create_event`, `delete_event` committed. The authoritative record of what the system did and when. Never deleted. |

### User configuration (PostgreSQL)

| Store | Written by | Read by | Why it exists |
|---|---|---|---|
| `user_preferences` | API / onboarding | Orchestrator → TaskEnvelope | Single row per user. One field: `notes TEXT`, plain text, user-edited, injected as-is into every TaskEnvelope. One read at task start. Full overwrite on update. The LLM interprets relevance per task. Phase 2: swap to pgvector retrieval if volume warrants semantic search. |

### Active session (Redis + in-memory)

| Store | Backend | Written by | Read by | Why it exists |
|---|---|---|---|---|
| `inbound queue` | Redis | Gmail webhook | EmailIngestor | Where Gmail Pub/Sub lands first. EmailIngestor pulls from here, processes, then either bypass-routes or pushes to `orch_q`. |
| `session turns` | Redis | Orchestrator | Orchestrator | Recent orchestrator conversation history. 30-day TTL. |
| `orch_q` | In-memory | HTTP endpoint, completed agents | Orchestrator loop | `asyncio.PriorityQueue`. Events requiring orchestrator reasoning. Priority 0: user messages. Priority 1: unknown inbound emails. Priority 2: agent results. Negotiation replies never enter here. |

### Agent working memory (in-memory)

| Store | Written by | Read by | Why it exists |
|---|---|---|---|
| `self.messages` | Agent | Agent only | All turns, `[PROPOSAL]` tags, tool results. Private. Dies with the agent. The draft IS `self.messages`. Never retrieved from outside. |

### Orchestrator email-blindness

The orchestrator never reads `email_chains`, `emails`, `negotiations`, or `actions_log`. Email content questions get delegated to EmailAgent.

This is a deliberate security boundary. The blast radius of an orchestrator compromise is bounded by what the orchestrator can read. It can't read email content directly, so injection attacks against the orchestrator can't exfiltrate email history.

---

## Restart recovery

On server startup, before accepting new events, the system inspects every row in `agent_state` and reconstructs accordingly.

**Short-lived agents (Email, Calendar):** dropped on restart. The user was present when the server died. The cost of restarting is near zero, the user re-requests if needed. Orphaned `agent_state` rows are deleted. Frontend shows the card as "interrupted."

**Long-lived agents (Schedulers):** reconstructed. An external party may have replied during downtime. The Scheduler reconstructs from `negotiations` + `email_chains` + `emails`:

```python
neg = await db.fetch("negotiations WHERE task_id = ?", row.task_id)
history = await db.fetch(
    "SELECT e.* FROM emails e "
    "JOIN email_chains ec ON e.chain_id = ec.id "
    "WHERE ec.id = ANY(%s) ORDER BY e.sent_at ASC",
    neg.chain_ids
)
agent = SchedulingAgent.reconstruct(neg, history)
active_agents[row.task_id] = agent
active_tasks[row.task_id] = asyncio.create_task(agent.run(orch_q, stream_q))
```

This is why `email_chains` and `emails` exist as durable tables. Not for orchestrator reads (which never happen). For Scheduler reconstruction.

A 30-day cutoff on the recovery query prevents the system from trying to reconstruct months-old stale negotiations.

---

## Streaming

The frontend gets data through three SSE channels, each with a distinct role:

| Channel | Created when | Closed when | Used for |
|---|---|---|---|
| `per_request_stream_q` | User sends a message via `POST /api/chat` | After None sentinel | Orchestrator response tokens for a single user message |
| `orchestrator_notify_q` | Server lifetime, never destroyed | Never | Agent completion synthesis, inbound email notifications, anything the orchestrator surfaces without a user message |
| `agent_task_stream_q` | Agent spawns via `_spawn_agent()` | Agent completes (None sentinel) | Per-task tokens and card events for task card rendering |

All puts use `_safe_stream(timeout=1.0)` so the agent never blocks if no consumer is listening.

The frontend opens one persistent EventSource on mount for `orchestrator_notify_q`, opens a new EventSource per user message for response tokens, and opens one EventSource per active task card. Page refresh reconstructs task cards from `agent_state`, then re-opens streams for any agents still running.

---

## Stack

- Python 3.12, FastAPI, uvicorn, async SQLAlchemy 2.0
- PostgreSQL with pgvector (memory and durable state)
- Redis (inbound queue, session turns)
- FastMCP for tool exposure (in-process Phase 1, separate services Phase 3)
- Anthropic API: Sonnet 4.7 for orchestrator and Schedulers, Haiku 4.5 for sub-agents
- Next.js 15, TypeScript, Tailwind, Zustand, Server-Sent Events
- Docker Compose for local dev
