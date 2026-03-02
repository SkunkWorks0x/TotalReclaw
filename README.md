# TotalReclaw

**Your OpenClaw agents never forget.**

A `.claw` plugin that gives OpenClaw agents persistent memory across sessions. SQLite-backed, token-budgeted, zero external dependencies.

---

## The Problem

Every time your OpenClaw agent starts a new session, it starts from zero. It doesn't know what it did yesterday. It re-discovers the same API keys, re-makes the same mistakes, re-asks the same questions. If the agent was working toward a multi-step goal, that progress is gone.

This isn't a model limitation — it's an infrastructure gap. The agent has no memory layer.

## The Solution

TotalReclaw adds persistent memory to any OpenClaw agent. It captures what matters during a session, reflects on it at the end, and loads the right context at the start of the next one.

Three things happen automatically:
1. **Session start:** Relevant memories from prior sessions are retrieved and injected into the agent's system prompt.
2. **During the session:** Events are captured and filtered — side effects are stored, read-only actions are skipped, user directives get highest priority.
3. **Session end:** A structured reflection extracts key facts, lessons learned, goal status, and a primer for the next session.

The result: your agent picks up where it left off instead of starting cold.

---

## Before / After

**Without TotalReclaw:**
```
Session 1: Agent sets up Stripe. Creates customer. Webhook fails (403).
Session 2: Agent starts fresh. "What Stripe API version are we using?"
            Re-creates the same customer. Hits the same webhook error.
Session 3: Agent starts fresh again. Same questions. Same mistakes.
```

**With TotalReclaw:**
```
Session 1: Agent sets up Stripe. Creates customer. Webhook fails (403).
            → Reflection stored: "Webhook 403 — need API key permissions."
Session 2: Agent loads memory. Knows the customer exists, knows the
            webhook failed, knows to fix permissions first.
Session 3: Agent loads ALL prior context. Finishes the job.
            Zero repeated work.
```

---

## Quick Start

### Quick Start (Open Source)

```bash
pip install ./totalreclaw
```

```python
from totalreclaw import MemoryStore, retrieve_memories, format_memory_block

# Create a memory store for your agent
store = MemoryStore(agent_id="my-agent")

# Save memories as your agent works
store.save_episode("Created Stripe customer cus_123", goal_tag="payments")
store.save_directive("Always confirm before making paid API calls")
store.save_reflection("Set up payments. Customer created. Webhook failed.", goal_tag="payments")

# Next session — retrieve and inject memories
memories = retrieve_memories(store, current_goal="payments")
context = format_memory_block(memories)
# Inject 'context' into your agent's system prompt
```

### Quick Start (Pro — 3-line integration)

```bash
# After purchasing from Gumroad, unzip and install:
pip install ./totalreclaw
```

```python
from totalreclaw import TotalReclawPlugin

def call_llm(messages: list[dict]) -> str:
    """Your LLM API call. TotalReclaw doesn't care which provider."""
    return your_api.chat(messages=messages).content

# That's it. Three lines and your agent has persistent memory.
with TotalReclawPlugin("my-agent", llm_call=call_llm) as memory:
    prompt = memory.get_system_prompt("You are a coding assistant.")
    # prompt now includes memories from all prior sessions

    # Capture events as the agent works
    memory.capture("api_call_success", "Created Stripe customer cus_123")
    memory.capture("api_call_error", "Webhook 403 Forbidden")
    memory.capture_message("Always confirm before making paid API calls")

# Session ends here — reflection fires automatically
```

On the next session, the agent's system prompt will contain:
- A summary of what happened last time
- Standing directives ("always confirm before making paid API calls")
- Key facts discovered in prior sessions
- Recent activity for continuity

Run the included demo to see it in action:
```
python -m totalreclaw.examples.multi_session_demo
```

---

## Free vs Pro

TotalReclaw's core engine is open source. The production-ready plugin is available on Gumroad.

| | Open Source (GitHub) | Pro (Gumroad) |
|---|---|---|
| MemoryStore (SQLite-backed storage) | ✓ | ✓ |
| 4-layer retrieval algorithm | ✓ | ✓ |
| Context injection formatting | ✓ | ✓ |
| Smart capture filtering | ✓ | ✓ |
| Basic reflection prompt | ✓ | ✓ |
| Basic agent example | ✓ | ✓ |
| **TotalReclawPlugin (2-3 line integration)** | | ✓ |
| **Full reflection engine with fallback handling** | | ✓ |
| **Multi-session demo** | | ✓ |
| **113-test production suite** | | ✓ |
| **Priority support** | | Builder+ |
| **Private community + architecture blueprints** | | Pro |

→ **Get TotalReclaw Pro: [GUMROAD_LINK]**

---

## How It Works

### 4-Layer Retrieval

When a session starts, TotalReclaw retrieves memories in strict priority order:

| Layer | What | Why |
|-------|------|-----|
| 1. Reflection | Most recent session summary | "Where was I?" |
| 2. Directives | All standing user instructions | "What are my rules?" |
| 3. Goal-tagged | Memories related to the current goal, ranked by importance | "What do I know about this task?" |
| 4. Recent episodes | Latest actions regardless of goal | "What did I do recently?" |

A token budget (default: 2,000 tokens) prevents memory from bloating the context window. Higher-priority layers fill first. Lower layers get whatever budget remains.

### Reflection Engine

At session end, the transcript is sent to your LLM with a structured reflection prompt. The LLM returns JSON with:

- **Session summary** — 2-3 sentences of what happened
- **Goal status** — completed, partial, blocked, or failed
- **Key facts** — concrete information worth remembering (API versions, URLs, configurations)
- **Lessons learned** — specific, actionable takeaways to prevent repeated mistakes
- **Next session primer** — what to do first next time

Each item is scored for importance (1-10) and stored as a typed memory. Old reflections decay automatically to prevent bloat.

If the LLM call fails, TotalReclaw falls back to storing a basic transcript summary. Memory still works — just less structured.

### Capture Filtering

Not everything deserves to be a memory. TotalReclaw filters events by type:

- **Captured:** API calls (success and error), file writes, database mutations, user directives, decisions made
- **Skipped:** File reads, search queries, status checks, log output

User messages are scanned for directive signals ("always...", "never...", "from now on...") and stored at high priority.

### Storage

SQLite with WAL mode. One file. No server. No external dependencies. Multiple agents can share the same database — memories are scoped by `agent_id`.

---

## API Reference

### `TotalReclawPlugin`

The main integration class. Wraps the full memory lifecycle.

```python
TotalReclawPlugin(
    agent_id: str,
    db_path: str = "./totalreclaw.db",
    llm_call: Optional[Callable[[list[dict]], str]] = None,
    token_budget: int = 2000,
    current_goal: Optional[str] = None,
)
```

| Method | Returns | Description |
|--------|---------|-------------|
| `start_session(current_goal?, session_id?)` | `list[Memory]` | Begin session. Retrieves prior memories. |
| `end_session(transcript_override?, agent_context?)` | `dict` | End session. Runs reflection. Returns `{session_id, duration_seconds, memories_created, reflection_status, memory_ids}`. |
| `get_system_prompt(base_prompt)` | `str` | Base prompt + injected memory context. |
| `capture(event_type, content, goal_tag?, importance?)` | `Optional[str]` | Capture an event. Returns memory ID or None. |
| `capture_message(message, goal_tag?)` | `Optional[str]` | Scan user message for directives. Returns memory ID or None. |
| `set_goal(goal)` | `None` | Change goal mid-session. |
| `store` | `Optional[MemoryStore]` | Property. The underlying store. |
| `session_active` | `bool` | Property. Whether a session is running. |

Supports context manager protocol (`with ... as`).

### `MemoryStore`

Low-level SQLite-backed storage. Use this if you need direct control.

```python
MemoryStore(
    agent_id: str,
    db_path: str = "./totalreclaw.db",
    session_id: Optional[str] = None,
)
```

| Method | Returns | Description |
|--------|---------|-------------|
| `save(content, memory_type, goal_tag?, importance?)` | `Memory` | Save a memory. `memory_type`: `"episode"`, `"reflection"`, `"fact"`, or `"directive"`. |
| `save_episode(content, goal_tag?, importance=5)` | `Memory` | Save an episodic memory. |
| `save_reflection(content, goal_tag?, importance=8)` | `Memory` | Save a reflection summary. |
| `save_fact(content, goal_tag?, importance=6)` | `Memory` | Save a persistent fact. |
| `save_directive(content, importance=9)` | `Memory` | Save a user directive. Global scope (no goal tag). |
| `get_by_id(memory_id)` | `Optional[Memory]` | Retrieve one memory by ID. |
| `get_recent(limit=10, memory_type?)` | `list[Memory]` | Most recent active memories. |
| `get_by_goal(goal_tag, limit=20)` | `list[Memory]` | Memories for a goal, ranked by importance. |
| `get_directives()` | `list[Memory]` | All active directives. |
| `get_last_reflection()` | `Optional[Memory]` | Most recent reflection. |
| `count(memory_type?)` | `int` | Count active memories. |
| `mark_accessed(memory_ids)` | `None` | Update access count and timestamp. |
| `deactivate(memory_id)` | `None` | Soft-delete a memory. |
| `decay_old_reflections(keep_recent=5, importance_penalty=2)` | `None` | Reduce importance of old reflections. |
| `new_session(session_id?)` | `str` | Start a new session. Returns session ID. |
| `stats()` | `dict` | Memory counts by type, total sessions, db path. |

### `Memory`

Dataclass representing a single memory entry.

| Field | Type | Description |
|-------|------|-------------|
| `id` | `str` | UUID |
| `agent_id` | `str` | Owner agent |
| `session_id` | `str` | Session that created it |
| `created_at` | `float` | Unix timestamp |
| `memory_type` | `str` | `"episode"` \| `"reflection"` \| `"fact"` \| `"directive"` |
| `content` | `str` | The memory text |
| `goal_tag` | `Optional[str]` | Associated goal |
| `importance` | `int` | 1-10 score |
| `access_count` | `int` | Times retrieved |
| `last_accessed` | `Optional[float]` | Last retrieval timestamp |
| `is_active` | `bool` | False if soft-deleted |

### Retrieval Functions

```python
retrieve_memories(store, current_goal=None, token_budget=2000) -> list[Memory]
retrieval_stats(memories) -> dict
estimate_tokens(text) -> int
```

### Reflection Functions

```python
build_reflection_prompt(session_transcript, agent_context=None) -> list[dict]
parse_reflection(raw_response) -> Optional[dict]
store_reflection(store, reflection) -> list[str]
fallback_store_summary(store, summary_text, goal_tag=None) -> str
```

### Injection Functions

```python
format_memory_block(memories) -> str
build_system_prompt_with_memory(base_system_prompt, memories) -> str
```

### Capture Functions

```python
capture_event(store, event_type, content, goal_tag=None, importance_override=None) -> Optional[str]
capture_user_message(store, message, goal_tag=None) -> Optional[str]
should_capture(event_type) -> bool
```

---

## Pricing

| | Core | Builder | Pro |
|---|---|---|---|
| **Price** | $29 | $59 | $149 |
| Full plugin source code | Yes | Yes | Yes |
| All features, unrestricted | Yes | Yes | Yes |
| Community support | Yes | Yes | Yes |
| Priority bug fixes | | Yes | Yes |
| Early access to updates | | Yes | Yes |
| Private community access | | | Yes |
| Architecture blueprint docs | | | Yes |
| v2 early access | | | Yes |
| Direct support channel | | | Yes |

All tiers include the complete, unrestricted plugin. Higher tiers add support priority, community access, and early access to v2.

---

## FAQ

**Does it slow my agent down?**
Retrieval takes a single SQLite query per layer (4 queries total). On a typical machine, the full retrieval + formatting cycle completes in under 10ms. The reflection step at session end depends on your LLM call speed, but it runs after the agent's work is done — it never blocks the agent mid-task.

**How much does it add to API costs?**
One extra LLM call per session for reflection. With a short model (GPT-4o-mini, Claude Haiku), that's typically $0.001-0.005 per session. The memory injection adds ~500-2000 tokens to your system prompt, which increases per-message cost slightly. For most agents, total overhead is under $0.01/session.

**What if OpenClaw updates?**
TotalReclaw hooks into the agent lifecycle through the plugin interface, not internal APIs. If OpenClaw changes its plugin spec, we'll ship a compatibility update. Pro tier gets these first.

**Can I use it with multiple agents?**
Yes. Each agent gets its own memory scope via `agent_id`. Multiple agents can share the same SQLite database file without conflicts.

**What Python version do I need?**
Python 3.10 or higher. No external dependencies — TotalReclaw uses only the standard library (sqlite3, json, uuid, time, dataclasses, logging).

---

## Requirements

- Python 3.10+
- No external dependencies

## License

MIT License. See LICENSE for terms.

---

Built by [@SkunkWorks0x](https://x.com/SkunkWorks0x). Building in public — follow the journey.
