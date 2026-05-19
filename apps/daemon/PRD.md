# PRD Técnico 360° — Daemon (`@open-design/daemon`)

## 1. Overview Técnico

### Stack Completa

| Camada | Tecnologia |
|---|---|
| Linguagem | TypeScript 5.6 (`"type": "module"`) |
| Runtime | Node.js ~24 |
| Framework HTTP | Express 4 |
| Bundler / Compilador | `tsc` (sem bundler; emite ESM puro em `dist/`) |
| Banco de dados | SQLite via `better-sqlite3` 12 (WAL mode, foreign keys) |
| Streaming SSE | Nativo via `res.write()` / `json-event-stream.ts` |
| Hash | `blake3-wasm` |
| Upload | `multer` 2 |
| Fetch externo | `undici` 7 |
| Telemetria | `posthog-node` 4 + `prom-client` 15 + Langfuse (HTTP nativo) |
| Testes | Vitest 2 |
| Compressão / ZIP | `jszip` + `tar` |
| File watch | `chokidar` 5 |
| MCP SDK | `@modelcontextprotocol/sdk` ^1.0 |

### Arquitetura Geral

O daemon é um **processo Node.js de longa duração** que:

1. Escuta em `127.0.0.1:7456` (padrão) ou porta configurada via `OD_PORT`.
2. Registra rotas Express modularizadas em `server.ts` por domínio.
3. Mantém estado de runs de agente **in-memory** (`runs.ts`) com TTL de 30 min.
4. Persiste metadados em SQLite (`db.ts`) e arquivos de projeto em `<dataDir>/projects/<id>/`.
5. Spawna CLIs de agentes como processos filho e faz streaming de stdout → SSE.
6. Expõe também um **servidor MCP stdio** (`od mcp`) independente do servidor HTTP.

```
apps/daemon/
├── src/
│   ├── server.ts          # Wiring de rotas + startup
│   ├── cli.ts             # Entrypoint `od` bin; roteador de subcomandos
│   ├── daemon-startup.ts  # startDaemonRuntime() + CLI flag parsing
│   ├── db.ts              # SQLite schema + migrations + queries
│   ├── runs.ts            # In-memory run service (SSE)
│   ├── agents.ts          # Re-exports de runtimes/*
│   ├── runtimes/          # Defs de agentes + detecção + spawn
│   ├── critique/          # Critique Theater (Design Jury)
│   ├── metrics/           # Prometheus registry
│   ├── live-artifacts/    # Live Artifacts store + refresh
│   ├── genui/             # GenUI surfaces
│   ├── plugins/           # Plugin system
│   ├── connectors/        # Connectors externos
│   ├── research/          # Research (web search)
│   ├── storage/           # Abstração de storage
│   ├── logging/           # Logging utilities
│   └── prompts/           # System prompt composers
├── tests/                 # Vitest tests (espelhando subpaths de src/)
└── sidecar/               # Sidecar entry (packaged / Electron)
```

---

## 2. Backend / Servidor

### Registro de Rotas

As rotas são registradas em `server.ts` agrupadas por domínio. Abaixo estão os domínios e módulos:

| Módulo de Rota | Prefixo | Domínio |
|---|---|---|
| `server.ts` (inline) | `/api/health`, `/api/version` | Bootstrap / process status |
| `active-context-routes.ts` | `/api/active` | Contexto UI ativo (projeto/arquivo aberto) |
| `chat-routes.ts` | `/api/projects/:id/chat`, `/api/runs/:runId/*` | Conversas + runs de agente |
| `project-routes.ts` | `/api/projects`, `/api/projects/:id/artifacts`, `/api/projects/:id/files`, `/api/projects/:id/uploads` | Projetos + arquivos |
| `import-export-routes.ts` | `/api/projects/:id/export`, `/api/import`, `/api/finalize` | Import/export de projetos |
| `live-artifact-routes.ts` | `/api/projects/:id/live-artifacts` | Live Artifacts |
| `deploy-routes.ts` | `/api/deploy`, `/api/deploy/check` | Deploy (Vercel, Cloudflare Pages) |
| `media-routes.ts` | `/api/media/*` | Geração de mídia (imagem/vídeo/áudio) |
| `mcp-routes.ts` | `/api/mcp/*` | MCP OAuth + config |
| `xai-routes.ts` | `/api/xai/*` | xAI/Grok OAuth |
| `routine-routes.ts` | `/api/routines`, `/api/routines/:id/runs` | Routines agendadas |
| `connectors/routes.ts` | `/api/connectors/*` | Connectors externos (Composio) |
| `static-resource-routes.ts` | `/api/skills`, `/api/design-systems`, `/artifacts/*`, `/frames/*` | Recursos estáticos |

### Endpoints Principais (inline em `server.ts`)

| Método | Path | Descrição |
|---|---|---|
| `GET` | `/api/health` | Health check — retorna `{ ok: true }` |
| `GET` | `/api/version` | Versão do daemon e do app |
| `GET` | `/api/agents` | Lista agentes detectados no PATH |
| `GET` | `/api/app-config` | Preferências persistidas do app |
| `PUT` | `/api/app-config` | Atualiza preferências (onboarding, agentId, skillId, etc.) |
| `GET` | `/api/analytics/config` | Config PostHog para o browser |
| `GET` | `/api/metrics` | Prometheus scrape endpoint |
| `GET` | `/api/skills` | Lista skills do registry |
| `GET` | `/api/design-systems` | Lista design systems |
| `GET` | `/api/media/config` | Lê config de provedores (keys mascaradas) |
| `PUT` | `/api/media/config` | Salva chaves de provedores de mídia |
| `POST` | `/api/runs/:id/tool-result` | Injeta `tool_result` em run stream-json aberta |
| `POST` | `/api/runs/:id/cancel` | Cancela run em andamento |
| `GET` | `/api/runs/:id/events` | SSE stream de eventos de uma run |

### Lógica de Negócio Principal

- **Composição de system prompt** (`prompts/system.ts`): combina instruções do daemon, skill body (se skill_id no projeto), design system (se design_system_id), craft sections e block final exclusivo Claude.
- **Spawn de agente** (`runtimes/`): resolve binário, constrói argv/stdin conforme formato do runtime (`text` ou `stream-json`), aplica env, inicia processo filho, parseia stdout com `streamFormat`.
- **In-memory run lifecycle** (`runs.ts`): `queued → running → succeeded/failed/canceled`; TTL de 30 min; máx de 2.000 eventos por run.
- **Plugin snapshots**: cada run recebe um `appliedPluginSnapshotId` imutável para replay byte-equal entre atualizações de plugin.
- **ACP / PI RPC**: runs de Claude usam `attachAcpSession()` para injetar respostas de `AskUserQuestion` via `tool_result` blocks sem re-spawn.

### Banco de Dados / Persistência

**SQLite (`<dataDir>/app.sqlite`)** — WAL mode, foreign keys ON, migração idempotente em `db.ts:migrate()`.

| Tabela | Descrição |
|---|---|
| `projects` | Projetos (id, name, skill_id, design_system_id, pending_prompt, metadata_json) |
| `templates` | Templates de projeto (snapshot de arquivos) |
| `conversations` | Conversas por projeto |
| `messages` | Mensagens (role, content, events_json, attachments, produced_files, feedback) |
| `preview_comments` | Comentários de inspeção visual por elemento HTML |
| `tabs` | Abas abertas por projeto |
| `deployments` | Deploy records (URL, provider, status, count) |
| `routines` | Tarefas agendadas (schedule, prompt, skill_id, agent_id) |
| `routine_runs` | Execuções de routine (status, conversation_id, summary) |
| `critique_runs` | Runs do Critique Theater com composite score |
| `critique_rounds` | Rounds do Critique Theater (per-run) |
| `media_tasks` | Tasks de geração de mídia |
| `applied_plugin_snapshots` | Snapshots imutáveis de plugins aplicados a runs |

**Arquivos no filesystem:**

| Path | Conteúdo |
|---|---|
| `<dataDir>/projects/<id>/` | Arquivos HTML, assets, sketches do projeto |
| `<dataDir>/projects/<id>/.live-artifacts/<laId>/` | Live Artifact (artifact.json, template.html, index.html, data.json, refreshes.jsonl) |
| `<dataDir>/app-config.json` | Preferências do app (onboarding, agentId, etc.) |
| `<dataDir>/media-config.json` | Chaves de provedores de mídia |
| `<dataDir>/memory/<projectId>/` | Entradas de memória por projeto |
| `~/.open-design/vercel.json` | Token Vercel (path governa `OD_USER_STATE_DIR`) |
| `~/.open-design/cloudflare-pages.json` | Token Cloudflare Pages |

---

## 3. Frontend / Cliente

O daemon é um backend puro. Não há componentes de UI próprios.

A única exceção é o **servidor de arquivos estáticos**: em modo dev, a web roda separada e o proxy Next.js redireciona `/api/*` para o daemon. No modo packaged, o daemon serve o build estático (`apps/web/out/`) via `STATIC_DIR`.

Contrato de comunicação web→daemon: `packages/contracts` (DTOs TypeScript compartilhados).

---

## 4. APIs e Contratos

### APIs Expostas

#### REST

Todos os endpoints respondem com `Content-Type: application/json`. Erros seguem o schema `ApiErrorResponse` de `packages/contracts`.

**Projetos**

| Método | Path | Payload | Response |
|---|---|---|---|
| `GET` | `/api/projects` | — | `{ projects: Project[] }` |
| `POST` | `/api/projects` | `{ name, skillId?, designSystemId? }` | `Project` |
| `GET` | `/api/projects/:id` | — | `Project` |
| `PUT` | `/api/projects/:id` | `Partial<Project>` | `Project` |
| `DELETE` | `/api/projects/:id` | — | `204` |
| `GET` | `/api/projects/:id/files` | — | `{ files: FileEntry[] }` |
| `GET` | `/api/projects/:id/files/:path` | — | Arquivo raw |
| `PUT` | `/api/projects/:id/files/:path` | `body: Buffer` | `{ ok: true }` |
| `DELETE` | `/api/projects/:id/files/:path` | — | `204` |

**Chat / Runs**

| Método | Path | Payload | Response |
|---|---|---|---|
| `POST` | `/api/projects/:id/chat` | `ChatRequest` | `{ runId: string }` |
| `GET` | `/api/runs/:runId/events` | — | SSE stream (`ChatSseEvent`) |
| `GET` | `/api/runs/:runId` | — | `RunStatus` |
| `POST` | `/api/runs/:runId/cancel` | — | `204` |
| `POST` | `/api/runs/:runId/tool-result` | `{ toolUseId, content, isError? }` | `204` |

**Conversas**

| Método | Path | Response |
|---|---|---|
| `GET` | `/api/projects/:id/conversations` | `Conversation[]` |
| `POST` | `/api/projects/:id/conversations` | `Conversation` |
| `GET` | `/api/projects/:id/conversations/:convId/messages` | `Message[]` |

**Media**

| Método | Path | Payload | Response |
|---|---|---|---|
| `POST` | `/api/media/generate` | `MediaGenerateRequest` | `MediaTask` |
| `GET` | `/api/media/tasks` | — | `MediaTask[]` |
| `GET` | `/api/media/tasks/:taskId` | — | `MediaTask` |

**Deploy**

| Método | Path | Payload | Response |
|---|---|---|---|
| `POST` | `/api/deploy/:provider` | `DeployRequest` | `{ url: string }` |
| `GET` | `/api/deploy/:provider/config` | — | Masked config |
| `PUT` | `/api/deploy/:provider/config` | `{ token, ... }` | `204` |

#### SSE (Server-Sent Events)

O evento principal é `GET /api/runs/:runId/events`. Tipos de evento definidos em `packages/contracts` como `ChatSseEvent`:

- `token` — fragmento de texto do agente
- `tool_use` / `tool_result` — chamadas de ferramenta
- `error` — erro do agente
- `status` — transição de status (running → succeeded)
- `usage` — token usage final
- `turn_end` — fim de turno do modelo
- `critique.*` — eventos do Critique Theater (round_start, round_end, ship, degraded, etc.)

#### Webhook / Streaming para Langfuse

`POST https://us.cloud.langfuse.com/api/public/ingestion` (batches de traces) — apenas se `prefs.metrics && prefs.content` e com `OPEN_DESIGN_TELEMETRY_RELAY_URL` ou `LANGFUSE_PUBLIC_KEY/SECRET_KEY`.

### Schemas de Request/Response

Definidos como tipos TypeScript em `packages/contracts/src/`. Principais:

- `ChatRequest` — `{ message, agentId?, skillId?, attachments?, research?, ... }`
- `ChatSseEvent` — union discriminada por `event` field
- `Project` — `{ id, name, skillId, designSystemId, metadata, createdAt, updatedAt }`
- `MediaGenerateRequest` — `{ project, surface?, model, prompt, output, aspect?, ... }`
- `ApiErrorResponse` — `{ error: ApiError }` com `code: ApiErrorCode`

### Versionamento de API

Não há versionamento explícito de path (sem `/v1/`). O contrato é mantido via `packages/contracts` com alterações backward-compatible; breaking changes exigem bumps no `packages/contracts` e no `CRITIQUE_PROTOCOL_VERSION` (para o Critique Theater).

---

## 5. Protocolos e Integrações

### Protocolos Usados

| Protocolo | Uso |
|---|---|
| HTTP/1.1 (Express) | Todas as rotas REST + SSE |
| SSE | Streaming de runs de chat e Critique Theater |
| MCP (Model Context Protocol) | `od mcp` stdio server + `/api/mcp` HTTP routes |
| ACP (Agent Communication Protocol) | `acp.ts` — injeção de mensagens em runs Claude `stream-json` |
| PI RPC | `pi-rpc.ts` — protocolo específico do agente Pi |
| Sidecar IPC (UNIX socket / named pipe) | Comunicação com `apps/desktop` e `apps/packaged` via `@open-design/sidecar` |
| PKCE OAuth 2.0 | MCP server OAuth (`mcp-oauth.ts`) e xAI OAuth (`xai-oauth.ts`) |
| Prometheus text format | `/api/metrics` scrape |

### Integrações com Outros Apps e Packages

| Dependência | Como usa |
|---|---|
| `@open-design/contracts` | DTOs, error codes, analytics headers, critique protocol |
| `@open-design/sidecar` | Bootstrap, IPC transport, runtime path resolution |
| `@open-design/sidecar-proto` | Stamp fields, SIDECAR_DEFAULTS, SIDECAR_ENV |
| `@open-design/platform` | `createCommandInvocation()`, process stamp serialization |
| `@open-design/plugin-runtime` | Plugin execution runtime |
| `@open-design/registry-protocol` | Plugin registry protocol |
| `@open-design/agui-adapter` | AGUI adapter para streaming de eventos |

### Integrações Externas

| Serviço | Protocolo | Autenticação |
|---|---|---|
| Claude Code CLI | `child_process.spawn` stdin/stdout | API key via CLI env |
| OpenAI Codex CLI | `child_process.spawn` | API key via env |
| GitHub Copilot CLI | `child_process.spawn` | Token via CLI config |
| Cursor Agent | `child_process.spawn` | Config local |
| Gemini CLI | `child_process.spawn` | API key via env |
| Vercel API | HTTPS REST | Bearer token (`~/.open-design/vercel.json`) |
| Cloudflare API | HTTPS REST | Bearer token (`~/.open-design/cloudflare-pages.json`) |
| PostHog | HTTPS | `POSTHOG_KEY` |
| Langfuse | HTTPS batch ingestion | `LANGFUSE_PUBLIC_KEY` + `LANGFUSE_SECRET_KEY` |
| OpenAI Images API | HTTPS | `OD_OPENAI_API_KEY` / `OPENAI_API_KEY` |
| Volcengine ARK | HTTPS | `OD_VOLCENGINE_API_KEY` / `ARK_API_KEY` |
| xAI Grok | HTTPS + OAuth PKCE | `OD_GROK_API_KEY` / `XAI_API_KEY` |
| Google Gemini / NanoBanana | HTTPS | `OD_NANOBANANA_API_KEY` / `GOOGLE_API_KEY` |
| Black Forest Labs | HTTPS | `OD_BFL_API_KEY` |
| fal.ai | HTTPS | `OD_FAL_KEY` |
| Replicate | HTTPS | `OD_REPLICATE_API_TOKEN` |
| Kling | HTTPS | `OD_KLING_API_KEY` |
| Midjourney | HTTPS | `OD_MIDJOURNEY_API_KEY` |
| MiniMax | HTTPS | `OD_MINIMAX_API_KEY` |
| Suno | HTTPS | `OD_SUNO_API_KEY` |
| Udio | HTTPS | `OD_UDIO_API_KEY` |
| ElevenLabs | HTTPS | `OD_ELEVENLABS_API_KEY` |
| ImageRouter | HTTPS | `OD_IMAGEROUTER_API_KEY` |
| Composio | HTTPS | Credencial via `FileConnectorCredentialStore` |

---

## 6. Segurança e Autenticação

### Modelo de Autenticação

O daemon **não implementa autenticação de usuários**. Ele opera no modelo de confiança local: escuta em `127.0.0.1` (loopback) por padrão e aceita requisições apenas de origens permitidas.

**Validação de origem** (`origin-validation.ts`):

- Aceita requisições sem `Origin` (chamadas de agente / CLI).
- Para requisições com `Origin`, valida que é uma origem same-origin local (hostname loopback, porta de web UI ou de deploy `check`) ou está na lista `OD_ALLOWED_ORIGINS`.
- `OD_ALLOWED_ORIGINS` suporta apenas origens `http://` e `https://`.

**Desktop Auth** (`desktop-auth.ts`):

- `setDesktopAuthSecret()` ativa um gate de segredo para imports originados do desktop.
- `signDesktopImportToken()` / `verifyDesktopImportToken()` protegem o fluxo de import de design do desktop.
- `consumedImportNonces` garante que tokens de import não sejam reutilizados.

### Secrets e Credenciais

| Secret | Onde armazenado | Mascaramento |
|---|---|---|
| Chaves de provedores de mídia | `<dataDir>/media-config.json` | GET `/api/media/config` retorna keys mascaradas |
| Token Vercel | `~/.open-design/vercel.json` | Mascarado como `saved-vercel-token` nos responses |
| Token Cloudflare | `~/.open-design/cloudflare-pages.json` | Mascarado como `saved-cloudflare-token` |
| `agentCliEnv` (API keys de agentes) | `<dataDir>/app-config.json` | Nunca retornado fora da máquina local |
| MCP OAuth tokens | `<dataDir>/mcp-tokens/` | TTL-aware; não expostos em endpoints públicos |
| xAI OAuth tokens | `<dataDir>/` | Gerenciado por `xai-tokens.ts` |

**Proteções de path traversal:**

- `validateProjectPath()` impede escrita fora do diretório do projeto.
- `live-artifacts/artifact-writer.ts` usa `O_NOFOLLOW` para prevenir symlink attacks.
- `resolveSafeProjectAttachments()` valida que caminhos de attachment estejam dentro do CWD do projeto.

---

## 7. Performance e Escalabilidade

### Requisitos de Performance

| Operação | Meta |
|---|---|
| Startup completo (boot + SQLite + listen) | ≤ 1 s |
| Primeira resposta SSE após POST chat | ≤ 500 ms |
| GET /api/health | ≤ 10 ms |
| GET /api/projects (< 1000 projetos) | ≤ 100 ms |
| Deploy de projeto simples (Vercel) | Dependente da API Vercel; sem target fixo |

### Estratégias de Cache e Otimização

- **SQLite WAL mode**: permite leituras concorrentes sem lock; escrita serializada.
- **In-memory run registry**: `runs.ts` mantém runs ativas em `Map` com TTL de 30 min; sem I/O para consultas de status de run ativa.
- **Active context TTL**: `ACTIVE_CONTEXT_TTL_MS = 5 min` evita stale context em `active-context-routes.ts`.
- **Plugin snapshot GC**: `snapshotUnreferencedTtlDays` (padrão 30 d) e `snapshotRetentionDays` limitam crescimento de snapshots orphaned.
- **Prompt budget check**: `checkPromptArgvBudget()` valida tamanho antes do spawn — evita falhas tardias com mensagem de erro clara.
- **blake3-wasm**: hash rápido para validação de artefatos (blake3 vs SHA é ~5× mais rápido).
- **Streaming SSE com backpressure**: `json-event-stream.ts` gerencia clients; eventos históricos entregues sob demanda com `since` param.
- **chokidar para file watch**: pool de FSEvents/inotify com debounce — evita I/O desnecessário em `project-watchers.ts`.

### Escalabilidade

O daemon é projetado para **uso single-user local** (processo único por workspace). Para múltiplos namespaces (`tools-dev`), cada namespace roda um processo daemon separado com seu próprio `OD_DATA_DIR`, porta e IPC socket.

Não há suporte a deploy multi-instance ou load balancing.

---

## 8. Testes e Qualidade

### Estratégia de Testes

Testes vivem em `apps/daemon/tests/` (sibling de `src/`), não dentro de `src/`. Framework: **Vitest 2** com `vitest.config.ts`.

| Tipo | Localização | Executor |
|---|---|---|
| Unit | `apps/daemon/tests/**/*.test.ts` | `vitest run` |
| Critique conformance | `apps/daemon/src/critique/__fixtures__/run-nightly.ts` | Execução manual/nightly |
| E2E (daemon HTTP boundary) | `e2e/tests/**/*.test.ts` | `vitest run` (pacote `e2e`) |

### Cobertura Esperada

| Módulo | Prioridade de Teste |
|---|---|
| `origin-validation.ts` | Alta — segurança crítica |
| `critique/parser.ts` | Alta — parser de protocolo |
| `critique/orchestrator.ts` | Alta — estado machine |
| `runs.ts` | Alta — lifecycle de run |
| `db.ts` (migrations) | Alta — integridade do schema |
| `runtimes/prompt-budget.ts` | Alta — previne falhas de spawn |
| `home-expansion.ts` | Média — path resolution |
| `media-config.ts` | Média — resolução de credenciais |
| `projects.ts` (path traversal) | Alta — segurança |

### Convenções de Qualidade

- Novos `*.test.ts` devem ir para `tests/`, nunca para `src/`.
- Testes de integração que cruzam fronteiras web/daemon pertencem a `e2e/tests/`.
- Para bugs, criar spec vermelha em `tests/` antes de tocar em `src/`.
- Fixtures do Critique Theater em `src/critique/__fixtures__/v1/` cobrem: happy-3-rounds, malformed-unbalanced, missing-artifact.
