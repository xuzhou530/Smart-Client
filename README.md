# Smart-Client AI Workspace

A unified AI application platform that brings together LLMs, agents, workflow orchestration, knowledge bases, skills, and MCP services.

In a single workspace, users can connect multiple model providers, compose workflows / Chatflows on a canvas, attach knowledge bases, skills and MCP tools to agents, then publish applications through Web pages, APIs, Webhooks and MCP endpoints — and review real call volume, latency and cost on the monitoring page.

---

## 1. Feature Overview

| Module | Capabilities |
|---|---|
| Chat | Multi-session streaming chat (SSE), prompt enhancement, message feedback, stop generation, source citation display |
| Studio (Apps) | Workflow / Chatflow app list, create blank app, orchestrate → test run → publish |
| Workflow Orchestration | Canvas nodes: Start / Knowledge Retrieval / LLM / Agent Collaboration / ANSWER / Conditional Branch / HTTP Request / Code Execution / MCP Tool |
| Agents | Agent config (prompt, model, temperature, history length, skills, knowledge bases, MCP tools), draft / publish / take offline, import & export |
| Agent Forge | One-line requirement → clarifying Q&A → plan generation → confirmation → publish as agent, with SSE events throughout |
| Knowledge Base | Creation wizard, pipeline orchestration, multi-format ingestion (PDF/Word/Excel/TXT/MD/CSV/JSON/HTML), chunking & vectorization, recall testing, external API |
| Skills | Skill list / detail, SKILL.md description & file tree management, zip import, test run, publish to Capability Hub |
| MCP Services | Connect MCP services (http / sse / stdio), tool list sync & on-demand enable, single-tool debugging, invocation logs |
| Model Access | Model CRUD, provider credentials, price per 1K tokens, enable / publish / take offline |
| Provider Market | 17 preset providers: browse → install → enter credentials → probe validation → pull models → set default → publish |
| Integrations | Triggers (scheduled / Webhook), Agent policies (guardrail / rate limit / model routing), extensions, custom endpoints, workspace members |
| Capability Hub / My Capabilities | Aggregation of published agents, skills, knowledge bases and MCP services |
| Ops & Monitoring | App logs (session-level detail), monitoring page (call volume, failure rate, tokens, cost), annotations, daily stat aggregation |
| Plugin Market | Sync Dify plugin market entries, whitelist management, install mapping to native capabilities |
| Misc | Login / sign-up, audit logs, font scaling, secure transport (toggleable), Electron desktop shell |

---

## 2. Tech Stack

**Backend**
- Java 21 / Spring Boot 3.3.5
- MyBatis-Plus 3.5.6 (soft delete, auto-filled timestamps)
- MySQL 8 + Redis (login state & rate-limit counters)
- Sa-Token 1.38 (JWT mode, request header `satoken`)
- WebFlux WebClient (calling models / MCP, pooled connections)
- PDFBox 2.0.31 + POI-OOXML 5.2.5 (document parsing)
- Unified response wrapper `{ code, message, data }`

**Frontend**
- Vue 3.4 + TypeScript + Vite 5
- Pinia (state), Vue Router 4 (hash routing)
- TailwindCSS 3 + SCSS (custom design tokens)
- Element Plus (icons)
- Electron 45 (optional desktop packaging)

---

## 3. Quick Start

Prerequisites: **JDK 21, Maven, Node 18+, MySQL 8, Redis**.

**No environment variables required**: every config has a default. Secret keys (JWT signing, API Key encryption) are auto-generated when left blank and persisted to the database table `t_system_secret` (or to `./data/` if the database is unavailable); the database itself is auto-created via `createDatabaseIfNotExist=true`.

### Option 1: One-command startup (recommended)

```bash
./start.sh            # start backend + frontend, print URLs once both are ready
./start.sh stop       # stop
./start.sh restart    # restart
./start.sh status     # show status
./start.sh logs       # tail backend + frontend logs (Ctrl+C exits the tail only)
```

The script handles: environment pre-check (Java/Maven/Node/MySQL/Redis), port conflict check, frontend dependency install (first run), backend health wait, rollback on startup failure, and log-location hints. To change ports: `BACKEND_PORT=8081 ./start.sh`.

### Option 2: Manual startup

```bash
# Backend: http://localhost:8080
cd backend && mvn spring-boot:run

# Frontend: http://localhost:5173
cd frontend && npm install && npm run dev
```

### Access & accounts

- Frontend: <http://localhost:5173>; backend health check: `GET /api/health`
- First-time use: click "Sign up" on the login page (auto-login after registration)
- The frontend points to the backend via `VITE_API_BASE` in `frontend/.env`

### Default connection config

| Item | Default | Override variable |
|---|---|---|
| MySQL | `localhost:3306/smart-client-db`, root/root | `MYSQL_HOST` `MYSQL_PORT` `MYSQL_DATABASE` `MYSQL_USER` `MYSQL_PASSWORD` |
| Redis | `localhost:6379`, no password | `REDIS_HOST` `REDIS_PORT` `REDIS_PASSWORD` |
| Backend port | 8080 | `APP_PORT` (deliberately not `SERVER_PORT`, to avoid accidental injection by containers / IDEs) |

On startup, `spring.sql.init` automatically runs:
- `db/schema.sql` — creates all business tables (including the `t_system_secret` key table)
- `db/providers.sql` — seeds 17 providers (`INSERT IGNORE` + unique key, idempotent)

---

## 4. Directory Structure

```
Smart-Client/
├── backend/                      # Spring Boot backend
│   └── src/main/java/xz/sc/smartclient/
│       ├── common/               # Unified response, exceptions, rate limiting, audit, secure codec, migrators
│       ├── config/               # Sa-Token / MyBatis-Plus / secure transport / plugin market config
│       ├── controller/           # REST controllers per module
│       ├── entity/ mapper/       # Entities & MyBatis-Plus mappers
│       ├── service/              # Business services (KB ingestion, MCP manager, etc.)
│       ├── llm/                  # AgentRuntime, gateway factory, RAG, Embedding, MCP client
│       │   └── provider/         # Built-in provider adapters (qwen/hunyuan/openai/dynamic/local fallback)
│       ├── providers/            # Provider module (market/credentials/validation/model pull/protocol adapters)
│       ├── workflow/             # Orchestration executor SPI & node implementations
│       └── ops/                  # Policy engine, stat aggregation, trigger scheduling, plugin sync
├── frontend/                     # Vue 3 frontend
│   └── src/
│       ├── api/                  # API wrappers per module
│       ├── components/           # Reusable canvas, panels, drawers
│       ├── views/                # Pages (agent/app/kb/skill/tools/providers/...)
│       ├── stores/               # Pinia stores
│       ├── styles/               # Global styles & design tokens
│       └── router/ config/       # Routing & sidebar menu config
├── doc/                          # Design docs & demo data
├── scripts/                      # Helper scripts
├── start.sh                      # One-command startup script
└── README.md
```

---

## 5. Core Design

### 1. Unified agent runtime assembly

`AgentRuntime.prepare(...)` is the single assembly entry point, building all context for one conversation in a fixed order:

1. **Policy check**: `PolicyEngine` runs guardrail and rate limiting first; a hit rejects the request immediately without consuming model quota
2. **System prompt**: agent prompt + prompts from bound skills
3. **RAG**: retrieve relevant chunks from bound knowledge bases and inject them into context (explicit warning when nothing matches; no fabricated references)
4. **Model selection**: explicit modelId > agent-bound model > routing policy > frontend-specified provider > default model
5. **MCP tools**: load tools from bound MCP services and their invocation bindings
6. **Message assembly**: truncate history per the agent's configured length to keep long sessions bounded

When a model is unavailable, the runtime **fails explicitly** (disabled / offline / incomplete config / masked-key placeholder) instead of silently falling back, so "the switch didn't take effect" is never a mystery. Only when no real provider is configured at all does it fall back to a local demo mode, clearly stated in the reply.

### 2. Workflow node executor SPI

```
NodeExecutor (type() + execute(NodeContext) → NodeResult)
├── McpNodeExecutor      MCP_TOOL  pooled connections, retries, separate connect/invoke timing
├── LlmNodeExecutor      LLM       temperature, system prompt, variable template rendering
└── AgentNodeExecutor    AGENT     invokes another agent with its own prompt/skills/RAG/MCP assembly
```

- Other nodes (knowledge retrieval / code / branch / start / end) do not alter the existing chain — they simply "skip" and advance
- Variable references go through `VariableResolver`, supporting paths such as `{{start.query}}` and `{{mcpResult.data}}`
- Orchestration runs push SSE events: `start / step(running|success|failed|skipped) / failed / done / run-cancelled / error`
- AGENT nodes enforce ownership isolation and self-invocation detection, preventing orchestration from borrowing another user's prompts and resources
- Orchestration runs also write session logs, so the monitoring page shows real usage, failure rate and the actual model used across the chain

### 3. Agent policies actually take effect

The three policy types configured on the Integrations page are executed at runtime by `PolicyEngine` (previously CRUD-only and never read at runtime):

| Type | Config | Behavior |
|---|---|---|
| guardrail | `blockKeywords` / `maxInputChars` | Keyword hit or overlong input → immediate `400` rejection |
| limit | `maxRequestsPerMinute` | Rate limit per user + agent → `429` on excess |
| routing | `modelId` | Use this model when none is explicitly specified |

`t_agent_policy` has no `agent_id` column; policies are linked to agents via an `agentIds` array in config, and its absence means the policy applies to all agents of that user.

### 4. Model providers: config-driven

- Only two protocol adapters exist — `OpenAiProtocolClient` / `AnthropicProtocolClient` — dispatched by the `provider.protocol` column; **writing a per-vendor Adapter is forbidden**
- Providers, credential fields (JSON Schema) and models all come from the database: **adding a provider = one SQL insert, zero Java/frontend changes**
- The frontend renders form controls dynamically from `credential_schema`; the backend validates required fields from the same source (double safeguard)
- **Validate before persisting**: a minimal probe runs before install (GET /models → POST /chat/completions → timeout fallback); a failure leaves no dirty data
- Provider validation is limited to 1 request / 5 seconds per workspace + provider (Redis INCR + EXPIRE)
- Model pulling runs asynchronously on a dedicated thread pool; a failure only marks the status and does not affect credentials already stored

### 5. Secrets & security

| Mechanism | Description |
|---|---|
| Field encryption | API Keys and similar fields are encrypted with AES-256-GCM (random 12B IV, base64(iv + ciphertext)), prefixed with `enc:v1:` |
| Masked read-back | Output only shows `sk-****abcd`; sending the masked value back never overwrites the real key (detected by `****`) |
| Secret custody | With no env injection, resolves as "database `t_system_secret` → `./data/*.key` → auto-generate", then writes into global config at startup |
| Log masking | Authorization / API Keys are never printed; audit logs record operator and target resource ID |
| Ownership isolation | Agents / skills / knowledge bases / models / MCP services are filtered by user and scope; unauthorized access is handled as "not found" |
| Secure transport | When `TRANSPORT_SECURE_ENABLED=true`, raw frontend/backend payloads go through "compress → encrypt → Base64" |

### 6. Real knowledge-base retrieval chain

- Ingestion: file → text extraction (PDF/Word/Excel/TXT/MD/CSV/JSON/HTML) → chunking → vectorization via embedding API → write slices, with SSE progress events
- Retrieval: vectorize the query and score with cosine similarity
- **When no embedding model is available, return null and automatically fall back to lexical search**, labeling the real mode in the result instead of disguising lexical search as vector search
- Ingestion task state is persisted in `t_kb_document`, so tasks continue after the page is closed and progress can be resumed on re-entry

---

## 6. Main Endpoints

Unified prefix `/api`, authenticated via the `satoken` request header (returned after login), with the unified response body `{ code, message, data }`.

| Module | Prefix | Description |
|---|---|---|
| Auth | `/api/auth` | Sign up / login / logout / current user |
| Chat | `/api/chat` | Session CRUD, message list, streaming `/stream`, prompt enhancement, stop, feedback |
| Agents | `/api/agents` | CRUD, publish / take offline, copy, export, import from file / URL |
| Orchestration | `/api/workflows` | Canvas read/write, run (SSE), stop, available variables, pre-publish check |
| Apps | `/api/apps` | Access entries, publish / take offline, API Key, Web config, channel toggles, triggers |
| Public entries | `/api/public/apps/{id}`, `/api/v1/apps/{id}/chat`, `/api/mcp/app/{id}` | Login-free Web / API / MCP entries, Webhook callbacks |
| App ops | `/api/apps/{appId}/logs`, `/monitor`, `/annotations`, `/agent` | Logs, monitoring, annotations, app-specific chat |
| Agent Forge | `/api/agent-forge` | Templates, create run, event stream, clarification, plan confirmation, cancel, publish |
| Knowledge Base | `/api/knowledge-bases`, `/api/kb` | CRUD, documents, retrieval, upload, ingestion progress, pipelines, recall testing, stats, external API |
| External KB | `/api/v1/datasets` | Public dataset retrieval API |
| Skills | `/api/skills` | CRUD, publish / take offline, zip upload, file tree read/write |
| MCP Services | `/api/mcp/services` | CRUD, tool list, sync, single-tool invoke & enable, test, import |
| MCP Node | `/api/mcp` | Schema & debugging when selecting tools in orchestration nodes |
| Model Access | `/api/models` | CRUD, enable, credentials, price per 1K tokens, publish / take offline |
| Provider Market | `/api/providers` | Market, detail, pre-validation, install, installed list, change credentials, sync / add models, default model |
| Capabilities | `/api/capabilities`, `/api/studio/apps/count` | Capability Hub aggregation, My Capabilities, app count |
| Integrations | `/api/integrations` | Triggers, policies, extensions, custom endpoints, members |
| Trigger Catalog | `/api/triggers` | Catalog of available trigger types |
| Plugin Market | `/api/plugin-market` | Entries, sync, install, whitelist, icons |
| Testing | `/api/test/chat`, `/api/mcp/{id}/invoke`, `/api/skills/{id}/run`, `/api/kb/{id}/search` | Debug entries for each capability |
| Docs | `/api/docs` | Document collaboration |

### Add a provider (SQL only)

```sql
INSERT IGNORE INTO provider
(name, display_name, provider_group, icon, protocol, default_base_url,
 credential_schema, model_fetch_mode, official_url, doc_url, description, notes, sort, enabled)
VALUES
('xmodel', 'X Model', 'AGGREGATOR', 'X', 'OPENAI', 'https://api.xmodel.com/v1',
 '{"type":"object","required":["apiKey"],
   "properties":{"apiKey":{"type":"string","format":"password","title":"API Key"},
                 "baseUrl":{"type":"string","title":"Base URL",
                           "default":"https://api.xmodel.com/v1"}}}',
 'REMOTE', 'https://xmodel.com', 'https://xmodel.com/docs', 'One-line intro', 'Integration notes (optional)', 2, 1);
```

Refresh the market page and the provider appears — no restart, no Java/frontend changes. Re-running the statement is harmless.

---

## 7. Database Tables

Tables use the `t_` prefix (the provider module uses `provider*`). All tables and columns are commented, with a `deleted` soft-delete column.

| Group | Tables |
|---|---|
| System | `t_system_secret`, `t_user`, `t_workspace_member` |
| Chat | `t_chat_session`, `t_chat_message`, `t_document` |
| Models | `t_llm_model`, `provider`, `provider_credential`, `provider_model`, `provider_default_model` |
| Agents | `t_agent`, `t_agent_version`, `t_agent_file`, `t_app_config`, `t_agent_policy`, `t_agent_tool_binding` |
| Orchestration | `t_workflow`, `t_mcp_tool_cache`, `t_mcp_invoke_log` |
| Knowledge Base | `t_knowledge_base`, `t_knowledge_doc`, `t_kb_document`, `t_kb_template` |
| Skills | `t_skill`, `t_skill_file` |
| MCP | `t_mcp_server`, `t_mcp_service` |
| Ops | `t_conversation_log`, `t_stat_daily`, `t_annotation` |
| Integrations | `t_trigger`, `t_extension`, `t_api_endpoint` |
| Plugin Market | `t_plugin_market_item`, `t_plugin_market_whitelist`, `t_installed_plugin` |

---

## 8. Scheduled Jobs

| Job | Frequency | Description |
|---|---|---|
| `StatAggregator` | Daily at 00:10 | Aggregates the previous day's conversation logs into `t_stat_daily` for historical trends on the monitoring page |
| `TriggerScheduler` | 20s after startup, polls every 30s | Executes due scheduled triggers |
| `PluginSyncScheduler` | Daily at 03:00 | Syncs Dify plugin market entries |

---

## 9. Production Deployment Notes

Variables that must be explicitly injected:

| Variable | Purpose | If not injected |
|---|---|---|
| `PROVIDER_AES_KEY` | Encrypts fields such as `t_llm_model.api_key` | Key and ciphertext live in the same DB; a full DB dump decrypts every key |
| `SA_JWT_SECRET` | Login token signing | Same as above: the key is stored in the DB, and multiple instances must share the same storage |
| `TRANSPORT_SECRET_KEY` | Secure transport codec (only needed when `TRANSPORT_SECURE_ENABLED=true`) | Startup fails when that feature is enabled |
| `APP_SECURITY_STRICT` | When `true`, missing keys above fail startup | Defaults to `false`: warn only, do not block |
| `APP_CORS_ALLOWED_ORIGINS` | Allowed CORS origins | Defaults to `*`, suitable for development only |

**After switching the key source, do not delete old key records / files**, or historical ciphertext and already-issued tokens become unrecoverable.

Model credentials (optional, for auto-seeding public models): `QWEN_BASE_URL` `QWEN_API_KEY` `QWEN_MODEL`
`HUNYUAN_BASE_URL` `LLM_API_KEY` `OPENAI_BASE_URL` `OPENAI_API_KEY` `OPENAI_MODEL`.

---

## 10. Known Trade-offs

1. **The openai-java SDK is not used**: its dependencies conflict with Boot 3.3.5 version coupling. The protocol layer is implemented directly on a unified WebClient (pooling / timeouts / redirects disabled), equivalently covering the "official SDK + WebClient" requirement.
2. **The manual add-model endpoint is an out-of-spec extension**: Anthropic (no `/models`), Volcengine Ark (`ep-` endpoints) and Ollama (MANUAL) all rely on it to close the loop.
3. **Publishing to the Capability Hub**: `t_llm_model` stores only masked keys (to avoid a second plaintext write); real chat calls decrypt credentials at the invocation layer. If a key is written as a masked placeholder, the call fails explicitly instead of returning a mysterious 401.
4. **Local demo fallback**: with no real provider configured at all, it falls back to `LocalProvider` and explicitly declares "local demo mode" in the reply body — never disguised as real model output.

---

## 11. Development Conventions

- Frontend type check: `cd frontend && npx vue-tsc --noEmit -p tsconfig.json` (the `-p` flag is required)
- Frontend build: `npm run build` (type check first, then vite build)
- Backend compile: `cd backend && mvn -o compile -DskipTests`
- Backend tests: `cd backend && mvn -o test`
- Style: reuse existing components and design tokens (`--primary #2f6bff`, etc.); do not add third-party dependencies or hard-override color values
