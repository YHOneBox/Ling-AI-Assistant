# Ling: Build Guide

Ling is a local-first AI assistant. It works like the Claude or Gemini apps (chat, streaming replies, saved conversations, model picker), but the user brings their own model: an API key for a hosted provider, or a model running on their own machine. No accounts and no Ling server. Everything runs and stays on the user's computer.

GitHub description: "A local-first AI assistant. Bring your own model, through an API key or running locally, and grow it with plugins for files, voice, and tools."

This file has three jobs:

1. Tell Cursor how to teach (Part 1).
2. Hold the decisions and the full stage 1 spec (Parts 2 to 4).
3. Give the step-by-step lesson plan and the progress log (Parts 5 and 6).

---

## How to use this file

Save it as `docs/LING_GUIDE.md` in the repo. In Cursor's chat (Agent mode), start with:

```text
@docs/LING_GUIDE.md Read Part 1 and follow it. Start Lesson 0.1.
```

After that, say `next` to move on, or ask questions at any point. Cursor updates the progress log in Part 6 as lessons finish, so a new chat can pick up where you left off:

```text
@docs/LING_GUIDE.md Check the progress log and continue from the next unchecked lesson.
```

---

## Part 1: Instructions for Cursor (teaching mode)

You are my tutor and pair programmer for building Ling. I want to learn how to build this system, not just end up with code. Follow these rules for the whole project.

**Pace**

- Teach one lesson at a time, in the order of Part 5. Never start the next lesson until I say `next`.
- Before starting, check the progress log in Part 6 and tell me which lesson we're on.
- If a lesson is big, split it into small steps and stop after each step so I can try it.

**Each lesson goes like this**

1. **Explain first.** In plain English, say what we're building in this lesson, why Ling needs it, and where it fits in the architecture. Keep it short: a few paragraphs, not an essay.
2. **Teach the new concepts.** For each item in the lesson's "Concepts" list, give a small example that isn't Ling code, then show how it applies to Ling.
3. **Build.** Each lesson says who writes the code:
   - **You write:** I type the code. Give me the file name, the function signatures and what each part must do, then let me try. Review my code when I'm done, point out problems, and explain why. Only show a full solution if I ask or I've been stuck twice.
   - **Cursor writes, you review:** you write the code, then walk me through it file by file. Explain every line that uses a concept I haven't seen yet.
4. **Check.** Run (or tell me to run) the "Done when" checks. A lesson isn't finished until they pass.
5. **Quiz.** Ask me 2 or 3 short questions about what I just built. If I get one wrong, explain it again in a different way.
6. **Commit.** Suggest a commit message in Conventional Commits style (`feat(chat): ...`, `test: ...`, `chore: ...`) and tell me the git commands.
7. **Log.** Tick the lesson in Part 6 and add one line under "Notes" with anything we changed from the spec.

**Code rules**

- Part 4 (the stage 1 spec) is the source of truth. If something I ask for conflicts with it, stop and say so before writing code.
- Only add dependencies listed in the spec. Ask before adding anything else, and explain why it's needed.
- Only files in `backend/src/ling/providers/` may import `openai` or `anthropic`.
- Never log, return or store an API key. Never log request bodies or message content.
- Every new API route gets Pydantic request/response models and at least one test.
- Every database change gets an Alembic migration. Never edit a migration that's already committed.
- After changing any API schema, run `make gen-types`. Never hand-edit `frontend/src/api/schema.d.ts`.
- Tests use the FakeProvider and never call real APIs.
- Backend: async everywhere, type hints everywhere, pyright strict on `src/`.
- Frontend: TypeScript strict, no `any`. Server data goes in TanStack Query; the live stream lives only in the Zustand store.
- Before saying a lesson is done, run `make lint test` and show me the result.

**Environment**

- I work on Windows with WSL (Ubuntu). The project lives inside WSL at `~/projects/ling-assistant`, and every command runs in the WSL bash terminal, never PowerShell.
- If a command differs between macOS, Windows and Linux, give me the Linux version only.

**Style**

- Explain in simple English. My first language isn't English, so prefer short sentences and define jargon the first time you use it.
- When I make a mistake, tell me directly what's wrong and why. Don't just fix it silently.
- If you think a decision in this spec is wrong, say so and explain. I'd rather learn why than follow a bad plan.

---

## Part 2: Decisions

- **Bring your own model (BYOM).** Users add an API key for a hosted provider (OpenAI, Anthropic, and later others), or point Ling at a model server running on their own machine (Ollama or LM Studio). No Ling accounts.
- **Keys** live in the OS keychain, are read only when calling a provider, and are never logged or returned by the API.
- **Stack:** FastAPI (Python) backend, React + TypeScript frontend, SQLite for storage. One repo with `backend/` and `frontend/`. A desktop wrapper (Tauri or Electron) comes in stage 6.
- **Designed for growth from day one:** one model-provider interface, one context-building step that retrieval plugs into, a tool registry (stage 5), and an eval script that runs on every change.
- **Local models early.** The OpenAI adapter takes a custom server address (`base_url`), and Ollama and LM Studio both speak the OpenAI API. So local models work in stage 1 with almost no extra code.

---

## Part 3: Roadmap

| Stage | What gets built |
| --- | --- |
| 1 | Text chat: streaming replies, Stop button, saved conversations, model picker (hosted and local), system prompt, error handling |
| 2 | RAG over a local folder, with citations |
| 3 | Evaluation set, then hybrid search, reranking and query rewriting |
| 4 | Voice in and out, streamed |
| 5 | Tools and agents: retrieval becomes a `search_my_files` tool, plus web search and MCP |
| 6 | Desktop app, and a VRM avatar with lip sync (three-vrm) |
| 7 | Tracing, caching, model routing |
| 8 | MCP support, custom skins, plugins |
| 9 | Built-in local runtime (no separate Ollama needed) and a small fine-tuned model |

**About local models:** before picking a model to run locally, check your GPU (VRAM) and RAM. A model that doesn't fit in memory will be very slow or won't load. Choose models based on your actual hardware, not on benchmark lists.

---

## Part 4: Stage 1 spec (text chat)

### 4.1 Scope and definition of done

Stage 1 ships a local text chat. It streams replies from OpenAI, Anthropic or a local model server, saves every conversation to SQLite, and can be stopped mid-reply. It ends with tag `v0.1.0`.

**In scope**

- Streaming replies over SSE, with a Stop button that keeps the partial reply
- Saved conversations: list, open, rename, delete
- Model picker per conversation (provider + model), with a free-text "custom model id" fallback
- Local models through any OpenAI-compatible server (Ollama by default), with no API key needed
- A default system prompt in settings, which each conversation can override
- API key entry per provider, stored in the OS keychain, never returned by the API or logged
- Clear error states: missing key, bad key, rate limit, context too long, network, provider down, local server not running. Each has a retry.
- The provider interface, a fake provider for tests, the eval script v0 and CI

**Out of scope for stage 1** (each has a hook ready for later): file upload and RAG, tools and function calling, images, voice, the desktop wrapper, titles written by an LLM, token-accurate context trimming, syncing between windows.

**Not yet:** the retrieval interface and the tool registry. Stage 1 reserves the folders and two hooks (`build_context()` and the `StreamEvent` union) so stages 2 and 5 plug in without a rewrite. An interface designed with no real implementation behind it usually has to be redesigned once the first one arrives.

**Definition of done**

- [ ] Fresh clone to a working chat in under 5 minutes using the README (`make setup`, `make dev`)
- [ ] The first token shows in the UI within 300 ms of the provider sending it (no buffering in the backend)
- [ ] Stop works mid-stream; the partial reply is saved with status `stopped` and survives a page reload
- [ ] Killing the backend mid-stream leaves no message stuck in `streaming` after restart
- [ ] Chat works with a local model through Ollama with no API key
- [ ] A test proves no API key appears in logs, API responses or the SQLite file
- [ ] Every error code has a UI message and a working retry
- [ ] CI is green: lint, type check, tests and the offline eval, for backend and frontend
- [ ] README has a short GIF, an architecture diagram and the roadmap

### 4.2 Stack

| Layer | Choice | Why |
| --- | --- | --- |
| Python | 3.12, managed with `uv` | Fast installs, one lockfile; uv creates a normal `.venv` |
| API | FastAPI + uvicorn, Pydantic v2 | Typed models also generate the OpenAPI spec the frontend uses |
| DB access | SQLAlchemy 2.0 async + `aiosqlite`, Alembic migrations | The schema will change every stage, so migrations from day one |
| Providers | Official `openai` and `anthropic` async SDKs | Retries and typed stream events. The `openai` SDK also talks to Ollama and LM Studio |
| Keys | `keyring` | macOS Keychain, Windows Credential Manager, Linux Secret Service |
| Paths | `platformdirs` | The DB goes in the OS app-data folder, not the repo |
| Lint and types | `ruff`, `pyright` | |
| Tests | `pytest`, `pytest-asyncio`, `httpx`, `respx` | `respx` replays recorded provider streams |
| Frontend | Vite, React, TypeScript strict | |
| Server state | TanStack Query | Caching and refetching for lists |
| UI state | Zustand | The in-flight stream lives here |
| Streaming client | `@microsoft/fetch-event-source` | SSE over POST, with abort support |
| Styling | Tailwind + a few shadcn/ui parts | |
| Markdown | `react-markdown`, `remark-gfm`, `rehype-highlight` | Raw HTML stays disabled |
| API types | `openapi-typescript` | Generated from FastAPI, never hand-written |
| Frontend tests | Vitest, Playwright (one end-to-end test) | |

### 4.3 Folder structure

```text
ling-assistant/
├── backend/
│   ├── pyproject.toml
│   ├── alembic.ini
│   ├── migrations/versions/0001_initial.py
│   ├── src/ling/
│   │   ├── main.py              # create_app(): middleware, routers, lifespan
│   │   ├── config.py            # settings from env (LING_DATA_DIR, LING_PORT, ...)
│   │   ├── logging.py           # JSON logs + key redaction filter
│   │   ├── api/
│   │   │   ├── deps.py          # get_session, get_registry, get_keystore
│   │   │   ├── errors.py        # LingError -> JSON error body
│   │   │   └── routes/
│   │   │       ├── health.py
│   │   │       ├── providers.py # keys, base URLs, test
│   │   │       ├── models.py
│   │   │       ├── conversations.py
│   │   │       ├── chat.py      # streaming endpoints
│   │   │       ├── generations.py # cancel
│   │   │       └── settings.py
│   │   ├── schemas/             # Pydantic request/response models
│   │   ├── core/
│   │   │   ├── chat_service.py  # runs one turn: save, stream, persist
│   │   │   ├── context.py       # build_context(): history -> ChatRequest (RAG hooks in here in stage 2)
│   │   │   ├── generations.py   # registry of in-flight runs + cancel events
│   │   │   └── sse.py           # event encoding + heartbeat
│   │   ├── providers/
│   │   │   ├── types.py         # ChatMessage, ChatRequest, StreamEvent, ModelInfo
│   │   │   ├── errors.py        # ProviderError + codes
│   │   │   ├── base.py          # ModelProvider abstract class
│   │   │   ├── registry.py
│   │   │   ├── openai_compat.py # OpenAI, Ollama, LM Studio, any OpenAI-compatible server
│   │   │   ├── anthropic.py
│   │   │   └── fake.py          # deterministic, for tests, evals and e2e
│   │   ├── secrets/keystore.py  # keyring wrapper + env fallback
│   │   ├── db/
│   │   │   ├── engine.py        # engine, PRAGMAs, session factory
│   │   │   ├── models.py        # SQLAlchemy tables
│   │   │   └── repos.py         # ConversationRepo, MessageRepo, SettingsRepo
│   │   ├── retrieval/           # empty in stage 1 (README.md only)
│   │   └── tools/               # empty in stage 1 (README.md only)
│   └── tests/
│       ├── conftest.py          # app with FakeProvider + temp DB
│       ├── fixtures/streams/    # recorded SSE from OpenAI and Anthropic
│       └── test_*.py
├── frontend/
│   ├── package.json
│   ├── vite.config.ts           # proxy /api -> 127.0.0.1:8765
│   └── src/
│       ├── main.tsx, App.tsx, routes.tsx
│       ├── api/
│       │   ├── schema.d.ts      # generated
│       │   ├── client.ts        # typed fetch wrapper
│       │   └── stream.ts        # streamChat(): SSE -> typed callbacks
│       ├── stores/chatStream.ts # Zustand: active run, draft text, status
│       ├── features/
│       │   ├── conversations/   # Sidebar, ConversationItem, hooks
│       │   ├── chat/            # ChatView, MessageList, Message, Composer, ErrorBanner
│       │   ├── models/          # ModelPicker
│       │   └── settings/        # SettingsPage, ApiKeyForm, LocalServerForm, SystemPromptEditor
│       ├── components/ui/       # shadcn parts
│       └── lib/                 # format, scroll, keyboard helpers
├── evals/
│   ├── run.py
│   ├── cases/chat_core.yaml
│   └── results/                 # gitignored, one JSON per run
├── docs/
│   ├── LING_GUIDE.md            # this file
│   └── architecture.md
├── .cursor/rules/ling.mdc
├── .github/workflows/ci.yml
├── Makefile                     # setup, dev, test, eval, gen-types, lint, e2e
└── README.md
```

The backend listens on `127.0.0.1:8765`. In development, Vite forwards `/api` to it, so the browser only ever talks to one address.

### 4.4 Provider interface

Every model goes through one async interface that yields the same kind of events (`StreamEvent`), whatever the provider. Nothing outside `providers/` imports `openai` or `anthropic`.

**Providers in stage 1**

| id | Class | Key | Default base URL |
| --- | --- | --- | --- |
| `openai` | `OpenAICompatProvider` | Required | OpenAI's API |
| `anthropic` | `AnthropicProvider` | Required | Anthropic's API |
| `local` | `OpenAICompatProvider` | Optional | `http://localhost:11434/v1` (Ollama). The user can change it, e.g. LM Studio's `http://localhost:1234/v1` |
| `fake` | `FakeProvider` | None | Only registered when `LING_ENABLE_FAKE=1` |

Because `OpenAICompatProvider` takes a `base_url`, more providers that offer an OpenAI-compatible endpoint (OpenRouter, Groq, Google Gemini) can be added later as config entries, not new classes. Check each provider's docs for the exact URL when adding it.

```python
# providers/types.py
from typing import Literal, Annotated
from pydantic import BaseModel, Field

Role = Literal["system", "user", "assistant"]  # "tool" added in stage 5

class TextPart(BaseModel):
    type: Literal["text"] = "text"
    text: str
# Stage 4+: ImagePart, AudioPart. Stage 5: ToolCallPart, ToolResultPart.
ContentPart = Annotated[TextPart, Field(discriminator="type")]

class ChatMessage(BaseModel):
    role: Role
    content: list[ContentPart]

class ChatRequest(BaseModel):
    model: str
    messages: list[ChatMessage]        # no system role in here
    system_prompt: str | None = None   # each adapter places it the way its API wants
    temperature: float | None = None
    max_output_tokens: int | None = None
    # stage 5: tools: list[ToolSpec] = []

class ModelInfo(BaseModel):
    provider_id: str
    id: str
    display_name: str
    context_window: int | None = None

class TextDelta(BaseModel):
    type: Literal["text_delta"] = "text_delta"
    text: str

class Usage(BaseModel):
    type: Literal["usage"] = "usage"
    input_tokens: int | None = None
    output_tokens: int | None = None

class Done(BaseModel):
    type: Literal["done"] = "done"
    finish_reason: Literal["stop", "length", "cancelled", "other"]

# Stage 5 adds ToolCallDelta here; the chat service ignores unknown types until then.
StreamEvent = Annotated[TextDelta | Usage | Done, Field(discriminator="type")]
```

```python
# providers/errors.py
class ProviderErrorCode(StrEnum):
    MISSING_KEY = "missing_key"            # no key stored or in env
    AUTH_FAILED = "auth_failed"            # 401/403
    RATE_LIMITED = "rate_limited"          # 429, carries retry_after_s when given
    CONTEXT_TOO_LONG = "context_too_long"
    MODEL_NOT_FOUND = "model_not_found"    # 404 or unknown model
    BAD_REQUEST = "bad_request"            # other 400s
    PROVIDER_UNAVAILABLE = "provider_unavailable"  # 5xx, overloaded
    LOCAL_SERVER_DOWN = "local_server_down"        # connection refused on a localhost base_url
    NETWORK = "network"                    # DNS, connect, read timeout
    UNKNOWN = "unknown"

class ProviderError(Exception):
    def __init__(self, code, message, *, retryable: bool, status: int | None = None,
                 retry_after_s: float | None = None): ...
```

`message` must be safe to show in the UI. Adapters write it themselves and never pass through raw SDK text, because that can echo request headers.

```python
# providers/base.py
class ModelProvider(ABC):
    id: ClassVar[str]               # "openai", "anthropic", "local", "fake"
    display_name: ClassVar[str]
    requires_key: ClassVar[bool]

    @abstractmethod
    async def list_models(self, api_key: str | None) -> list[ModelInfo]: ...

    @abstractmethod
    async def validate_key(self, api_key: str | None) -> None:
        """Cheap call (list models). Raises ProviderError on failure."""

    @abstractmethod
    def stream_chat(self, req: ChatRequest, api_key: str | None) -> AsyncIterator[StreamEvent]:
        """Yields TextDelta*, then an optional Usage, then exactly one Done.
        Raises ProviderError for any failure. Must stop quickly and close the
        HTTP stream when the caller stops reading (aclose / CancelledError)."""
```

```python
# providers/registry.py
class ProviderRegistry:
    def register(self, p: ModelProvider) -> None: ...
    def get(self, provider_id: str) -> ModelProvider: ...   # KeyError -> 404
    def all(self) -> list[ModelProvider]: ...
```

**Adapter notes**

- The API key is passed into each call, never stored on the provider object. Create the SDK client per call, or cache it keyed by a hash of the key. That way a changed key takes effect immediately, and keys don't sit in long-lived objects.
- Set the SDK's `max_retries=2` for connection errors only. Never retry after the first token has been sent; a retry would duplicate text.
- Anthropic needs `max_tokens`. Default it to 4096 when `max_output_tokens` is unset.
- OpenAI-compatible: use Chat Completions with `stream_options={"include_usage": True}`. Some local servers ignore that option, so treat usage as optional.
- `local` with no key: the `openai` SDK requires a key string, so pass a placeholder like `"not-needed"`.
- When `local` gets "connection refused", raise `LOCAL_SERVER_DOWN` with a message like "Can't reach the local model server at http://localhost:11434. Is Ollama running?"
- `list_models` comes from each provider's models endpoint, cached for 10 minutes (60 seconds for `local`, since users pull new models often). Don't hardcode model names; they go out of date. OpenAI's list includes embedding and audio models, so filter out ids containing `embedding`, `tts`, `whisper`, `dall-e`, `moderation`. The picker's custom id field covers anything the filter gets wrong.
- `FakeProvider` reads instructions from the last user message: `echo:<text>` streams the text back in 3-character chunks with a 20 ms delay, `error:<code>` raises that code, `slow:<n>` streams n chunks at 200 ms each (for testing Stop), `long` streams 2,000 tokens.

**Context building (the stage 2 hook)**

`core/context.py` has one function: `build_context(conversation, history, settings) -> ChatRequest`. It drops messages with status `error`, keeps `stopped` partial replies, applies the system prompt (the conversation's own, otherwise the default), and drops the oldest turns once the history passes about 24,000 characters. Stage 2 adds retrieved chunks here, and nothing else has to change.

### 4.5 API endpoints

All routes are under `/api`, take and return JSON, and return errors as `{"error": {"code", "message", "retryable"}}`. The two chat routes return `text/event-stream` instead.

| Method | Path | Body | Returns |
| --- | --- | --- | --- |
| GET | `/api/health` | | `{status, version, db: "ok"}` |
| GET | `/api/providers` | | `[{id, display_name, requires_key, configured, key_source: "keychain" \| "env" \| null, key_hint: "…a1b2" \| null, base_url}]` |
| PUT | `/api/providers/{id}/key` | `{api_key}` | Validates first, then stores. `{configured: true, key_hint}` or 400 `auth_failed` |
| DELETE | `/api/providers/{id}/key` | | 204 |
| PUT | `/api/providers/{id}/base-url` | `{base_url}` | Only for OpenAI-compatible providers. Saved in settings, not the keychain |
| POST | `/api/providers/{id}/test` | | `{ok: true}` or the provider error |
| GET | `/api/models` | `?provider=` optional | `[ModelInfo]` from configured providers only |
| GET | `/api/conversations` | `?limit=50&before=<updated_at>` | `{items: [ConversationSummary], next_before}` |
| POST | `/api/conversations` | `{provider_id?, model_id?, system_prompt?}` | `Conversation`; missing fields use the defaults from settings |
| GET | `/api/conversations/{id}` | | `Conversation` with `messages` ordered by `seq` |
| PATCH | `/api/conversations/{id}` | any of `{title, system_prompt, provider_id, model_id}` | `Conversation` |
| DELETE | `/api/conversations/{id}` | | 204, also deletes its messages |
| POST | `/api/conversations/{id}/messages` | `{content}` | SSE stream (4.6) |
| POST | `/api/conversations/{id}/retry` | | SSE stream; deletes the last assistant message if it's `error` or `stopped`, then generates again |
| POST | `/api/generations/{id}/cancel` | | 202. Safe to call twice; an unknown id also returns 202 |
| GET | `/api/settings` | | `{default_provider_id, default_model_id, default_system_prompt, temperature}` |
| PATCH | `/api/settings` | any subset | `Settings` |

**Title rule:** after the first user message, set the title to its first 60 characters, cut at a word boundary. LLM-written titles can come in stage 7, once model routing exists.

### 4.6 Streaming protocol (SSE)

One event per frame: an `event:` line plus a JSON `data:` line. A heartbeat comment `: ping` goes out every 15 seconds so idle connections don't get dropped.

```text
event: start
data: {"generation_id":"…","conversation_id":"…","user_message_id":"…","assistant_message_id":"…","provider_id":"openai","model_id":"…"}

event: delta
data: {"text":"Hel"}

event: usage
data: {"input_tokens":812,"output_tokens":64}

event: done
data: {"finish_reason":"stop","assistant_message_id":"…","status":"complete"}

event: error
data: {"code":"rate_limited","message":"OpenAI is rate limiting this key. Try again in 20 s.","retryable":true,"retry_after_s":20}
```

A stream always ends with exactly one `done` or one `error`. Errors found before streaming starts (unknown conversation, missing key) come back as normal JSON 4xx responses, not SSE, so the client handles them like any other request.

**One turn, step by step (`chat_service.run_turn`)**

1. Load the conversation and work out the provider, model and key. A missing key returns 409 `missing_key` before anything is written.
2. In one transaction: insert the user message (`complete`) and an empty assistant message (`streaming`), and update `conversations.updated_at`.
3. Register a `generation_id` in the in-memory registry, with an `asyncio.Event` for cancelling.
4. Send `start`. Call `build_context()` and loop over `provider.stream_chat()`. Forward each delta right away and add it to a buffer.
5. Save the buffer to the DB every 2 seconds during the stream, so a crash loses at most 2 seconds of text.
6. Handle the end in one `try/except/finally`:
    - Normal end: status `complete`, save usage and `finish_reason`, send `done`.
    - Cancel event set, or the client disconnected (`CancelledError` or `request.is_disconnected()`): close the provider stream, status `stopped`, keep the partial text, send `done` with `finish_reason: "cancelled"` if the connection is still open.
    - `ProviderError`: status `error`, save `error_code` and `error_message`, keep any partial text, send `error`.
    - Anything else: log the traceback (redacted), status `error` with code `unknown`, send a generic `error`.
7. Remove the generation from the registry in `finally`.

**Stop button:** the client calls `POST /api/generations/{id}/cancel` and then aborts the fetch. Either one alone is enough; doing both makes Stop reliable even when the cancel request is slow.

**Startup recovery:** when the app starts, set every message still marked `streaming` to `stopped`. That covers crashes and killed processes.

**One stream per conversation:** a second POST while a generation is running returns 409 `generation_in_progress`.

### 4.7 Database schema (SQLite)

Three tables, created by the Alembic migration `0001_initial`. The DB file is `<platformdirs user_data_dir>/Ling/ling.db`, or `$LING_DATA_DIR/ling.db` when that's set. Tests use a temporary folder.

```sql
CREATE TABLE conversations (
  id            TEXT PRIMARY KEY,                 -- uuid4
  title         TEXT NOT NULL DEFAULT 'New chat',
  system_prompt TEXT,                             -- NULL = use the default from settings
  provider_id   TEXT NOT NULL,
  model_id      TEXT NOT NULL,
  created_at    TEXT NOT NULL,                    -- ISO 8601 UTC, millisecond precision
  updated_at    TEXT NOT NULL
);
CREATE INDEX ix_conversations_updated ON conversations(updated_at DESC);

CREATE TABLE messages (
  id              TEXT PRIMARY KEY,
  conversation_id TEXT NOT NULL REFERENCES conversations(id) ON DELETE CASCADE,
  seq             INTEGER NOT NULL,              -- 1, 2, 3 ... within a conversation
  role            TEXT NOT NULL CHECK (role IN ('user','assistant','system','tool')),
  content         TEXT NOT NULL DEFAULT '',      -- plain text in stage 1
  status          TEXT NOT NULL DEFAULT 'complete'
                  CHECK (status IN ('complete','streaming','stopped','error')),
  provider_id     TEXT,                          -- assistant rows only
  model_id        TEXT,
  finish_reason   TEXT,
  input_tokens    INTEGER,
  output_tokens   INTEGER,
  error_code      TEXT,
  error_message   TEXT,
  metadata        TEXT,                          -- JSON; citations (stage 2), tool calls (stage 5)
  created_at      TEXT NOT NULL,
  updated_at      TEXT NOT NULL,
  UNIQUE (conversation_id, seq)
);

CREATE TABLE settings (
  key        TEXT PRIMARY KEY,   -- 'default_provider_id', 'default_system_prompt', 'base_url.local', ...
  value      TEXT NOT NULL,      -- JSON-encoded
  updated_at TEXT NOT NULL
);
```

- On every connection: `PRAGMA foreign_keys=ON`, `journal_mode=WAL`, `busy_timeout=5000`.
- `seq` is set to `max(seq)+1` in the same transaction as the insert. Sorting by timestamp breaks when two rows land in the same millisecond.
- `role` already allows `tool`, and `metadata` already exists, so stages 2 and 5 can add data without a migration. Content stays plain text until images or tool calls actually need structured parts.
- No API key columns anywhere, ever. The no-leak test (4.8) checks this.
- `provider_id` and `model_id` are stored on each assistant message as well as on the conversation. If the user switches models mid-chat, the record still shows which model wrote what. Stage 3 evals and stage 7 routing both need this.

### 4.8 Key storage and security

Keys go into the OS keychain through `keyring`, are read only at the moment of a provider call, and never leave the backend.

**Keystore (`secrets/keystore.py`)**

- Service name `ling-assistant`, username = provider id. Methods: `get(provider_id)`, `set(provider_id, key)`, `delete(provider_id)`, `source(provider_id)`.
- Lookup order: keychain, then an environment variable (`OPENAI_API_KEY`, `ANTHROPIC_API_KEY`). Env keys are for development and CI, and they're never copied into the keychain.
- If there's no keychain (common in WSL and headless Linux), `set` raises `keychain_unavailable` and the UI says so. There's no silent fallback to plain text. In WSL, use env vars for development.
- `key_hint` is the last 4 characters. The full key is never in any response.
- Keys are checked before saving, so a typo fails when you enter it, not on your first message.
- Base URLs for `local` and other OpenAI-compatible providers aren't secrets. They go in the `settings` table.

**Logging**

- A `logging.Filter` on the root logger replaces anything that looks like a key (`sk-[A-Za-z0-9_-]{16,}`, `sk-ant-[A-Za-z0-9_-]{16,}`, and any value currently in the keystore) with `[REDACTED]`.
- Request bodies are never logged. Access logs record method, path, status and duration only.
- The `httpx`, `openai` and `anthropic` loggers are set to `WARNING`.
- Logs record message ids and lengths, not message content. Set `LING_LOG_CONTENT=1` to log content for local debugging.

**Hardening the local server**

Any website the user visits can try to send requests to `127.0.0.1:8765`. These four rules block that:

1. Bind to `127.0.0.1` only, never `0.0.0.0`.
2. `TrustedHostMiddleware` allows only `127.0.0.1`, `localhost` and `[::1]`. This blocks DNS rebinding attacks.
3. CORS allows only the Vite dev address, and only when `LING_ENV=dev`. In production the backend serves the built frontend from the same address.
4. Requests that change data must have `Content-Type: application/json`, or they get 415. That forces the browser to send a CORS preflight, which other websites fail.

**The no-leak test (`tests/test_secrets.py`)**

Set a fake key `sk-test-LEAKCHECK0123456789abcdef` through the API, with the fake provider and a mocked validator. Run a normal chat turn and an error turn. Then check that the string is missing from the captured logs, every JSON response, the SSE output and the raw bytes of the SQLite file.

### 4.9 Frontend

Two routes, and one rule: server data lives in TanStack Query, and the single in-flight stream lives in a Zustand store.

**Routes**

- `/` and `/c/:conversationId`: sidebar + chat view. `/` shows an empty state with the message box; sending creates the conversation and then goes to `/c/:id`.
- `/settings`: API keys, local server address, default model, default system prompt, temperature.

**Components**

| Component | What it does |
| --- | --- |
| `Sidebar` | New chat button, conversations newest first, rename in place, delete with a confirm, link to settings |
| `ChatView` | Header (title, `ModelPicker`, system prompt button), `MessageList`, `Composer` |
| `MessageList` | Shows saved messages plus the live draft from the store. Stays scrolled to the bottom unless the user has scrolled up more than 80 px, then shows a "Jump to latest" button |
| `Message` | Markdown with code highlighting and a copy button on each code block. Shows "Stopped" for `stopped`, and an inline `ErrorBanner` for `error` |
| `Composer` | Text box that grows as you type. Enter sends, Shift+Enter adds a line, Esc stops. The Send button turns into Stop while streaming |
| `ModelPicker` | Grouped by provider (hosted and local). Unconfigured providers are greyed out with an "Add key" or "Set up" link. A "Custom model id" field at the bottom. Saves to the conversation with PATCH |
| `SystemPromptEditor` | Side drawer. Shows the default as placeholder text; "Reset to default" sets it back to NULL |
| `ApiKeyForm` | Password field, Save (checks the key first), Test, Remove. Shows `key_hint` and whether the key comes from the keychain or an env var |
| `LocalServerForm` | Base URL field (default Ollama), Test button, and a list of models the server reports |
| `ErrorBanner` | Turns each error code into a message and an action (table below) |

**Stream handling (`api/stream.ts` + `stores/chatStream.ts`)**

```ts
type StreamHandlers = {
  onStart(e: StartEvent): void;
  onDelta(text: string): void;
  onUsage(u: UsageEvent): void;
  onDone(e: DoneEvent): void;
  onError(e: StreamError): void;
};
function streamChat(url: string, body: unknown, h: StreamHandlers, signal: AbortSignal): Promise<void>;
```

- Putting every delta straight into React state causes one re-render per token. Collect deltas in a ref and push them to the store once per `requestAnimationFrame`.
- On `onDone` or `onError`, invalidate the conversation query so the saved message replaces the draft. Invalidate the list too, since `updated_at` changed.
- Turn off automatic retry in `fetch-event-source`. A retried POST would send the message twice.
- Stop = `POST /cancel` (don't wait for it) + `controller.abort()`. Keep the partial text on screen and mark it "Stopped".

**Error messages**

| Code | Message | Action |
| --- | --- | --- |
| `missing_key` | No API key for {provider} yet. | Open settings |
| `auth_failed` | {provider} rejected this key. | Open settings |
| `rate_limited` | {provider} is rate limiting this key. | Retry (disabled for `retry_after_s`, with a countdown) |
| `context_too_long` | This chat is longer than the model can take. | Start a new chat, or pick a model with a bigger context window |
| `model_not_found` | {model} isn't available. | Open the model picker |
| `provider_unavailable` | {provider} is having trouble right now. | Retry |
| `local_server_down` | Can't reach your local model server. | Show the URL, "Is Ollama running?", Retry |
| `network` | Couldn't reach {provider}. Check your connection. | Retry |
| `keychain_unavailable` | No system keychain was found. | Link to the README section on env keys |
| `unknown`, or backend down | Something went wrong. | Retry; if `/api/health` fails, show a top bar: "Backend not running" |

### 4.10 Testing, eval script and CI

The eval script starts in stage 1 so the habit is in place before RAG. In stage 1 it checks the chat pipeline. Stage 3 adds retrieval metrics to the same runner.

**Backend tests (pytest)**

- `test_providers_openai.py`, `test_providers_anthropic.py`: replay recorded SSE fixtures with `respx`. Check delta text, usage, finish reason, and that each HTTP status maps to the right `ProviderErrorCode`.
- `test_providers_local.py`: a connection refused on a localhost URL becomes `local_server_down`; no key is required.
- `test_chat_flow.py`: with `FakeProvider`, a full turn saves two messages with the right `seq`, status and content.
- `test_cancel.py`: start `slow:50`, cancel after 3 deltas, check status `stopped`, the partial content is saved, and the provider generator was closed.
- `test_disconnect.py`: drop the client mid-stream; same checks as above.
- `test_recovery.py`: a row left in `streaming` becomes `stopped` after startup.
- `test_conversations.py`, `test_settings.py`: create, read, update, delete, cascade delete, defaults.
- `test_secrets.py`: the no-leak test, plus a 404 for unknown provider ids.
- `test_security.py`: a wrong `Host` gets 400, a `text/plain` POST gets 415, a foreign `Origin` gets no CORS headers.

Record fixtures once with a real key using `scripts/record_fixture.py`, then strip all headers before committing.

**Frontend tests**

- Vitest: the SSE parser (split frames, multi-line data, heartbeats), the store logic (start, delta, done, error, stop) and the error-code-to-message map.
- Playwright, one test: backend running with `LING_ENABLE_FAKE=1`, send `echo:hello world`, watch it stream, press Stop on `slow:50`, reload, see "Stopped".

**Eval script (`evals/run.py`)**

```bash
uv run python evals/run.py --suite chat_core --provider fake                  # CI, offline, seconds
uv run python evals/run.py --suite chat_core --provider openai --model <id>   # local, uses your key
uv run python evals/run.py --suite chat_core --provider local --model <id>    # your local model
```

Each case in `cases/chat_core.yaml` has an input, an optional system prompt and checks. Stage 1 checks are all exact rules: `contains`, `not_contains`, `regex`, `max_chars`, `finish_reason`, `status`. Example cases:

- System prompt "Always answer in French" + "What colour is the sky?": regex matches common French words (live mode only)
- System prompt "Reply with exactly one word": `max_chars: 20` (live mode only)
- `echo:ping`: `contains: ping`, `status: complete` (both modes)
- `error:rate_limited`: `status: error` (fake mode)
- `long` with cancel after 500 ms: `status: stopped` (fake mode)

The script calls the real HTTP API with `httpx`, so it tests the same path the UI uses. It prints a pass/fail table, plus p50 and p95 time to first token and total time, and writes `evals/results/<timestamp>.json`. It exits with code 1 if anything fails. Running the live suite against a hosted model and a local model also gives you a fair comparison to show in the README.

**CI (`.github/workflows/ci.yml`)**

Runs on every push and pull request, as two parallel jobs:

- Backend: `uv sync`, `ruff check`, `ruff format --check`, `pyright`, `pytest`, `evals/run.py --provider fake`. CI machines have no keychain, so `conftest.py` sets an in-memory `keyring` backend.
- Frontend: `npm ci`, `gen-types` and fail if `schema.d.ts` changed (this catches API changes nobody synced), `tsc --noEmit`, `eslint`, `vitest run`, `vite build`.

Playwright runs locally with `make e2e`. Add it to CI in stage 3 if it stays stable.

---

## Part 5: Stage 1 lessons

Four parts, about one day each if things go well. If a day runs over, cut in this order: Playwright test, rate-limit countdown, temperature setting, rename. Don't cut the cancel, recovery or no-leak tests. Those are what make this read as engineering rather than a demo.

"You write" means I type the code with guidance. "Cursor writes" means Cursor writes it and walks me through it.

### Part A: Setup

**Lesson 0.1: Install the tools (WSL)**
- Concepts: WSL vs Windows, where Linux installs things, PATH, `which`
- Build: install git, make, uv, Python 3.12 (through uv), nvm + Node LTS, and the GitHub CLI, all inside WSL. Install Cursor on Windows and connect it to WSL. Optional: install Ollama and pull one small model that fits my hardware.
- Who: You write (run the commands yourself)
- Done when: `git --version`, `make --version`, `uv --version`, `uv python list`, `node -v` and `npm -v` all work in WSL, and Cursor shows "WSL: Ubuntu" in the bottom-left corner

**Lesson 0.2: Create the repo**
- Concepts: git basics (clone, add, commit, push), `.gitignore`, project layout
- Build: create `ling-assistant` on GitHub, clone it into `~/projects`, create `backend/`, `frontend/`, `evals/`, `docs/`, `.cursor/rules/`. Add this guide and a `.gitignore` for Python, Node and `.env`.
- Who: You write
- Done when: the first commit is pushed and visible on GitHub

### Part B: Backend foundation

**Lesson 1: Python project setup**
- Concepts: virtual environments, `pyproject.toml`, lockfiles, `uv add` vs `uv add --dev`, linters and type checkers, Makefile basics
- Build: `uv init --package --name ling`, add the dependencies, configure ruff and pyright, write the `Makefile` (`setup`, `dev`, `test`, `lint`)
- Who: Cursor writes, you review
- Done when: `make lint` runs clean on an empty package

**Lesson 2: First FastAPI app**
- Concepts: what an HTTP API is, FastAPI routes, the app factory pattern, lifespan events, `async def`, testing with `httpx.AsyncClient`
- Build: `create_app()` in `main.py`, `GET /api/health`, the first test
- Who: You write
- Done when: `make dev` starts the server, `curl http://127.0.0.1:8765/api/health` returns JSON, and the test passes

**Lesson 3: Config and logging**
- Concepts: settings from environment variables (`pydantic-settings`), Python logging (loggers, handlers, filters), regex basics
- Build: `config.py`, `logging.py` with the redaction filter, a test that a fake key gets redacted
- Who: You write the filter; Cursor writes the config
- Done when: the redaction test passes

**Lesson 4: Security middleware**
- Concepts: CORS and preflight requests, the Host header, DNS rebinding, middleware in FastAPI
- Build: trusted hosts, the CORS rule, the content-type check, `test_security.py`
- Who: Cursor writes, you review
- Done when: all three security tests pass

**Lesson 5: Database and migrations**
- Concepts: SQL tables, primary and foreign keys, indexes, ORMs (SQLAlchemy 2.0), async sessions, migrations (Alembic), SQLite PRAGMAs and WAL
- Build: `db/engine.py`, `db/models.py`, migration `0001_initial`
- Who: You write the models; Cursor writes the engine setup and Alembic config
- Done when: `alembic upgrade head` creates the DB, and you can open it with `sqlite3` and see the three tables

**Lesson 6: Repositories and conversation routes**
- Concepts: the repository pattern, Pydantic schemas vs database models, dependency injection with `Depends`, REST design, pagination
- Build: `repos.py`, schemas, the conversations and settings routes, their tests
- Who: You write `ConversationRepo` and one route; Cursor writes the rest following your pattern
- Done when: CRUD tests pass, including cascade delete

**Lesson 7: Keystore and provider routes**
- Concepts: OS keychains, the `keyring` library, why WSL usually has no keychain, env var fallback
- Build: `secrets/keystore.py`, the `/api/providers` routes (validation stubbed until Lesson 12), the skeleton of `test_secrets.py`
- Who: You write the keystore; Cursor writes the routes
- Done when: setting and removing a key works through the API in tests (in-memory keyring)

**Lesson 8: CI**
- Concepts: GitHub Actions, workflows, jobs, caching
- Build: the backend job in `ci.yml`
- Who: Cursor writes, you review
- Done when: a green check appears on GitHub

### Part C: Providers and streaming

**Lesson 9: The provider interface**
- Concepts: abstract base classes, async generators (`async for`, `yield`), discriminated unions in Pydantic, why one interface matters for Ling's whole roadmap
- Build: `providers/types.py`, `errors.py`, `base.py`, `registry.py`
- Who: You write
- Done when: pyright is happy and you can explain how stage 5 tool calls will fit in

**Lesson 10: FakeProvider**
- Concepts: test doubles, deterministic tests, `asyncio.sleep`
- Build: `providers/fake.py` with `echo:`, `error:`, `slow:`, `long`
- Who: You write
- Done when: unit tests for each command pass

**Lesson 11: Streaming a turn**
- Concepts: Server-Sent Events, `StreamingResponse`, how a long request stays open, transactions
- Build: `core/sse.py`, `core/context.py`, `core/chat_service.py` (normal path only), the `/messages` route
- Who: You write `run_turn`; Cursor writes `sse.py`
- Done when: `curl -N` shows `echo:hello` streaming from the fake provider, and `test_chat_flow.py` passes

**Lesson 12: Stop, disconnect, recovery**
- Concepts: asyncio cancellation, `CancelledError`, `try/finally`, `asyncio.Event`, crash recovery
- Build: the generation registry, the cancel route, disconnect handling, startup recovery, `/retry`, their tests
- Who: You write the cancel and disconnect handling; Cursor writes the tests, and you read them
- Done when: the cancel, disconnect and recovery tests pass

**Lesson 13: Real providers (hosted and local)**
- Concepts: the OpenAI and Anthropic streaming formats, how Ollama copies the OpenAI API, mapping HTTP errors to your own error codes, recording fixtures
- Build: `openai_compat.py` (used by `openai` and `local`), `anthropic.py`, `/api/models`, the base-url route, fixtures, adapter tests, the finished no-leak test
- Who: You write `openai_compat.py`; Cursor writes `anthropic.py` following your pattern
- Done when: `curl -N` streams from a real hosted model and from your local model, and all adapter tests pass

**Lesson 14: Eval script v0**
- Concepts: what an eval is versus a unit test, latency percentiles (p50, p95), YAML test cases
- Build: `evals/run.py`, `cases/chat_core.yaml`, add it to CI
- Who: You write
- Done when: the fake suite passes in CI, and you've run the live suite against one hosted and one local model

### Part D: Frontend core

**Lesson 15: React and TypeScript tour**
- Concepts: components, props, state, `useEffect`, JSX, TypeScript types and interfaces, how Vite runs the app
- Build: create the Vite app, set up Tailwind and shadcn, add the `/api` proxy, make a page that shows `/api/health`
- Who: You write the health page; Cursor writes the setup
- Done when: the browser shows "backend ok"

**Lesson 16: API client and server state**
- Concepts: generating types from OpenAPI, fetch, TanStack Query (queries, mutations, invalidation)
- Build: `make gen-types`, `client.ts`, the Query provider, a conversations list hook
- Who: Cursor writes, you review
- Done when: a list of conversations (made with curl) appears in the browser

**Lesson 17: The stream client and store**
- Concepts: reading a stream with fetch, parsing SSE, `AbortController`, Zustand, batching updates with `requestAnimationFrame`
- Build: `api/stream.ts`, `stores/chatStream.ts`, Vitest tests for both
- Who: You write
- Done when: the parser and store tests pass

**Lesson 18: Layout and sidebar**
- Concepts: React Router, layout components, lists and keys
- Build: the app layout, `Sidebar`, routes, the empty state
- Who: Cursor writes, you review
- Done when: clicking a conversation changes the URL and the view

**Lesson 19: Chat view and messages**
- Concepts: rendering markdown safely, code highlighting, scroll management with refs
- Build: `ChatView`, `MessageList`, `Message`
- Who: You write `MessageList`'s scroll logic; Cursor writes the rest
- Done when: saved messages render with markdown, and the view sticks to the bottom correctly

**Lesson 20: Composer and Stop**
- Concepts: controlled inputs, keyboard events, connecting UI to the store
- Build: `Composer` with Send/Stop, Enter, Shift+Enter and Esc
- Who: You write
- Done when: you can chat with the fake provider in the browser, stop mid-reply, reload and still see "Stopped"

**Lesson 21: Model picker**
- Concepts: grouped selects, optimistic updates
- Build: `ModelPicker` with hosted and local groups and a custom id field
- Who: Cursor writes, you review
- Done when: switching between a hosted model and a local model in one chat works, and each message shows which model wrote it

### Part E: Finish and ship

**Lesson 22: Settings page**
- Concepts: forms, password fields, showing a secret's status without showing the secret
- Build: `SettingsPage`, `ApiKeyForm`, `LocalServerForm`, default model and system prompt
- Who: Cursor writes, you review
- Done when: adding a key, testing it, removing it, and changing the local server URL all work

**Lesson 23: Per-conversation system prompt**
- Build: `SystemPromptEditor` drawer with "Reset to default"
- Who: You write
- Done when: a conversation's prompt overrides the default, and reset brings the default back

**Lesson 24: Errors and retry**
- Concepts: designing error states, countdown timers, health checks
- Build: `ErrorBanner` for every code, retry, the rate-limit countdown, the "Backend not running" bar
- Who: Cursor writes, you review
- Done when: each `error:<code>` from the fake provider shows the right message and the retry works. Also stop Ollama and confirm `local_server_down` shows up.

**Lesson 25: Rename, delete, titles**
- Build: rename in place, delete with confirm, the 60-character title rule
- Who: You write
- Done when: tests and a manual check pass

**Lesson 26: End-to-end test**
- Concepts: what e2e tests catch that unit tests don't, Playwright basics
- Build: the one Playwright test, `make e2e`
- Who: Cursor writes, you review
- Done when: `make e2e` passes

**Lesson 27: README and release**
- Build: README (what Ling is, a GIF, quick start, architecture diagram, roadmap, security notes, how to use a local model), `docs/architecture.md`, a manual test pass (bad key, no network, Stop during a long code block, two tabs open, a very long chat, local server off), tag `v0.1.0`, GitHub release
- Who: You write the README; Cursor helps with the diagram
- Done when: every box in the definition of done (4.1) is ticked

---

## Part 6: Progress log

Cursor ticks a lesson when its "Done when" checks pass.

- [x] 0.1 Install the tools
- [ ] 0.2 Create the repo
- [ ] 1 Python project setup
- [ ] 2 First FastAPI app
- [ ] 3 Config and logging
- [ ] 4 Security middleware
- [ ] 5 Database and migrations
- [ ] 6 Repositories and conversation routes
- [ ] 7 Keystore and provider routes
- [ ] 8 CI
- [ ] 9 The provider interface
- [ ] 10 FakeProvider
- [ ] 11 Streaming a turn
- [ ] 12 Stop, disconnect, recovery
- [ ] 13 Real providers (hosted and local)
- [ ] 14 Eval script v0
- [ ] 15 React and TypeScript tour
- [ ] 16 API client and server state
- [ ] 17 The stream client and store
- [ ] 18 Layout and sidebar
- [ ] 19 Chat view and messages
- [ ] 20 Composer and Stop
- [ ] 21 Model picker
- [ ] 22 Settings page
- [ ] 23 Per-conversation system prompt
- [ ] 24 Errors and retry
- [ ] 25 Rename, delete, titles
- [ ] 26 End-to-end test
- [ ] 27 README and release

**Notes** (changes from the spec, one line each)

- 0.1: Miniconda (Python 3.13, `(base)`) was already installed and still provides `python`. Ling's Python is uv's 3.12, invoked as `python3.12` until the project venv exists. `gh` was already installed and logged in. Node is nvm's v26 LTS.