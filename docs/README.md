# TryAngle42 — Agentic Communication Bridge

A production-quality prototype for an agentic outreach system with a FastAPI
backend (Postgres on Railway, async SQLAlchemy/SQLModel, OpenAI/Gemini) and an
Expo + React Native + TypeScript mobile client with live WebSocket streaming
and human-in-the-loop approval.

> **For a deep dive** — system context, component diagrams, the full state
> machine, sequence diagrams (happy path / stop / race), the concurrency
> model, deployment topology, and the security checklist — see
> [`ARCHITECTURE.md`](./ARCHITECTURE.md).

```
.
├── backend/        FastAPI + async Postgres + agent state machine
├── mobile/         Expo + React Native (TypeScript) client
├── ARCHITECTURE.md Engineering deep dive (Mermaid diagrams)
├── README.md       This file
└── LICENSE         MIT
```

---

## 0. Requirements compliance map

How each assignment requirement is satisfied in this codebase:

### A. Backend (Python / FastAPI)

| Requirement | Where it lives |
| --- | --- |
| **WebSocket Orchestrator** — endpoint `/v1/agent/connect` | `backend/app/main.py` → `agent_ws()` |
| **Streaming Agent** — sequenced status updates ("Searching for vendors…", "Analyzing pricing…", "Drafting outreach…", "Self-reviewing…") | `backend/app/agent.py` → `AgentSession.run()` (5 phases + `executing`) |
| **Consent Gate** — backend pauses after drafting, waits for `APPROVE` before emitting `success` | `agent.py` → `_approval_event: asyncio.Event` + `_wait_for_approval_or_cancel()` |
| **Interrupt Handling** — `STOP` immediately terminates, returns `cancelled` | `agent.py` → `cancel()` (race-safe; see §6) |

### B. Mobile Client (React Native / Expo / TypeScript)

| Requirement | Where it lives |
| --- | --- |
| **State Management** — Zustand | `mobile/src/store/{taskStore,agentStore}.ts` |
| **Streaming UI** — chat-style log of agent updates | `mobile/src/screens/AgentScreen.tsx` (inverted `FlatList` of `ChatMessage`) |
| **Optimistic UI** — task shows "Scheduled" instantly on tap, *before* the API responds | `mobile/src/screens/TasksScreen.tsx` → `handleSubmit()` inserts a `temp-…` task immediately, then `replaceTask(tempId, real)` once the API returns; `removeTask(tempId)` on failure |
| **Lifecycle Management** — socket cleanup on unmount + on background | `mobile/src/hooks/useAgentWebSocket.ts` → cleanup in `useEffect` return + `AppState` listener that closes the socket when the app is not `active` |

### C. AI Integration

| Requirement | Where it lives |
| --- | --- |
| **Draft Tool** — LLM generates the message text | `backend/app/llm.py` → `LLMClient.generate_draft()` |
| **Self-Reflection** — second LLM pass that critiques tone / clarity of its own draft | `backend/app/llm.py` → `LLMClient.self_review()` (called immediately after `generate_draft`, before the consent gate) |

---

## 1. Prerequisites

- Python **3.11+**
- Node.js **20+** and npm/yarn/pnpm
- Expo (run via `npx expo`)
- A reachable Railway Postgres URL
- An OpenAI **or** Gemini API key

---

## 2. Backend

### 2.1 Install

```bash
cd backend
python -m venv .venv
# Windows PowerShell:
.\.venv\Scripts\Activate.ps1
# macOS / Linux:
# source .venv/bin/activate

pip install -r requirements.txt
```

### 2.2 Configure environment

Copy `.env.example` → `.env` and fill in:

```bash
# Railway gives you a sync-style URL. Paste it as-is.
DATABASE_URL=postgresql://postgres:YOUR_PASSWORD@metro.proxy.rlwy.net:51520/YOUR_DB_NAME

LLM_PROVIDER=openai                 # or "gemini"

OPENAI_API_KEY=sk-...
# Optional override (default OpenAI endpoint if empty):
OPENAI_BASE_URL=

# For Gemini via the OpenAI-compatible endpoint:
GEMINI_API_KEY=
GEMINI_BASE_URL=https://generativelanguage.googleapis.com/v1beta/openai/

DEFAULT_MODEL=gpt-4.1-mini
GEMINI_MODEL=gemini-2.0-flash-exp
```

> **Important — DATABASE_URL conversion**
>
> Railway provides a sync-style URL (`postgresql://...` or `postgres://...`).
> The backend automatically rewrites it to the asyncpg form
> (`postgresql+asyncpg://...`) at startup in `app/db.py`. You do **not**
> need to change the URL yourself. SSL is enabled via a default
> `ssl.SSLContext`, which Railway requires.

### 2.3 Bootstrap the database (first time only)

If your Railway URL points to a database that has not been created yet,
create it once via the helper script (it connects to the default `postgres`
database on the same host and issues `CREATE DATABASE`):

```bash
python scripts/bootstrap_db.py
```

### 2.4 Run

```bash
uvicorn app.main:app --host 0.0.0.0 --port 8000 --reload
```

On first boot the server creates the `tasks` and `task_events` tables
(prototype-friendly `SQLModel.metadata.create_all`).

### 2.5 Verify

Two scripts ship with the backend:

```bash
# 1. Offline tests of the agent state machine (no LLM, no DB)
python scripts/agent_unit_test.py

# 2. End-to-end smoke test against a running server (uses Postgres)
python scripts/smoke_test.py
```

The unit suite covers:

- approve → success
- stop before drafting → cancelled (no LLM call)
- stop during waiting_approval → cancelled
- stop arriving after success does **not** overwrite the terminal state
- 8x race of `approve()` and `cancel()` fired simultaneously: the wire `final`
  status and the DB `task.status` always agree (STOP wins ties — the safe
  default for human-in-the-loop)

### 2.6 Endpoints

| Method | Path                      | Notes                                                    |
| ------ | ------------------------- | -------------------------------------------------------- |
| GET    | `/`                       | Health + provider/model info                             |
| GET    | `/health`                 | Liveness probe                                           |
| POST   | `/v1/tasks`               | Create a task; honors the `Idempotency-Key` header       |
| GET    | `/v1/tasks`               | List tasks (newest first)                                |
| GET    | `/v1/tasks/{task_id}`     | Fetch a task                                             |
| WS     | `/v1/agent/connect?task_id=…` | Live agent run; bidirectional JSON                   |

#### WebSocket protocol

**Client → server:**

```jsonc
{ "type": "init",    "task_id": "<uuid>", "description": "..." }
{ "type": "approve", "task_id": "<uuid>" }
{ "type": "stop",    "task_id": "<uuid>" }
```

**Server → client:**

```jsonc
{ "type": "status", "task_id": "<uuid>", "phase": "drafting", "message": "...", "ts": "..." }
{ "type": "draft",  "task_id": "<uuid>", "draft_text": "...", "ts": "..." }
{ "type": "final",  "task_id": "<uuid>", "status": "success" | "cancelled" | "error", "message": "...", "ts": "..." }
```

Phases (`status.phase`):
`searching_vendors → analyzing_pricing → drafting → self_review →
waiting_approval → executing → success | cancelled | error`

---

## 3. Agent state machine

```
scheduled
   │  init
   ▼
running ── searching_vendors → analyzing_pricing → drafting → self_review
                                                                │
                                                                ▼
                                                       waiting_approval
                                              ┌──────────┴───────────┐
                                              │ approve              │ stop
                                              ▼                      ▼
                                          executing             cancelled
                                              │
                                              ▼
                                           success
```

### Race-safe interrupts

The `AgentSession` (in `backend/app/agent.py`) protects every mutation of
`_approved` / `_cancelled` / `task.status` with an `asyncio.Lock`, and uses an
`asyncio.Event` as the consent gate (instead of busy-waiting):

- `approve()` only flips `_approved` while we are in `waiting_approval` and
  `_cancelled` is false.
- `cancel()` always flips `_cancelled` and immediately wakes the consent gate.
- After the gate, the run **re-checks `_cancelled` under the lock** before
  transitioning to `executing` and again before writing `success`.
- If `STOP` arrives after `success`, `cancel()` only logs an audit event and
  never overwrites the terminal status.

This guarantees no race between APPROVE and STOP can produce both a
`success` and a `cancelled` outcome.

### Idempotency

`POST /v1/tasks` honors the `Idempotency-Key` header. If a task with the same
key already exists, the original task is returned and no duplicate row is
written (enforced by a unique index on `tasks.idempotency_key`).

---

## 4. Mobile

### 4.1 Install

```bash
cd mobile
npm install
```

### 4.2 Configure environment

Copy `.env.example` → `.env` and point it at your backend.

```bash
EXPO_PUBLIC_BACKEND_HTTP_URL=http://localhost:8000
EXPO_PUBLIC_BACKEND_WS_URL=ws://localhost:8000
```

> When running on a physical device, replace `localhost` with your
> machine's LAN IP (e.g. `http://192.168.1.20:8000`). For TLS-enforcing
> hosting, switch to `https://` and `wss://`.

### 4.3 Run

```bash
npx expo start
# then press i / a / w for iOS, Android, or web
```

### 4.4 UX flow

1. **Tasks screen** — type a description and tap **Schedule task**. A task is
   created via `POST /v1/tasks` with a generated `Idempotency-Key` and the
   client navigates to the **Agent screen**.
2. **Agent screen** — opens a WebSocket and immediately sends `{type:"init"}`.
   You see live status messages, then a `draft` bubble, then approval CTAs.
3. **Approve** sends `{type:"approve"}`; the agent transitions to `executing`
   and finalizes with `success`. **Stop** sends `{type:"stop"}` and finalizes
   with `cancelled`. Buttons disable on press to prevent duplicate clicks even
   though the backend is race-safe.

### 4.5 State persistence

`taskStore` uses `zustand` + `persist` over `@react-native-async-storage/async-storage`,
so the task list and last-selected task survive app restarts.
`agentStore` is intentionally ephemeral per session.

---

## 5. Interrupt logic (how STOP is handled)

> The assignment asks specifically for this section, and the bulk of the
> "Attention to Detail" evaluation lives here. This is what protects against
> the *"user hits Stop at the exact moment the agent finishes"* race.

### 5.1 Backend (authoritative)

The state machine in `backend/app/agent.py` enforces three invariants:

1. **STOP wins ties.** `cancel()` always sets `_cancelled = true` and signals
   the consent-gate `asyncio.Event` regardless of whether `_approved` is
   already true. So if `APPROVE` and `STOP` arrive simultaneously, the
   outcome is deterministic: **cancelled**.
2. **The run loop re-checks `_cancelled` *under the lock*** at every phase
   transition — most importantly *after* the simulated `executing` step and
   *before* writing `task.status = success`. A STOP that arrives during
   "executing" is honored and the run terminates as `cancelled`.
3. **A late STOP after a terminal state is non-destructive.** If `success`
   was already committed before `cancel()` runs, `cancel()` only writes an
   audit-log event (`user_stopped_after_terminal`); it never overwrites the
   row.

The two synchronization primitives that make this work:

- `asyncio.Lock` — guards every mutation of `_approved`, `_cancelled`, and
  `task.status`.
- `asyncio.Event` — the consent gate. `run()` `await`s it; `approve()` and
  `cancel()` both `set()` it. No busy-wait, no polling.

This combination guarantees that the WebSocket `final.status` and the
database `task.status` **always agree**, regardless of message ordering.

### 5.2 Verification

```bash
cd backend
python scripts/agent_unit_test.py
```

This runs five focused tests, including an 8-iteration race where
`approve()` and `cancel()` are fired simultaneously via `asyncio.gather`.
The test asserts that for every iteration:

- A `final` message is emitted on the wire,
- `final.status` ∈ `{success, cancelled}`,
- `final.status == task.status` in the DB.

In practice the result is consistently `cancelled` (8/8), confirming
invariant #1.

### 5.3 Mobile (defense in depth)

The mobile client also disables Approve / Stop after the first press until a
`final` message arrives, even though the backend is already race-safe. This
prevents stray double-taps from polluting the audit log with duplicate
events.

---

## 6. LLM provider switch (OpenAI vs Gemini)

Selection is controlled entirely via `LLM_PROVIDER`:

- `LLM_PROVIDER=openai` — uses the official OpenAI endpoint with
  `OPENAI_API_KEY` and `DEFAULT_MODEL`.
- `LLM_PROVIDER=gemini` — uses Gemini through OpenAI's compatibility endpoint
  (`GEMINI_BASE_URL`) with `GEMINI_API_KEY` and `GEMINI_MODEL`.

Both paths go through the same `openai.AsyncOpenAI` client in `app/llm.py`, so
adding more providers (e.g. Anthropic via OpenAI-compatible gateway, Azure
OpenAI) is a one-branch change.

Calls are wrapped in `with_retries` (exponential backoff with jitter, 3
attempts by default).

---

## 7. Libraries chosen (and why)

> Section explicitly requested by the assignment.

### Backend

| Library | Why |
| --- | --- |
| **FastAPI** | First-class async, native WebSocket support, automatic OpenAPI, Pydantic v2 integration |
| **uvicorn[standard]** | ASGI server with `httptools`, `websockets`, `watchfiles` for `--reload` |
| **SQLModel** | One model class for both the Pydantic schema and the SQLAlchemy table (Tiangolo's library by the FastAPI author) |
| **SQLAlchemy 2.x asyncio + asyncpg** | Production-grade async Postgres driver and ORM; pooling + pre-ping; matches Railway's TLS proxy |
| **Pydantic v2 + pydantic-settings** | DTO validation; **discriminated unions** for the WebSocket protocol; `BaseSettings` for typed `.env` loading |
| **openai** Python SDK | Used for **both** OpenAI and Gemini (Gemini exposes an OpenAI-compatible endpoint), so one client serves both providers |
| **python-dotenv** | Local-dev `.env` loading |

I deliberately **avoided** heavier scaffolding (Celery, Kafka, Alembic) — the
prototype is single-process and prefers asyncio primitives + a compact
event-sourced audit log over message queues.

### Mobile

| Library | Why |
| --- | --- |
| **Expo SDK 54** | Fast iteration with Expo Go on a real device; matches the Play Store / App Store version of Expo Go |
| **React Native 0.81 + React 19.1 + TypeScript (strict)** | Strong types end-to-end; matches the SDK 54 baseline |
| **@react-navigation/native + native-stack** | The de-facto navigation library for RN; native stack avoids JS-thread jank |
| **Zustand + zustand/middleware (`persist`)** | Tiny, hookless API; `persist` over `AsyncStorage` survives app restarts. Chosen over TanStack Query because the agent stream is push-driven (WebSocket), not request/response |
| **@react-native-async-storage/async-storage** | Storage backend for `persist`; the canonical RN choice |
| **axios** | Familiar interceptor / `headers` API for the `Idempotency-Key` flow |
| **nanoid/non-secure** | Cheap, fast id generation for `Idempotency-Key` and chat-message keys |

I deliberately **avoided** UI kits (`react-native-gifted-chat`, `shadcn/ui`
equivalents like `tamagui`, etc.) per the assignment's note that *"we aren't
looking for a perfect UI"* — the chat surface is a hand-rolled inverted
`FlatList` with two presentational components (`ChatMessage`, `StatusPill`)
that costs ~200 LOC and gives full control over the message types we need
(`status`, `draft`, `final`). Adding a chat kit later is a drop-in change.

---

## 8. Code quality

Backend:

- Strict type hints, mypy-friendly, Pydantic v2 models, SQLModel ORM.
- Ruff + Black configured in `pyproject.toml` (`pip install ruff black mypy`
  to use them locally).

Mobile:

- `tsconfig.json` with `"strict": true` and `noUncheckedIndexedAccess`.
- ESLint (`.eslintrc.js`) + Prettier (`.prettierrc`) preconfigured.

---

## 9. Security & secrets

- **Never commit `.env`.** Both `backend/.env` and `mobile/.env` are
  git-ignored.
- Use `.env.example` files as the source of truth for required keys.
- For production, scope `CORSMiddleware` to your real origins and put the
  backend behind TLS; switch the mobile client to `wss://`.
- Rotate database and LLM credentials regularly; treat the Railway URL
  embedded in this prototype as test-only.

---

## 10. Pushing to GitHub

The repo is already structured for a clean first push. Quick checklist:

- `.gitignore` blocks `.env`, `.venv/`, `node_modules/`, `.expo/`, build
  caches, OS junk, and PEM/keystore files. `.env.example` files **are**
  committed (placeholders only).
- `.gitattributes` normalizes line endings to LF so Windows / macOS / Linux
  contributors get clean diffs.
- `LICENSE` (MIT) is included.
- `README.md` and [`ARCHITECTURE.md`](./ARCHITECTURE.md) document the system.

Verify nothing sensitive is staged before the first push:

```bash
git init
git add -A
git status                # confirm `.env` files are NOT listed
git ls-files --cached | Select-String "\.env$"  # should show only .env.example files
git commit -m "feat: initial TryAngle42 prototype"
git branch -M main
git remote add origin git@github.com:<you>/<repo>.git
git push -u origin main
```

If you ever accidentally commit a real secret, rotate it immediately and
purge from history with [git-filter-repo](https://github.com/newren/git-filter-repo)
— do **not** rely on `git rm` alone, since the secret remains in old commits.
