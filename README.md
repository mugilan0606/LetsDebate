# Let's Debate

Multi-agent company debate app on Cloudflare Workers. Add company agents from URLs, let each build a profile from public web sources, run structured opening statements and 3 rebuttal cycles, and receive a judged final verdict.

**Live:** [https://cf-ai-competitor-research.mugilan0606.workers.dev/](https://cf-ai-competitor-research.mugilan0606.workers.dev/)

---

## Overview

This project evolved from a single-company competitive research chatbot into a **debate-first workflow**. Instead of summarizing one company at a time, you assemble multiple company agents and ask comparative questions — each agent argues from its own scraped profile while an impartial judge scores the debate.

### What you get

- **Split-screen UI**
  - Left: add/remove company agents, monitor scrape status, view revenue and evidence counts
  - Right: debate chat with markdown rendering and quick-prompt suggestions
- **Background agent queue**
  - Queue multiple companies without waiting
  - Chat stays locked until all queued scrapes finish and at least 2 agents are ready
  - Cancel queued or in-progress agents from the sidebar
- **Structured multi-agent debate**
  - Opening statements (one per company agent)
  - 3 rebuttal cycles (each agent counters rival claims from the prior round)
  - Confidence scoreboard with per-company scores and rationale
  - Final verdict delivered as a separate assistant message
- **Tie-break logic**
  - If top scoreboard entries tie, a dedicated tie-break judge picks the winner with reasoning

---

## Core Features

| Area | Details |
|---|---|
| Real-time chat | WebSocket via Cloudflare Agents SDK (`/agents/research-agent/:id`) |
| Agent management | `agent_add`, `agent_remove`, `agent_list` commands |
| Web discovery | DuckDuckGo search + company site paths (`/about`, `/pricing`, etc.) + Wikipedia |
| Scraping | Jina Reader (`r.jina.ai`) with plain `fetch` HTML fallback |
| Profile extraction | Workers AI (`llama-3.3-70b-instruct-fp8-fast`) via Vercel AI SDK |
| Persistence | D1 (`company_agents` table) + Durable Objects (`ResearchAgentSQLite`) |
| Debate cap | Up to 6 company agents participate per debate |

---

## Architecture

```text
React UI (client/src/DebateApp.tsx)
  ├─ Left panel: Company Agents (queue, status, remove)
  └─ Right panel: Debate Chat (WebSocket, quick prompts)
           │ WebSocket
           ▼
Cloudflare Worker (src/index.ts)
  ├─ /agents/* → routeAgentRequest()
  └─ /*        → static assets (client/dist)
           ▼
ResearchAgentSQLite Durable Object (src/debateAgent.ts)
  ├─ Agent commands: agent_add / agent_remove / agent_list
  ├─ Profile build: discover sources → scrape → LLM JSON extraction
  ├─ Debate flow: openings → cycle 1 → cycle 2 → cycle 3 → judge → tie-break
  ├─ D1 storage: company_agents (company, url, profile_json, updated_at)
  └─ Workers AI: profile extraction, rebuttals, judging
```

### Company profile schema

Each agent stores a JSON profile with:

- `overview`, `revenue`, `keyMetrics`, `strengths`, `weaknesses`
- `evidence` — list of source URLs used during extraction

### Debate scoring rubric

The judge weighs:

- Evidence quality and specificity (40%)
- Strength of rebuttal against rivals (40%)
- Internal consistency and uncertainty handling (20%)

---

## API / Event Flow (WebSocket)

Connect to: `wss://<host>/agents/research-agent/default`

### Client → Agent

| Type | Payload | Purpose |
|---|---|---|
| `agent_add` | `content: "<url or domain>"` | Queue company agent creation |
| `agent_remove` | `target: "<company or domain>"` | Remove a stored agent |
| `agent_list` | — | Fetch all ready agents |
| `message` | `content: "<debate question>"` | Run a multi-agent debate |

### Agent → Client

| Type | When |
|---|---|
| `agent_add_started` | Scrape/profile build begins |
| `agent_added` | Agent ready (includes company, url, revenue, evidenceSources) |
| `agent_add_failed` | Invalid input or scrape failure |
| `agent_removed` | Agent deleted from D1 |
| `agents_list` | Response to `agent_list` |
| `message` | Chat content (user/assistant) |
| `tool_start` | Progress indicators (`addCompanyAgent`, `sourceDiscovery`, `debateRound`, `judgeVerdict`) |
| `done` | Request finished |
| `error` | Unhandled failure |

Debate responses arrive as **two assistant messages**: one with opening statements + rebuttal cycles + scoreboard, and a second with the final verdict.

---

## Tech Stack

**Worker**

- Cloudflare Workers + Wrangler 4
- [Cloudflare Agents SDK](https://agents.cloudflare.com) (`AIChatAgent`)
- [Vercel AI SDK](https://sdk.vercel.ai) + `workers-ai-provider`
- D1, Durable Objects, Workers AI

**Client**

- React 19 + Vite
- `react-markdown` for assistant output
- `lucide-react` icons
- Dark theme (`client/src/index.css`)

---

## Running Locally

### Prerequisites

- Node.js 18+
- [Cloudflare account](https://dash.cloudflare.com) (free tier works)
- Wrangler CLI (installed via project devDependencies)
- Logged in: `wrangler login`

### 1) Install dependencies

```bash
npm install
cd client && npm install && cd ..
```

### 2) Start dev

```bash
npm run dev
```

This runs `wrangler dev`, which builds the client and serves the Worker + React app together.

Optional frontend-only dev server (UI only; needs a running Worker for WebSocket):

```bash
npm run client:dev
```

### 3) Build and deploy

```bash
npm run client:build
npm run deploy
```

Dry-run deploy (no upload):

```bash
npx wrangler deploy --dry-run
```

---

## Environment and Bindings

Configured in `wrangler.jsonc`:

| Binding | Resource | Used in debate flow |
|---|---|---|
| `AI` | Workers AI | Yes — profiles, rebuttals, judging |
| `DB` | D1 `competitor-research-db` | Yes — `company_agents` table |
| `RESEARCH_AGENT` | Durable Object `ResearchAgentSQLite` | Yes — agent state + orchestration |
| `ASSETS` | `client/dist` static files | Yes — serves React UI |
| `VECTORIZE` | `competitor-research-index` | Configured; not used in current debate flow |

To provision resources from scratch:

```bash
wrangler d1 create competitor-research-db
# Copy database_id into wrangler.jsonc

wrangler vectorize create competitor-research-index --dimensions=768 --metric=cosine
```

---

## Project Structure

```text
src/
  index.ts              # Worker entry, CORS, routeAgentRequest, static assets
  debateAgent.ts        # Durable Object: agent mgmt + debate orchestration
client/
  src/
    DebateApp.tsx       # Split-screen UI (current app entry)
    App.tsx             # Previous competitive-research UI (kept in repo)
    main.tsx            # Mounts DebateApp
    index.css           # Theme tokens and global styles
wrangler.jsonc          # Worker bindings, D1, DO migrations, build hook
PROMPTS.md              # AI prompts used during development (assignment artifact)
package.json
client/package.json
README.md
```

---

## Example Usage

1. **Add agents** on the left panel:
   - `google.com`
   - `stripe.com`
   - `morganstanley.com`
2. **Wait** until all show **Ready** (revenue and source count appear on each card).
3. **Ask** a comparative question on the right:
   - `Which company has the best revenue?`
   - `Which company has better work-life balance?`
   - `Which company has stronger long-term growth potential?`
4. **Review** the two assistant responses:
   - Debate rounds: openings, 3 rebuttal cycles, confidence scoreboard
   - Final verdict: winner, confidence, reasoning, ranking

---

## Development Notes

- `PROMPTS.md` documents the AI prompts used while building the original competitive-research version of this project.
- `client/src/App.tsx` is the legacy single-agent chat UI; `DebateApp.tsx` is the active entry point mounted from `main.tsx`.
- Durable Object migrations in `wrangler.jsonc` include both `ResearchAgent` (v1) and `ResearchAgentSQLite` (v2).
