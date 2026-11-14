# Content AI Agent

A chat-based marketing assistant built for a startup's marketing team. You tell it what you need in plain English, and it writes the post, scores it against the brand guidelines, turns it into other formats, plans a content calendar, or makes a graphic in Canva.

It is a small multi-agent system on Gemini. One orchestrator reads your message, works out which agents to run, and passes the output of one to the next.

## What it can do

- **Draft** content for LinkedIn, X (Twitter), Instagram, Email, Blog, or graphic copy (headline, supporting text, CTA).
- **Review** a draft and get a 0-100 score across tone, human voice, clarity, CTA, grammar and SEO basics.
- **Repurpose** a piece into a LinkedIn post, X thread, Instagram caption, email summary, carousel copy, or quote card.
- **Plan** a content calendar and download it as an `.xlsx` file.
- **Design** a graphic in Canva sized for the channel, then edit its text with follow-up messages.
- **Chain** steps in one message, e.g. "Write a LinkedIn post, check the tone, and make a graphic for it."
- **Auto-revise**: ask it to "keep working on it until it passes" and it loops generate, review, revise (max 3 attempts).
- **Multi-channel**: ask for a LinkedIn post and an Instagram caption together and each one gets its own draft card.
- **Follow-ups** like "make it shorter" or "now do one for Instagram" work, because the conversation is stored.

Drafts stream into the page as they are generated.

## How it works

```
Browser (frontend/)
   │   POST /api/chat  (streamed, newline-delimited JSON)
   ▼
Flask (backend/app.py)
   ▼
Orchestrator (orchestrator.py)
   │  1. Router (Gemini, JSON schema) turns the message into intents + a brief
   │  2. Fills in missing fields from the conversation, or asks you for them
   │  3. Runs the agents in order, feeding each one's output to the next
   ▼
Generator · Reviewer · Repurposer · Planner · Designer
```

| Agent | File | What it does |
|---|---|---|
| Orchestrator | `orchestrator.py` | Routes requests, keeps conversation state, assembles the reply. Never writes content itself. |
| Generator | `generator.py` | Writes drafts using `brand_voice.md` and `product.pdf`. Enforces per-channel length limits. |
| Reviewer | `reviewer.py` | Scores a draft against the brand voice doc. Returns structured JSON with a pass or needs_work verdict. |
| Repurposer | `repurposer.py` | Converts existing content into another format. |
| Planner | `planner.py` | Builds a calendar of slots (date, channel, pillar, topic, angle, tone, ICP, CTA). Plans only, does not draft. |
| Designer | `designer.py`, `canva_auth.py` | Drives Canva's MCP server to create, resize, clean up and export a design, and to edit its text later. |

Supporting files: `exporter.py` (calendar to `.xlsx`, no model call), `product_context.py` (reads `product.pdf`, strips email addresses), `tracing.py` (optional LangSmith tracing).

## Project structure

```
.
├── backend/
│   ├── app.py               Flask app and API routes
│   ├── orchestrator.py      Router, sub-agent registry, session memory
│   ├── generator.py
│   ├── reviewer.py
│   ├── repurposer.py
│   ├── planner.py
│   ├── designer.py          Canva design agent
│   ├── canva_auth.py        Canva OAuth + token storage
│   ├── exporter.py          Calendar to .xlsx
│   ├── product_context.py   Loads product.pdf
│   └── tracing.py           Optional LangSmith tracing
├── frontend/
│   ├── index.html
│   ├── script.js
│   └── style.css
├── brand_voice.md           The brand voice doc every agent is prompted with
├── product.pdf              Internal product handover doc (grounding for drafts and plans)
├── Marketing-Comms.pdf      Source material the brand voice doc was written from
├── requirements.txt
└── .env.example
```

## Requirements

- Python 3.11 or newer
- A [Gemini API key](https://aistudio.google.com/apikey)
- A MongoDB database (Atlas works). See the note below.
- A Canva account with MCP access, only if you want the design agent

**MongoDB is needed for chat to work properly.** Conversations and the Canva token are stored there. If `MONGO_URI` is not set, nothing is saved between turns, so follow-ups like "make it shorter" will not work and the sidebar will stay empty. Only the Canva token has a local-file fallback.

## Setup

```bash
git clone https://github.com/iam-ogomide/Content-AI-Agent.git
cd Content-AI-Agent

python -m venv .venv
source .venv/bin/activate        # Windows: .venv\Scripts\activate
pip install -r requirements.txt

cp .env.example .env             # then fill in your keys
```

Keep `.env` in the project root, next to `.env.example`.

### Environment variables

Required:

| Variable | Purpose |
|---|---|
| `GEMINI_API_KEY` | Gemini API key |
| `MONGO_URI` | MongoDB connection string. Stores conversations and the shared Canva token, so treat it like a credential. |
| `MONGO_DB` | Database name |

Optional (all have defaults in the code):

| Variable | Default | Purpose |
|---|---|---|
| `GEMINI_MODEL` | `gemini-3.1-flash-lite` (`gemini-2.5-flash` for the Designer) | Model used by every agent. Setting it overrides both defaults. |
| `CANVA_REDIRECT_URI` | `http://localhost:8765/callback` | OAuth callback. Must also be allowed on your Canva app. |
| `CANVA_AUTH_TIMEOUT` | `300` | Seconds to wait for the browser login before giving up |
| `CANVA_MCP_URL` | `https://mcp.canva.com/mcp` | Canva MCP endpoint |
| `CANVA_READ_TIMEOUT` | `300` | Read timeout in seconds for Canva calls |
| `LANGSMITH_TRACING` | `false` | Set to `true` to trace Gemini calls |
| `LANGSMITH_API_KEY` | | Needed for tracing |
| `LANGSMITH_PROJECT` | | Project name that groups your traces |
| `LANGSMITH_ENDPOINT` | | Only for the EU region |

## Running it

The backend imports its own modules by bare name, so start it from inside `backend/`:

```bash
cd backend
python app.py
```

Open http://localhost:5000. Flask serves the frontend itself, so there is nothing separate to start.

Things to try:

- `Write a LinkedIn post about Income Insight for lenders, professional tone, under 200 words`
- `Write a blog post and turn it into an X thread`
- `Write an Instagram caption about Credit Insight and keep working on it until it passes review`
- `Plan LinkedIn and Email content for the next 2 weeks`
- `Make a LinkedIn visual for Spectrum`, then `change the headline to "Stop loan stacking"`

If a required detail is missing (topic, channel, tone), it asks instead of guessing.

## Canva setup

The design agent signs in to Canva over OAuth.

1. Run one design request on a machine that has a browser (or run `python designer.py` from `backend/`).
2. A browser tab opens. Log in with the Canva account the system should act as. Use a shared brand account, since every design lands there.
3. The token is saved to MongoDB, so other machines and deployed servers reuse it without opening a browser. Without Mongo it is saved to `backend/.canva_tokens.json` (git-ignored) and each machine has to sign in separately.
4. If the token is revoked, repeat step 1.

Designs are built from a text prompt each time, so visuals are only as consistent as the prompt keeps them. Autofilling a saved brand template needs Canva Enterprise.

The exported image link from Canva expires after roughly 18 hours. The `design_id` and edit link do not.

## API

| Method | Route | Purpose |
|---|---|---|
| `POST` | `/api/chat` | Main endpoint. Body: `{message, session_id}`. Streams newline-delimited JSON events (`text_start`, `text_chunk`, `text_done`, then a final `result`). |
| `GET` | `/api/chat/sessions` | List stored conversations, newest first |
| `GET` | `/api/chat/history?session_id=` | Full history for one conversation |
| `POST` | `/api/chat/reset` | Delete a conversation |
| `POST` | `/api/generate` | Draft directly: `topic`, `channel`, `tone`, optional `audience`, `cta`, `keyword` |
| `POST` | `/api/review` | Score a draft: `draft`, `channel`, optional `brief` |
| `POST` | `/api/repurpose` | `source_content`, `target_format`, optional `tone_shift`, `word_limit` |
| `POST` | `/api/plan` | `timeframe`, `channels`, optional `pillars`, `theme`, `icp`, `posts_per_week` |
| `POST` | `/api/plan/export` | Send a calendar, get back an `.xlsx` |

The UI uses `/api/chat`, the three `/api/chat/*` routes, `/api/review` and `/api/plan/export`. `/api/generate`, `/api/repurpose` and `/api/plan` skip the orchestrator and call one agent each, which is handy for testing.

## Making it your own

Everything brand-specific is in two files, so you can point this at a different company without touching the code:

- `brand_voice.md`: tone, vocabulary rules, audiences, content pillars, channel rules, compliance guardrails. Every agent reads it on every call.
- `product.pdf`: product facts and status (live, shipping, planned). Optional. If it is missing or unreadable, agents fall back to `brand_voice.md` alone. Email addresses in it are redacted before they reach the model.

Other places that hold company-specific values: the channel list and ICPs in `planner.py`, the channel limits in `generator.py`, and the Canva dimensions in `designer.py` (`CHANNEL_SPECS`).

## Tracing

Optional LangSmith tracing is off by default. Set `LANGSMITH_TRACING=true` and a `LANGSMITH_API_KEY` to turn it on. It records one span per Gemini call. Prompts and drafts are sent to LangSmith when it is on. Canva steps are not traced, so check the terminal for those errors.

## Known limitations

- No authentication. Anyone who can reach the server can use it and spend your Gemini and Canva quota.
- `app.py` runs Flask in debug mode on port 5000. Put it behind a production server such as gunicorn before deploying.
- Session storage ignores write errors on purpose, so a Mongo outage fails quietly rather than breaking a turn.
- Canva edits change text only. Requests to change colors, imagery or layout are not supported.
- The calendar planner caps X (Twitter) at 2-3 substantive slots a week by design.

## Tech stack

Python, Flask, Google Gemini (`google-genai`), MongoDB (`pymongo`), Canva MCP (`mcp`), `openpyxl`, `pypdf`, LangSmith (optional). Frontend is plain HTML, CSS and JavaScript with no build step.
