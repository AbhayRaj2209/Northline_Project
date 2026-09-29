# Memory Architecture

Two separate tiers with separate keys. They are never mixed.

| Tier | Technology | Key | Scope | Stores |
|------|------------|-----|-------|--------|
| **Short-term** | LangGraph `PostgresSaver` (Neon) | `thread_id` | One chat session | Full `TravelState`: messages, every agent output, itinerary, errors |
| **Long-term** | [Mem0](https://docs.mem0.ai/integrations/langgraph) | `user_id` | All sessions, forever | Durable facts only: "I'm vegetarian", "budget ~$3000", "prefer direct flights" |

```
user_id   = rahul        → Mem0 (preferences)
thread_id = rahul_chat   → PostgresSaver (this session's trip state)
```

## Flow
- **Read:** `retrieve_memory` (graph start) searches Mem0 with `user_id` + the message and fills `memory_context` for every agent.
- **Write:** `store_memory` (graph end) uses an LLM to extract durable facts and saves them to Mem0. Greetings, one-off requests and weather questions are **not** stored.
- **Checkpoint:** PostgresSaver saves the full state after **every node**. It is used for session restore (`GET /api/chat/plan`) and follow-ups.
- **Follow-ups** read the checkpoint only (single LLM call, no agents); Mem0 adds optional context.
- **Corrections** ("Actually I'm vegan") are saved to Mem0 immediately via `chat_router.is_explicit_correction()`.

## Failure behavior
| Failure | Behavior |
|---------|----------|
| Mem0 unavailable | Logged; graph continues with empty `memory_context` |
| Postgres unavailable | Checkpoint error (short-term memory is required) |

## Config (`backend/.env`)
```env
DATABASE_URL=postgresql://...-pooler...neon.tech/neondb?sslmode=require
MEM0_API_KEY=m0-...
MEM0_ENABLED=true     # false disables Mem0; graph still runs
MEMORY_TOP_K=8        # max memories per query
```

## Modules
```
memory/
  memory_manager.py   facade injected into graph nodes (agents never touch Mem0/DB directly)
  retriever.py        Mem0 search + formatting
  extractor.py        LLM fact extraction + save
  provider/base.py    abstract provider (Mem0 is swappable)
  provider/mem0_provider.py
  state.py            TravelState TypedDict
graph/nodes/retrieve_memory.py, store_memory.py
```

## Test
```powershell
cd backend
python -m pytest tests/test_memory_manager.py tests/test_user_profile_memory.py -v
python scripts/test_memory_diagnostic.py      # live check (needs keys)
```
UI: as `rahul`, send `I'm vegetarian and prefer direct flights. Plan Tokyo.` Later, send `Plan Bali`: the planner should respect both preferences.

References: [Mem0 + LangGraph](https://docs.mem0.ai/integrations/langgraph) · [LangGraph persistence](https://docs.langchain.com/oss/python/langgraph/persistence)
