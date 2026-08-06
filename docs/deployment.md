# ShopSage Deployment

Free-tier stack: **Render** (backend) + **Vercel** (frontend) + **Neon** (Postgres — the same database used locally, no separate setup needed).

- **Backend blueprint:** [`render.yaml`](../render.yaml)
- **Frontend rewrite config:** [`frontend/vercel.json`](../frontend/vercel.json)
- **Multi-key Groq rotation:** [`src/groq_router.py`](../src/groq_router.py)
- **CORS + rate limit:** [`backend/main.py`](../backend/main.py)

## Why not a single simpler host

The backend isn't stateless: it needs a persisted `chroma_db/` directory and a long-running MCP stdio subprocess. That rules out pure serverless platforms (Vercel/Netlify functions, AWS Lambda) for the backend — it needs a real container or VM, not a request-scoped function. Vercel is still the right fit for the frontend, which is just a static build.

## Steps

### 1. Backend on Render

1. Push the repo to a GitHub repo you have admin rights on. Render needs to install a GitHub App to deploy from it, which needs repo-admin access — if you don't own the source repo, fork it into your own account first (same git history, you just own the fork).
2. [render.com](https://render.com) → sign in with GitHub → **New → Blueprint** → select the repo → it auto-detects `render.yaml`.
3. Set env vars when prompted:

   | Var | Value |
   |---|---|
   | `DATABASE_URL` | your Neon connection string |
   | `GROQ_API_KEYS` | comma-separated keys from **separate** Groq accounts (or `GROQ_API_KEY` for a single key) |

4. Deploy, then copy the resulting URL (`https://<name>.onrender.com`).

`render.yaml`'s `startCommand` rebuilds the vector store on every boot (`python -m src.Ingest_Embedding`) since Render's free plan has no persistent disk — takes ~10s for the 100-product catalog. Postgres data is *not* reloaded on boot; Neon already persists it, so `src.db.load` only needs to run once, from your own machine, against the same `DATABASE_URL`.

### 2. Frontend on Vercel

1. Edit `frontend/vercel.json`, replacing the placeholder with your real Render URL from step 1.
2. Commit and push that change.
3. [vercel.com](https://vercel.com) → sign in with GitHub → **New Project** → same repo → set **Root Directory** to `frontend` → Deploy.
4. Copy the resulting URL (`https://<name>.vercel.app`) — this is the public link.

### 3. Close the CORS loop

On Render, set `DEPLOYED_FRONTEND_ORIGINS` to your Vercel URL from step 2 (comma-separated if there's more than one). Render redeploys automatically when an env var changes.

## Protecting the shared Groq quota

A public link means every visitor draws from the same Groq daily token budget, so two mitigations are built in:

- **Multi-key rotation** (`GROQ_API_KEYS`, read by `src/groq_router.py`) — sticky selection across clients, rotates to the next key on a 429 and parses Groq's own "try again in Xm Ys" message to know when a key frees up. Only multiplies your quota if each key is on a **separate** Groq account — Groq's daily limit is scoped per-account, so multiple keys under one account share the same pool and gain nothing from rotation. Check Groq's terms before creating multiple accounts to work around this.
- **Per-IP rate limit** on `/api/chat` (`backend/main.py`) — defaults to 30 messages/hour per visitor, tune with `CHAT_RATE_LIMIT_PER_HOUR`. In-memory only: resets on restart, and doesn't coordinate across instances — fine for Render's free single-instance plan, not meant to survive a horizontal scale-out.

## Known free-tier quirks

- Render's free web service sleeps after 15 minutes idle — the first request after that takes ~30-50s (cold boot + re-ingest).
- The Chroma vector store isn't persisted between restarts or redeploys — it's rebuilt every boot, which is fine for a fixed 100-product catalog but would need a paid disk (or a hosted vector DB) if the catalog grows large enough for ingestion to become slow.
- Neon's free tier also autosuspends after inactivity — the first query after idle has its own short cold start, independent of Render's.
