# AlignSpace

**AI-Assisted Interior Design Discovery & Designer Handoff Platform**

AlignSpace is a collaborative UPenn SPARC project exploring how AI-assisted workflows can help turn ambiguous client design preferences into structured, actionable information for interior designers.

> **Team Project — UPenn SPARC 2026**  
> My primary role: Frontend Engineering, Product/UX, and Frontend–Backend Integration

---

## Problem

Interior design projects often require significant back-and-forth before the actual design process begins.

Designers need to understand a client's style preferences, functional requirements, budget, materials, fixtures, and project constraints. Clients, however, may struggle to communicate these preferences clearly or translate inspiration into concrete design decisions.

AlignSpace explores how AI-assisted workflows can make this discovery process more structured and create a clearer handoff between clients and designers.

---

## What It Does

AlignSpace provides separate experiences for clients and designers.

### Client Experience

Clients can:

- Complete project discovery through a conversational interface
- Communicate style preferences and functional requirements
- Explore generated design directions
- Review material and fixture selections
- Track project budget information
- Confirm selections and hand the project off to a designer

### Designer Experience

Designers can:

- View incoming client projects from a project dashboard
- Review project requirements and collected preferences
- Review selected design directions and material selections
- Access structured project information collected during discovery
- Continue the professional design process from the client handoff

---

## My Contributions

I worked primarily on the frontend and product experience for AlignSpace, translating the team's AI-assisted workflow into usable client- and designer-facing experiences.

My contributions included:

- Built client- and designer-facing interfaces using **Next.js, React, TypeScript, and Tailwind CSS**
- Developed the conversational discovery interface, including message threads, preference chips, image-upload interactions, design-direction selection, and project-cost displays
- Built the **Designer Projects dashboard** and project-detail experiences for reviewing client projects and material selections
- Integrated frontend workflows with team-developed **FastAPI REST APIs**
- Implemented authentication and role-aware client/designer application flows using **Clerk**
- Worked across frontend/backend boundaries to align API response structures with UI requirements and troubleshoot integration issues
- Contributed product and UX decisions based on real-world interior-design workflows and translated those workflows into software requirements

---

## Tech Stack

### Frontend
- Next.js
- React
- TypeScript
- Tailwind CSS

### Backend & APIs
- FastAPI
- REST APIs

### AI Pipeline
- Anthropic Claude
- Structured AI outputs
- Multi-stage design recommendation pipeline
- Deterministic fallback for offline development and testing

### Data & Infrastructure
- PostgreSQL
- Redis
- Socket.io
- Docker

### Product & Analytics
- Clerk
- PostHog

---

## System Overview

At a high level, AlignSpace connects a conversational client experience with a structured AI-assisted design pipeline and a professional designer handoff.

```text
Client
  │
  ▼
Next.js Client Experience
  │
  │  project information
  │  preferences
  │  conversation
  ▼
Backend / FastAPI
  │
  ▼
AI Pipeline
  │
  ├── Intent Extraction
  ├── Design Direction Matching
  ├── Selection Assembly
  ├── Budget Validation
  └── Document Generation
  │
  ▼
Structured Project Data
  │
  ├──────────────► Client Review
  │
  └──────────────► Designer Dashboard
```

The AI pipeline converts conversational and structured client inputs into design directions, material selections, budget information, and a designer-ready project package.

The production MVP intentionally uses a lean sequential pipeline. More complex capabilities such as embedding-based memory retrieval and a LangGraph orchestration layer were considered as future architecture rather than represented as completed MVP functionality.

---

## Key Engineering Challenges

### Frontend–Backend Integration

One of my main challenges was coordinating frontend requirements with backend API development across a team.

The client and designer experiences depended on project, preference, material, message, and handoff data produced by different parts of the system. This required aligning API response structures with frontend state and UI requirements while debugging integration issues as the system evolved.

### Translating AI Output Into Product Experiences

The frontend could not simply display raw model output.

AI-generated information needed to become understandable product components such as design-direction cards, material selections, project summaries, budget information, and designer-facing project data.

This required thinking about both the structure of the AI output and how users would interact with it.

### Designing Across Two User Roles

AlignSpace supports two connected but different workflows:

**Client → discovery and decision making**

**Designer → project review and professional continuation**

The frontend therefore needed role-aware navigation, authentication, interfaces, and handoff behavior while maintaining continuity between both sides of the project.

---

## What I Learned

This project gave me practical experience working at the intersection of frontend engineering, AI-enabled product development, and backend integration.

In particular, I gained experience with:

- Designing interfaces around structured AI outputs
- Integrating frontend applications with REST APIs
- Coordinating frontend and backend development in a team environment
- Designing role-aware product workflows
- Debugging application state and integration issues
- Translating domain-specific workflows into software requirements

It also showed me that building an AI product involves much more than calling an LLM: the model output must fit into reliable APIs, application state, user workflows, persistence, testing, and a usable product experience.

---

# Technical Documentation

The following section contains the team's detailed documentation for the AlignSpace AI pipeline.

---

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
# pip install -r requirements.txt

python demo.py
pytest
uvicorn main:app --app-dir src --reload --port 8000
```

**What a correct run looks like:** for the sample "calm spa-like, warm, minimal"
brief, `Japandi` ranks #1 (~90%+), every pick is `standard` tier, and the budget
reads `within`. The over-budget sample (a roomy, luxe-leaning project) flips to
`over` and lists cheaper swaps. If you see those, all five agents are working.

### Turning on Claude

Intent extraction uses a deterministic keyword fallback by default, so demos and
CI never need a key. To use the real model, set the key in your shell before
running:

```bash
export ANTHROPIC_API_KEY=sk-ant-...
python demo.py
```

`.env.example` documents the variable name; copy it to `.env` to keep your key
handy. `GET /health` reports which path is live (`"intent_source": "claude"` vs
`"offline_fallback"`).

---

## How the pipeline works

Input is a `ClientBrief`. It flows through 5 agents and comes out as a
`RenovationPackage`:

```text
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
curl -X POST http://localhost:8000/intake \
  -H "Content-Type: application/json" \
  -d '{
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
`src/pipeline/models.py` (internal contracts).

Two example responses are checked in:

- `sample_deliverable.md`
- `sample_deliverable_over_budget.md`

Full interactive docs are available at `/docs`.

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

## Architecture Note: Lean Now, LangGraph-Ready Later

`Architecture.md` describes a full LangGraph + Celery + Redis + pgvector +
OpenAI-embeddings stack. That's the **target architecture**, not the current MVP.

The MVP is built leaner for two reasons:

1. The repository's current requirements are intentionally lean and do not yet
   include LangGraph, Celery, Redis, or pgvector for this pipeline.
2. The MVP definition explicitly cuts memory lookup, embeddings, vision, and the
   multi-loop optimizer.

The current implementation therefore uses a clean sequential runner where each
agent is a pure, testable node.

A future LangGraph implementation could register these functions as graph nodes
without requiring the underlying agent logic to be rewritten.

---

## File Map

```text
as-ai-server/
  Dockerfile
  requirements.txt
  pytest.ini
  demo.py
  .env.example
  src/
    main.py
    api_schemas.py
    pipeline/
      models.py
      presets.py
      pipeline.py
      agents/
        intent.py
        matching.py
        assembly.py
        budget.py
        document.py
  tests/
    conftest.py
    test_pipeline.py
    test_api.py
  sample_deliverable.md
  sample_deliverable_over_budget.md
```

The test suite never calls the live API. A fixture strips the API key so tests
run through the deterministic fallback, keeping CI fast and reproducible.

The Claude path can be validated separately through:

```bash
python demo.py
```

---

## Status & Next Steps

The current AI pipeline covers:

- Structured agent contracts
- Intent extraction
- Design-direction matching
- Selection assembly
- Budget validation
- Document generation
- FastAPI endpoints
- Deterministic offline fallback
- Automated pipeline/API tests
- Dockerized service setup

Potential future work includes:

- Persisting more generated project data across the complete client → designer workflow
- Adding embedding-based memory retrieval
- Introducing LangGraph orchestration where more complex stateful agent behavior becomes useful
- Connecting pipeline progress events to real-time infrastructure
- Replacing seed material data with production-ready catalogs
- Expanding AI evaluation and regression testing

> **Pricing note:** Cost estimates in the current MVP are illustrative material
> ranges rather than professional quotes. Final project pricing remains part of
> the designer workflow.
