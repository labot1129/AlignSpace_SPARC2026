## Problem

Interior design projects often require significant back-and-forth between clients and designers before the actual design process begins. Designers need to understand client preferences, budget, materials, fixtures, and other project requirements, while clients may struggle to communicate these preferences clearly.

AlignSpace explores how AI can make this discovery and handoff process more structured by helping translate client conversations and preferences into actionable project information for designers.

## What It Does

AlignSpace is an AI-assisted interior design platform with separate client and designer experiences.

### Client Experience
- Complete project discovery through a conversational interface
- Communicate design preferences and project requirements
- Explore design directions and material selections
- Review selections and hand off the project to a designer

### Designer Experience
- View client projects and project details
- Review collected preferences, design directions, and material selections
- Continue the design process using structured information collected during client discovery

# AlignSpace — AI Pipeline service (`as-ai-server`)

**Owner:** Engineer 3 (AI & Agentic Pipeline) · **Status:** running end to end, verified on the live Claude API · 17 tests passing

This service takes a client's messy renovation input (chat + preference chips +
budget + room size) and turns it into a **structured, priced, designer-ready
renovation brief**. It's the "brain" of AlignSpace, exposed as a FastAPI service
the main backend calls.

It runs **fully offline** (deterministic fallback — no key, DB, or Redis needed)
and uses **Claude automatically** the moment an API key is present.

### Jump to your part
- **Running it / reviewing the PR** → [Quick start](#quick-start)
- **Backend (Adam):** the API you call → [API contract](#api-the-contract-for-the-backend)
- **Frontend (Laura):** the JSON you render → [`/intake` + sample outputs](#api-the-contract-for-the-backend); material list swaps into `src/pipeline/presets.py`
- **Infra (Sam):** `Dockerfile` + `/health` + tests → [Quick start](#quick-start); CI red right now is the missing frontend, not the backend
- **One team decision** → [lean now, LangGraph later](#architecture-note-for-the-team-lean-now-langgraph-ready-later)

---

## Quick start

```bash
cd as-ai-server

# Fast path — just the packages this service uses (seconds):
pip install fastapi uvicorn pydantic anthropic pytest httpx
# Full team environment (also pulls heavier ML libs for the future memory agent; slower):
#   pip install -r requirements.txt

python demo.py                                         # watch the whole flow run
pytest                                                 # run the 17-test suite
uvicorn main:app --app-dir src --reload --port 8000    # run the API, docs at /docs
```

**What a correct run looks like:** for the sample "calm spa-like, warm, minimal"
brief, `Japandi` ranks #1 (~90%+), every pick is `standard` tier, and the budget
reads `within`. The over-budget sample (a roomy, luxe-leaning project) flips to
`over` and lists cheaper swaps. If you see those, all five agents are working.

### Turning on Claude

Intent extraction uses a deterministic keyword fallback by default, so demos and
CI never need a key. To use the real model, set the key in your shell before
running (the reliable way — don't commit it anywhere):

```bash
export ANTHROPIC_API_KEY=sk-ant-...
python demo.py        # intent extraction now routes through Claude
```

`.env.example` documents the variable name; copy it to `.env` to keep your key
handy. `GET /health` reports which path is live (`"intent_source": "claude"` vs
`"offline_fallback"`).

---

## How the pipeline works

Input is a `ClientBrief`. It flows through 5 agents and comes out as a
`RenovationPackage`:

```
ClientBrief  (chat text + chips + budget band + room sqft)
   │
   ▼
[1] Intent Extraction   →  ClientProfile        (Claude, or offline fallback)
   │                        styles{}, functions[], must_haves[]
   ▼
[3] Preset Matching     →  6 ranked directions  (Japandi, Organic Spa, ...)
   │
   ▼   ← client picks one direction in the UI
[4] Selection Assembly  →  MaterialPackage      (a product per category,
   │                        quantities from room size, confidence + flags)
   ▼
[5] Budget Validation   →  BudgetReport         (within/over + cheaper swaps)
   │
   ▼
[6] Document Generation →  RenovationPackage     (scope + selection sheet +
                            budget summary, as markdown/JSON)
```

The flow is **two-phase**, matching the product:
- **Phase A (`/intake`)** runs agents 1 + 3 → shows the client 6 directions.
- **Phase B (`/assemble`)** runs agents 4 → 5 → 6 on the direction they tapped.

> Agent 2, *Memory Lookup* (embeddings/pgvector), is intentionally **out of MVP
> scope** per the MVP Definition. It's left as a documented no-op slot right
> before Preset Matching so it can be added later without touching other agents.

Each stage fires a progress event through an `on_stage(stage, message)`
callback — the hook the backend wires to Redis pub/sub so the designer dashboard
can show "Extracting intent… / Matching presets…" live over Socket.io.

---

## API (the contract for the backend)

| Method | Route | Purpose |
|---|---|---|
| `GET` | `/health` | Liveness probe (app.yaml points here). Reports intent source. |
| `GET` | `/presets/directions` | The 6 directions, renderable before any intake. |
| `POST` | `/intake` | **Phase A** — returns `{profile, directions[6]}`. |
| `POST` | `/assemble` | **Phase B** — body `{brief, direction_key}` → full deliverable. |
| `POST` | `/pipeline/run` | Whole arc in one call (auto-picks top direction if none). |

Example:

```bash
curl -X POST http://localhost:8000/intake -H "Content-Type: application/json" -d '{
  "firm_id": "firm_demo",
  "project_id": "proj_001",
  "room_sqft": 45,
  "budget_band": "medium",
  "priorities": ["more storage"],
  "style_chips": ["warm", "minimal"],
  "chat_text": "calm spa-like bathroom, natural wood, walk-in shower, timeless"
}'
```

Request/response shapes live in `src/api_schemas.py` (wire models) and
`src/pipeline/models.py` (internal contracts). Two example responses are checked
in: `sample_deliverable.md` (within budget) and `sample_deliverable_over_budget.md`
(over budget + swaps). Full interactive docs at `/docs`.

---

## How it plugs into everyone else's work

| Boundary | Contract |
|---|---|
| **Backend (Eng 2)** | Builds `ClientBrief` from chat + chips, calls `/intake` then `/assemble`. All shapes in `models.py` / `api_schemas.py`. |
| **Frontend (Eng 1)** | Renders the 6 `DesignDirection` cards, then the selection sheet + budget panel from the deliverable JSON. |
| **Infra/Data (Eng 4)** | Maps `models.py` dataclasses → Postgres tables (shapes line up 1:1). |
| **Real-time (Eng 1+4)** | Subscribes to the `on_stage` events for Socket.io stage updates. |

This service is meant to **run alongside** the main backend and be called by it —
it is not a replacement for the backend's own routes (auth, projects, billing).

---

## Architecture note for the team: lean now, LangGraph-ready later

`Architecture.md` describes a full LangGraph + Celery + Redis + pgvector +
OpenAI-embeddings stack. That's the **target**. The MVP is built leaner, for two
reasons:

1. The repo's own `requirements.txt` is already lean — `anthropic`, `fastapi`,
   `chromadb`; **no** `langgraph`, `celery`, `redis`, or `pgvector` yet.
2. The MVP Definition explicitly **cuts** memory lookup, embeddings, vision, and
   the multi-loop optimizer.

So this is a clean sequential runner where **each agent is a pure, testable
node**. Promoting it to LangGraph later just means registering these same
functions as graph nodes; wrapping Phase B in a Celery task is a few lines. The
agent logic doesn't change either way. **Worth a 5-minute team decision: confirm
we're MVP-lean before anyone stands up the heavier infra.**

---

## File map

```
as-ai-server/
  Dockerfile               # used by app.yaml; serves uvicorn on :8000
  requirements.txt         # (unchanged from the repo)
  pytest.ini               # puts src/ on the path for tests
  demo.py                  # runnable end-to-end example (offline-friendly)
  .env.example             # documents ANTHROPIC_API_KEY (never commit the real one)
  src/
    main.py                # FastAPI app: /health + pipeline routes
    api_schemas.py         # Pydantic wire models (the HTTP contract)
    pipeline/
      models.py            # shared data contracts (the API everyone codes to)
      presets.py           # 6 directions + tiered material catalog (seed data)
      pipeline.py          # orchestrator + stage events (LangGraph-ready)
      agents/
        intent.py          # [1] Claude intent extraction + offline fallback
        matching.py        # [3] rank the 6 directions
        assembly.py        # [4] build package, confidence scoring, flagging
        budget.py          # [5] budget check + cheaper-swap alternatives
        document.py        # [6] assemble the deliverable (markdown/JSON)
  tests/
    conftest.py            # forces offline extraction so tests are hermetic
    test_pipeline.py       # core logic, incl. regression guards (offline)
    test_api.py            # HTTP contract via TestClient
  sample_deliverable.md            # example output: within budget (Japandi)
  sample_deliverable_over_budget.md# example output: over budget + swaps
```

The test suite never calls the live API (a fixture strips the key), so it runs
the same fast, deterministic way on every machine and in CI. The Claude path is
validated by hand via `python demo.py`.

---

## Status & next steps

Covers the Engineer 3 line through **Week 3**: agent contracts, the working
pipeline, and the **intent extraction agent** — now running end to end as a
FastAPI service, verified on the live model, with tests and a Dockerfile that fit
the repo.

**Next:** wire `on_stage` to real Redis pub/sub once Eng 4's Redis is up; replace
the seed catalog with Laura's real per-firm material list; (post-MVP) add the
memory-lookup agent.

**Pricing caveat:** all cost numbers are **materials-only illustrative ranges,
not quotes**. Price per item, quantities from room size, designer sets the final
number (per the team's intake discussion). Seed prices are placeholders for the
real material list.
