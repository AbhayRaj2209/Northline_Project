# LangSmith Observability

Northline traces LangGraph runs in [LangSmith](https://smith.langchain.com) following the official [Trace LangGraph applications](https://docs.langchain.com/langsmith/trace-with-langgraph) guide. Since agents use LangChain's `ChatGroq` inside LangGraph, tracing is enabled by **environment variables only** — no `@traceable` decorators.

## What gets traced
- Top-level run: `travel_planning` (new plans) or `travel_follow_up` (follow-ups)
- A nested span per graph node, with nested `ChatGroq` LLM calls
- Metadata: `user_id`, `thread_id`
- Tags: `northline`, `langgraph`, `user:<username>`
- Thumbs up/down feedback attached to the exact `run_id`

| File | Role |
|------|------|
| `backend/observability.py` | `configure_langsmith()` sets `LANGSMITH_*` env vars at startup; exposes status |
| `backend/main.py` | `build_run_config()` adds `run_name`, `metadata` and `tags` to each run |
| `backend/app/services/feedback_service.py` | Sends thumbs feedback to LangSmith |

## Setup
1. Create an API key at smith.langchain.com → **Settings → API Keys**.
2. Add to `backend/.env`:
   ```env
   LANGSMITH_TRACING=true
   LANGSMITH_API_KEY=lsv2_pt_...
   LANGSMITH_PROJECT=northline-travel
   # EU/APAC or multi-workspace only:
   # LANGSMITH_ENDPOINT=https://eu.api.smith.langchain.com
   # LANGSMITH_WORKSPACE_ID=...
   ```
3. Verify (from `backend/`):
   ```powershell
   python -c "from observability import get_langsmith_status; print(get_langsmith_status())"
   ```
   Expect `enabled: True`. If `reason` is `missing_api_key`, check `.env`.
4. Run the app, plan a trip, then open project **northline-travel** → latest `travel_planning` run → **Details** view for the node tree.

## Troubleshooting
| Symptom | Fix |
|---------|-----|
| `missing_api_key` | Add `LANGSMITH_API_KEY` and restart the backend |
| No traces | Project name must match `LANGSMITH_PROJECT`; wait ~10s and refresh |
| 401 / auth errors | Set `LANGSMITH_ENDPOINT` (EU/APAC) or `LANGSMITH_WORKSPACE_ID` |
| Traces cut off | `LANGCHAIN_CALLBACKS_BACKGROUND=false` (set by default in `observability.py`) |
