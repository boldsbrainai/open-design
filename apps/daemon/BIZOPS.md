# BizOps Blueprint — Daemon (`@open-design/daemon`)

## 1. Visão Geral

### Propósito e Valor de Negócio

O `daemon` é o núcleo privilegiado do Open Design. Ele serve como:

- **Servidor de API local** (Express + SQLite) que mantém estado de projetos, conversas, artefatos e histórico de agentes.
- **Orquestrador de agentes de IA**: spawn, streaming, cancelamento e rastreamento de runs de Claude, Codex, Copilot, Cursor, Gemini, DeepSeek, Grok/Hermes, Pi, Qoder/Qwen, Devin, Kilo, Kimi, Kiro, OpenCode, Vibe e outros.
- **Motor de geração de mídia**: imagem, vídeo e áudio via APIs de provedores externos (OpenAI, Volcengine, xAI Grok, Google, Black Forest Labs, fal, Replicate, Kling, Midjourney, MiniMax, Suno, Udio, ElevenLabs).
- **Servidor MCP stdio** (`od mcp`): expõe ferramentas de projeto para coding agents externos (Claude Code, Cursor, Zed) sem o ciclo export-zip-import.
- **Plano de controle de deploy**: publica projetos HTML diretamente no Vercel e no Cloudflare Pages.
- **Host de Critique Theater (Design Jury)**: orquestra painelistas de IA em múltiplas rodadas de avaliação de design com pontuação Prometheus.

O daemon é o único componente com acesso ao disco local, banco de dados, CLI de agentes e chaves de API — toda a web UI e o shell Electron consomem seus endpoints.

### Stakeholders e Usuários

| Papel | Como usa o daemon |
|---|---|
| Usuário final (web) | Indiretamente: a web UI chama `/api/*` |
| Usuário desktop (Electron) | Descoberta de URL via sidecar IPC; consume `/api/*` |
| Coding agent externo | `od mcp` (stdio) ou `/api/mcp` (HTTP) |
| Operador / DevOps | Variáveis de ambiente, `pnpm tools-dev`, packaged release |
| Equipe de Produto | OKRs de qualidade de design (Critique Theater), deploy coverage |
| Mantenedor | `apps/daemon/src/`, `apps/daemon/tests/`, CI |

### Posição no Ecossistema Open Design

```
┌──────────────────┐     HTTP /api/*      ┌─────────────────────┐
│  apps/web        │ ──────────────────► │  apps/daemon        │
│  Next.js 16      │                     │  Express + SQLite   │
└──────────────────┘                     │  Node.js ~24        │
                                         └──────────┬──────────┘
┌──────────────────┐     Sidecar IPC             ▼ ▲
│  apps/desktop    │ ◄───────────────── spawn CLI agents
│  Electron        │                    (claude, codex, copilot…)
└──────────────────┘
┌──────────────────┐     od mcp stdio
│  Coding agent    │ ◄──────────────────── od mcp
│  (Claude Code…)  │
└──────────────────┘
```

O daemon é o único app com acesso ao sistema de arquivos local, banco SQLite, APIs de LLMs e chaves de provedores de mídia.

---

## 2. Objetivos de Negócio (OKRs)

### O1 — Fluxo zero-latência de design assistido por IA

- **KR1.1**: Tempo médio de startup do daemon ≤ 1 s (boot + SQLite open + port listen).
- **KR1.2**: Primeira resposta de streaming SSE do agente entregue ao browser ≤ 500 ms após POST `/api/projects/:id/chat`.
- **KR1.3**: ≥ 95 % das runs de chat completam sem erro de timeout em sessões típicas.

### O2 — Qualidade de design verificável via Critique Theater

- **KR2.1**: Critique Theater disponível para 100 % das skills com `od.critique` no frontmatter.
- **KR2.2**: Taxa de runs com status `shipped` ≥ 70 % das execuções iniciadas (excluindo interrupções manuais).
- **KR2.3**: Métricas Prometheus expostas em `/api/metrics` com ≥ 9 séries `open_design_critique_*`.

### O3 — Cobertura de provedores de mídia

- **KR3.1**: ≥ 15 provedores de mídia suportados (imagem, vídeo, áudio).
- **KR3.2**: Fallback via alias de modelo (`OD_MEDIA_MODEL_ALIASES`) sem alteração de código.

### O4 — Observabilidade e telemetria opt-in

- **KR4.1**: PostHog e Langfuse desativados por padrão; nenhum dado enviado sem `prefs.metrics = true`.
- **KR4.2**: Traces de run completos (prompt, output, usage) enviados ao Langfuse em ≤ 3 s após term da run.

### O5 — Cobertura de deploy

- **KR5.1**: Deploy para Vercel e Cloudflare Pages operacional em ≤ 5 trocas de mensagens com o agente.

---

## 3. Fluxos Operacionais

### 3.1 Ciclo de Vida de uma Run de Chat

```
1. Web POST /api/projects/:id/chat  (ChatRequest)
2. daemon cria run in-memory (runs.ts)
3. daemon resolve agentDef + snapshot de plugin
4. daemon compõe system prompt (prompts/system.ts)
5. daemon faz spawn do CLI do agente via child_process.spawn
6. daemon alimenta stdin com prompt (text ou stream-json)
7. daemon parseia stdout → SSE events → /api/runs/:runId/events
8. Web consome SSE; exibe tokens em tempo real
9. Ao encerrar: daemon persiste mensagem no SQLite, dispara Langfuse trace
10. (opcional) Critique Theater inicia nova run de avaliação
```

### 3.2 Critique Theater (Design Jury)

```
1. Skill com od.critique habilitado
2. daemon.critique.orchestrator inicia run com panel de panelistas
3. Para cada rodada: spawn de CLI com prompt de panelista
4. Parser detecta <CRITIQUE_RUN>, <PANELIST>, <SHIP>
5. Scores calculados → composite score
6. Se score >= threshold OU max_rounds atingido → SHIP
7. Artifact gravado em .od/projects/:id/
8. Métricas emitidas para Prometheus
```

### 3.3 Geração de Mídia

```
1. Web / CLI: POST /api/media/generate  (ou `od media generate`)
2. daemon lê chave do provedor (env → media-config.json)
3. daemon dispara request para API do provedor
4. Resposta gravada como arquivo no projeto
5. MediaTask persistida no SQLite (media_tasks)
```

### 3.4 Deploy para Vercel / Cloudflare Pages

```
1. Web POST /api/deploy/:provider
2. daemon lê token em ~/.open-design/<provider>.json
3. daemon empacota arquivos HTML + assets do projeto
4. deploy.ts chama API do provedor
5. deployment row inserida/atualizada no SQLite
6. URL devolvida para a web
```

### 3.5 Responsabilidades entre times

| Área | Time responsável | Depende de |
|---|---|---|
| Runtimes de agentes (`runtimes/defs/`) | Eng. de IA | Mantenedor de cada CLI externo |
| Schema SQLite (`db.ts`) | Backend | `packages/contracts` |
| Rotas HTTP (`*-routes.ts`) | Backend | `packages/contracts` |
| Critique Theater | Eng. de Qualidade | `packages/contracts/critique` |
| Provedores de mídia | Eng. de Mídia | APIs externas |
| Sidecar / IPC | Eng. de Desktop | `packages/sidecar-proto` |

---

## 4. Métricas e KPIs

### Saúde do Processo

| Métrica | Onde medir | Meta |
|---|---|---|
| Tempo de startup | Logs `[od] listening on` | ≤ 1 s |
| Erros 5xx por minuto | `/api/health` + logs | < 1 % |
| Corridas ativas simultâneas | `runs.ts` in-memory map | monitorar pico |

### Critique Theater (Prometheus `/api/metrics`)

| Série | Tipo | Descrição |
|---|---|---|
| `open_design_critique_runs_total` | Counter | Runs por status/adapter/skill |
| `open_design_critique_rounds_total` | Counter | Rodadas processadas |
| `open_design_critique_round_duration_ms` | Histogram | Duração por rodada |
| `open_design_critique_composite_score` | Histogram | Score composto por rodada |
| `open_design_critique_degraded_adapters` | Gauge | Adaptadores degradados (TTL 24h) |
| `open_design_critique_errors_total` | Counter | Erros por tipo |

### Qualidade de Agentes

| KPI | Fonte |
|---|---|
| Taxa de run `succeeded` | SQLite `messages.events_json` |
| Token usage médio por run | Langfuse traces |
| Latência de primeira resposta SSE | Logs de run |

---

## 5. Riscos e Mitigações

| Risco | Probabilidade | Impacto | Mitigação |
|---|---|---|---|
| CLI de agente não instalado na máquina do usuário | Alta | Crítico | `detectAgents()` lista ausentes; UI exibe link `installUrl` |
| Prompt excede budget de argv/stdin | Média | Alto | `checkPromptArgvBudget()` retorna erro antes do spawn |
| API key de provedor de mídia inválida ou expirada | Alta | Médio | GET `/api/media/config` mascarado; PUT atualiza em runtime |
| Token de deploy Vercel/Cloudflare expirado | Média | Médio | `/api/deploy/:provider/test` valida antes do deploy |
| Banco SQLite corrompido | Baixa | Crítico | WAL mode; migração idempotente em `db.ts:migrate()` |
| CSRF / host header injection | Baixa | Alto | `origin-validation.ts` valida Origin + Host; bind 127.0.0.1 por padrão |
| Path traversal em project files | Baixa | Alto | `validateProjectPath()` + `O_NOFOLLOW` em `artifact-writer.ts` |
| Critique adapter degradado silenciosamente | Média | Médio | `adapter-degraded.ts` rastreia degradação com TTL 24h; métricas Prometheus alertam |
| OD_DATA_DIR apontando para diretório de outro namespace | Baixa | Alto | `resolveDataDir()` valida gravabilidade; `legacy-data-migrator.ts` recusa merge se destino não estiver vazio |

---

## 6. Roadmap de Capacidades

### Capacidades Atuais (v0.7.0)

- [x] Spawn de ≥ 15 runtimes de agente de IA
- [x] Streaming SSE para chat (claude, codex, copilot, qoder, cursor, etc.)
- [x] ACP (Agent Communication Protocol) e PI RPC para agentes interativos
- [x] Sqlite-backed: projetos, conversas, mensagens, tabs, deployments, rotinas, media tasks
- [x] Critique Theater com 5 papéis de panelista e Prometheus
- [x] ≥ 15 provedores de mídia (imagem, vídeo, áudio)
- [x] Deploy para Vercel e Cloudflare Pages
- [x] Live Artifacts (artefatos HTML com refresh por agente)
- [x] Memória por projeto (extração automática via LLM)
- [x] MCP server stdio (`od mcp`) + HTTP MCP routes
- [x] Skills registry (SKILL.md frontmatter + craft)
- [x] Design Systems registry
- [x] Plugin system (snapshots imutáveis, dev loop, GC)
- [x] Routines / orbit (tarefas agendadas)
- [x] Connectors externos (Composio)
- [x] xAI OAuth (PKCE)
- [x] Telemetria opt-in: PostHog + Langfuse relay

### Capacidades Planejadas

- [ ] Critique Theater v2: cast config por skill (pesos de panelista via frontmatter)
- [ ] Settings UI para edição de pesos por projeto
- [ ] Critique conformance nightly (automação de avaliação de adaptadores)
- [ ] Snapshot GC worker (Phase 5 do plugin system)
- [ ] Client_id próprio para xAI OAuth (substituir PoC de Hermes)
