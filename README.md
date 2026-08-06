# ShopSage

**An AI shopping assistant that understands what you want, remembers who you are, and keeps you safe.**

ShopSage helps shoppers discover and compare products from a catalog using conversational search (RAG), checks live inventory and order status via MCP tools, and remembers style and budget preferences across visits.

---

## Core Capabilities

### 1. Conversational product search (RAG)
Describe what you need in plain language — ShopSage retrieves matching products from the catalog and ranks them.

*"A crisp formal shirt for office wear, men, size L"* → a ranked shortlist with reasons, each item grounded in a real catalog entry.

### 2. Follow-up questions
Context carries across the conversation, so you can ask about an item you've already been shown.

*"Does the second one come in green?"* → resolves the reference and answers from the catalog's colour list.

### 3. Live inventory & order tracking
Stock and order status come from **tool calls against Postgres**, not from the product description — so the assistant can't guess at availability.

*"Is it in stock in size M?"* · *"Where's my order?"*

### 4. Remembered preferences
A budget stated in one session is applied in the next, unprompted — and the assistant says so rather than filtering silently.

Session 1: *"hiking gear under ₹1500"* → Session 2: *"show me jackets"* filters to ₹1500.

### 5. Guardrails
Two rules filter candidates before they ever reach the shopper, and both are logged to the trace:

- **Stock filter** — retrieved products are checked against Postgres in one bulk query; anything out of stock is dropped.
- **Age filter** — when the shopper is buying for a child, retrieval is restricted to `age_appropriate=True` items.

A remembered budget is applied as a *suggestion*, not a hard filter — the assistant says it is applying it rather than silently narrowing results.

---

## Architecture

| Layer | Component | Notes |
|---|---|---|
| Agent | `src/Agent_3.py` | The current agent — routing, RAG, MCP client, memory, guardrails. `Agent_1`/`Agent_2` are earlier prototypes kept for reference |
| Retrieval | LangChain + ChromaDB + HuggingFace `all-MiniLM-L6-v2` | One document per product, embedded from `product_catalog_v2.jsonl` |
| LLM | Llama 3.3 70B via Groq | Slot extraction (routing) + grounded generation |
| Tools | `check_inventory`, `track_order` over **MCP** (stdio) | Back onto Neon Postgres — see [docs/tools.md](docs/tools.md) |
| Memory | JSON store, per shopper | See [docs/memory.md](docs/memory.md) |
| Observability | Per-request trace + dashboard aggregation + optional LangSmith | See [docs/observability.md](docs/observability.md) |
| API | FastAPI | `backend/main.py` |
| UI | React + Vite (primary) · Gradio (demo) | |

Query flow: **extract slots → route** → `new_search`/`follow_up` take the RAG path, `inventory_check` resolves a product name to a SKU then calls the inventory tool, `order_tracking` calls the order tool and skips retrieval entirely.

---

## Setup

**Prerequisites:** Python 3.11+, Node 18+, a [Groq API key](https://console.groq.com/keys), and a Postgres database ([Neon](https://neon.tech) free tier is what we use).

### 1. Install Python dependencies

```bash
python3 -m venv .venv
source .venv/bin/activate          # Windows: .venv\Scripts\activate
pip install -r requirements.txt
```

Or, with [uv](https://docs.astral.sh/uv/) (much faster):

```bash
uv venv .venv --python 3.12
source .venv/bin/activate
uv pip install -r requirements.txt
```

`requirements.txt` includes `-e .`, which installs the project itself so `src.*` and `DataBase.*` import correctly from anywhere.

### 2. Configure secrets

Copy `.env.example` to `.env` and fill in both values:

```bash
DATABASE_URL=postgresql://user:password@ep-xxxx-pooler.region.aws.neon.tech/dbname?sslmode=require
GROQ_API_KEY=your_key_here
```

Both are required — the app raises on startup without them. `.env` is gitignored; never commit it.

Running with a shared/public link instead of just locally? You can set `GROQ_API_KEYS` (comma-separated) instead of a single `GROQ_API_KEY` to rotate across multiple Groq accounts — see [docs/deployment.md](docs/deployment.md).

LangSmith tracing is optional and off by default — set both `LANGSMITH_TRACING=true` and `LANGSMITH_API_KEY` to switch it on. The in-app trace panel and dashboard work either way. See [docs/observability.md](docs/observability.md).

### 3. Load the dataset into Postgres

```bash
python -m src.db.load
```

Creates the `products` / `inventory` / `order_tracking` tables and loads them from `DataBase/`. Safe to re-run — it clears and reloads in FK-safe order. The inventory and order tools query these tables.

### 4. Build the vector store

```bash
python -m src.Ingest_Embedding
```

Embeds the 100-product catalog (`DataBase/product_catalog_v2.jsonl`) into `chroma_db/` as collection `shopsage_catalog`. Only needed once, and re-run when the catalog changes; the agent auto-ingests if it finds the store empty.

> **Run every command from the repo root.** `chroma_db/` is a relative path, so running from inside `src/` creates a second, empty store.

### 5. Run it

**React UI (primary)** — two terminals:

```bash
uvicorn backend.main:app --reload --port 8000    # terminal 1
cd frontend && npm install && npm run dev        # terminal 2
```

Open the Vite URL (default `http://localhost:5173`). It proxies `/api` to port 8000.

If port 8000 is already taken, run the backend elsewhere and point the proxy at it:

```bash
uvicorn backend.main:app --reload --port 8001
cd frontend && VITE_API_TARGET=http://127.0.0.1:8001 npm run dev
``` Log in with a customer ID — e.g. `CUST-0083` — to see personalization; any unknown ID creates a guest.

**Gradio demo** — single process, prompts for a customer ID in the terminal before launching:

```bash
python -m src.Shopsage_RAG_Demo2
```

---

## Tests

```bash
python tests/retrieval_test.py # sample queries against the vector store   (no secrets needed)
python -m src.memory           # memory write / cap-at-3 / read-back, temp copy (no secrets needed)
python tests/tool_test.py      # both MCP tools, known + unknown inputs   (needs DATABASE_URL)
python tests/trace_test.py     # agent trace: routing, memory recall, tool calls (needs both)
python tests/test_prompts.py   # budget-aware system prompt               (needs GROQ_API_KEY)
python tests/eval_suite.py     # full eval: RAG, tools, budget, guardrails, memory (needs both)
```

`eval_suite.py` scores the live agent across all five capabilities and writes a report; with LangSmith configured it also pushes the runs to your project dashboard.

Captured runs live in [docs/evidence/](docs/evidence/).

---

## Deployment

Free-tier stack: **Render** (backend) + **Vercel** (frontend) + **Neon** (Postgres, same database as local dev). Config is already in the repo — `render.yaml` and `frontend/vercel.json`. Full walkthrough, the Groq-quota caveats, and what to do if you don't have admin rights on this repo: [docs/deployment.md](docs/deployment.md).

---

## Project Structure

```
DataBase/          product_catalog_v2.jsonl (live) · product_catalog.jsonl (v1)
                   inventory.csv · order_tracking.csv
                   shopper_profiles.json · product_reviews.csv
src/
  Ingest_Embedding.py    catalog → Chroma
  Agent_1.py             prototype agent (RAG only)
  Agent_2.py             earlier agent: routing, RAG, local tool stubs
  Agent_3.py             current agent: routing, RAG, MCP client, memory, guardrails
  groq_router.py         rotates across GROQ_API_KEYS so no single account's daily quota caps a public deploy
  memory.py              shopper preference store
  observability.py       per-request trace + LangSmith wiring
  dashboard.py           aggregates traces: tool failure rate, guardrail hits, cache, latency
  system_prompt.py       standalone budget-aware prompt (used by tests/test_prompts.py)
  tools/                 check_inventory · track_order · retail_mcp_server
  db/                    SQLAlchemy models, session, CSV → Postgres loader
  Shopsage_RAG_Demo1.py  Gradio UI over Agent_1
  Shopsage_RAG_Demo2.py  Gradio UI over Agent_3
backend/main.py    FastAPI: /api/login, /api/chat, /api/memory, /api/health,
                   /api/dashboard, /api/dashboard/reset
frontend/          React + Vite chat UI · vercel.json (prod API rewrite)
render.yaml        Render blueprint for the backend (free-tier deploy)
docs/              team · data · tools · memory · observability · error analysis
                   demo script · evidence · pitch deck · deployment
tests/             retrieval · tool · trace · prompt tests + eval_suite
```

Product images are served from `frontend/public/images/` and referenced by SKU.

---

## Documentation

| Doc | Contents |
|---|---|
| [docs/Data Deatils.md](docs/Data%20Deatils.md) | All five data files: schema, columns, joins, worked examples |
| [docs/tools.md](docs/tools.md) | Tool signatures, inputs, outputs, error cases |
| [docs/memory.md](docs/memory.md) | Memory schema, precedence rules, cross-session recall |
| [docs/observability.md](docs/observability.md) | Trace shape, event kinds, LangSmith setup |
| [docs/error_analysis.md](docs/error_analysis.md) | Known failure modes and fixes |
| [docs/demo_script.md](docs/demo_script.md) | Walkthrough script for the demo |
| [docs/deployment.md](docs/deployment.md) | Free-tier hosting (Render + Vercel), multi-key Groq rotation, rate limiting |
| [docs/team.md](docs/team.md) | Team roles, stack, sign-offs |

## Team

Team 2 — see [docs/team.md](docs/team.md) for roles and contacts.
