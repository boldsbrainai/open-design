# BizOps Blueprint — Open Design Web (`apps/web`)

## 1. Visão Geral

### Propósito e valor de negócio

`apps/web` é o runtime web principal do Open Design: uma SPA Next.js 16 + React 18 que entrega a interface completa de chat-driven design ao usuário final. Ele transforma prompts de linguagem natural em artefatos de design renderizados ao vivo (HTML/CSS, protótipos, decks, imagens, vídeos, áudio), com revisão colaborativa via Design Jury (Critique Theater) e integração com dezenas de provedores de IA.

O valor de negócio central é reduzir o ciclo prompt → artifact → feedback de horas para segundos, tornando a criação de design acessível a qualquer pessoa com uma ideia.

### Stakeholders e usuários

| Papel | Interesse |
|---|---|
| Usuário final (designer/dev/PM) | Criar e iterar artefatos de design via chat |
| Operador de instância (self-hosted) | Configurar provedores de IA, design systems locais |
| Equipe de produto Open Design | Evolução de features, qualidade, telemetria |
| Daemon (`apps/daemon`) | Fornece todas as APIs REST/SSE consumidas pelo web |
| Electron shell (`apps/desktop`) | Embute o web como surface nativa |

### Posição no ecossistema Open Design

```
Usuário
  │
  ▼
apps/web  ──── HTTP/SSE ────►  apps/daemon  ──►  Agentes IA
  │                                │
  │                                ▼
  │                           .od/ (SQLite, artefatos)
  │
  ▼
apps/desktop (Electron shell, opcional)
```

- `apps/web` é o único app com interface visual de usuário.
- Toda persistência (projetos, conversas, artefatos) vive no daemon.
- Em produção estática (`output: 'export'`) o daemon serve os assets do web; em modo packaged, um sidecar Next.js SSR próprio faz o serving.

---

## 2. Objetivos de Negócio (OKRs)

### Objetivo 1 — Experiência de criação fluida e responsiva

- **KR1.1** Latência percebida do primeiro token de resposta do agente < 2 s em condições normais de rede local.
- **KR1.2** Artefatos HTML/CSS renderizados inline sem recarregamento de página.
- **KR1.3** Transições UI com `cubic-bezier(0.23, 1, 0.32, 1)`, entrada ~200 ms, saída ~140 ms.

### Objetivo 2 — Alcance e acessibilidade global

- **KR2.1** Interface disponível em 19 locales (ar, de, en, es-ES, fa, fr, hu, id, it, ja, ko, pl, pt-BR, ru, th, tr, uk, zh-CN, zh-TW).
- **KR2.2** Suporte a RTL (árabe, persa) sem quebra de layout.
- **KR2.3** Funciona em contextos não-HTTPS (self-hosted em LAN, Docker, unRAID).

### Objetivo 3 — Extensibilidade e integração com provedores

- **KR3.1** Suporte a qualquer provedor compatível com OpenAI, Anthropic, Azure, Google ou Ollama via configuração no UI de Settings.
- **KR3.2** Design Systems customizados importáveis via GitHub ou upload local.
- **KR3.3** Plugins descobríveis e instaláveis via Marketplace sem restart.

### Objetivo 4 — Qualidade e manutenibilidade

- **KR4.1** Cobertura de tipos TypeScript estrita (`strict: true`, `noUncheckedIndexedAccess: true`).
- **KR4.2** Testes Vitest executam em CI sem dependência de daemon em execução.
- **KR4.3** Adição de nova locale não exige alteração de código; apenas novo arquivo em `src/i18n/locales/`.

---

## 3. Fluxos Operacionais

### Fluxo 1 — Criar artefato via chat

```
Usuário digita prompt no ChatComposer
  │
  ▼
ProjectView.tsx (streamViaDaemon)
  │
  ▼
POST /api/runs  ──SSE──►  daemon executa agente IA
  │
  ├── eventos: text_delta, tool_use, tool_result, usage
  │
  ▼
AssistantMessage.tsx renderiza resposta incremental
  │
  ▼
FileViewer.tsx exibe artefato HTML ao vivo (srcDoc / url-load)
```

**Responsabilidades:**
- Web: compor prompt, montar histórico da conversa, exibir artefato.
- Daemon: executar agente, persistir mensagens, servir artefatos.

### Fluxo 2 — Design Jury (Critique Theater)

```
Usuário clica "Run Design Jury" em ProjectView
  │
  ▼
POST /api/projects/:id/critique  (via daemon)
  │
  ▼
SSE /api/projects/:id/events  ──►  Theater/state/sse.ts
  │
  ├── PanelEvent → Theater/state/reducer.ts
  │
  ▼
TheaterStage.tsx (PanelistLane, ScoreTicker, RoundDivider)
  │
  ▼
TheaterCollapsed.tsx (shipped | interrupted | failed)
```

### Fluxo 3 — Configurar provedor de IA

```
Usuário abre SettingsDialog → aba do protocolo
  │
  ├── Seleciona provider (Anthropic, OpenAI, Azure, etc.)
  ├── Insere API key
  ├── Testa conexão (POST /api/connection-test ou local)
  │
  ▼
state/config.ts → salva em localStorage ('open-design:config')
  │
  ▼
syncConfigToDaemon() → PATCH /api/config
```

### Fluxo 4 — Navegação SPA (router customizado)

Roteamento via `pushState` sem react-router. Rotas suportadas:
- `/` → HomeView / EntryShell
- `/projects/:id` → ProjectView
- `/projects/:id/conversations/:cid` → conversa específica
- `/projects/:id/files/:path` → arquivo aberto
- `/plugins`, `/design-systems`, `/integrations`, `/tasks` → sub-views da entrada

### Fluxo 5 — Deploy de artefato

```
Usuário clica "Deploy" no FileViewer
  │
  ├── Vercel Self (padrão): PUT /api/projects/:id/deploy/vercel-self
  ├── Cloudflare Pages: PUT /api/projects/:id/deploy/cloudflare-pages
  │
  ▼
DeployConfigResponse → link para preview ao vivo
```

---

## 4. Métricas e KPIs

| KPI | Descrição | Fonte |
|---|---|---|
| First Token Latency | Tempo até o primeiro `text_delta` no SSE | PostHog / analytics |
| Artifact Render Rate | % de artefatos renderizados com sucesso sem erro | Frontend error boundary |
| Locale Coverage | % de chaves presentes em todas as 19 locales | TypeCheck CI |
| Session Length | Duração média de sessão por usuário | PostHog |
| Chat Run Completion Rate | % de runs que terminam com `stop` vs `error` | daemon logs / telemetria |
| Provider Config Success | % de testes de conexão bem-sucedidos no Settings | `trackSettingsByokTestResult` |
| Critique Theater Usage | Runs de Design Jury iniciados por semana | PostHog `critique_theater.*` |

---

## 5. Riscos e Mitigações

| Risco | Impacto | Mitigação |
|---|---|---|
| Daemon indisponível | UI quebrada, sem dados | `daemonIsLive()` probe na inicialização; `fail-soft` em todas as chamadas fetch (retorna `[]` / `null`) |
| Contexto HTTP inseguro (LAN) | `crypto.randomUUID()` indisponível | `src/utils/uuid.ts` tem fallback para `crypto.getRandomValues` / `Math.random` |
| CORS em dev sem proxy | Chamadas `/api/` bloqueadas | `next.config.ts` proxia `/api/*`, `/artifacts/*`, `/frames/*` via rewrite para `OD_PORT` |
| Credenciais de API no localStorage | Exposição por XSS | Chaves armazenadas apenas no lado cliente/daemon; nenhuma chave trafega em logs |
| Vazamento de headers de analytics para terceiros | Privacidade | `isSameOriginApiCall()` garante que headers de analytics só vão para `/api/` same-origin |
| Locale faltando chave de tradução | Erro de typecheck | `Dict` tipado força todas as locales a implementar 100% das chaves |
| Flash de tema incorreto (FOUC) | UX ruim | Script inline antes da hidratação React lê `localStorage` e seta `data-theme` imediatamente |
| Regressão no comportamento do iframe | Artefato não renderiza | `file-viewer-render-mode.ts` controla decisão url-load vs srcDoc com regras explícitas por feature |

---

## 6. Roadmap de Capacidades

### Capacidades atuais (v0.7.0)

- Chat-driven design com streaming SSE
- Visualizador de artefatos HTML inline (srcDoc + url-load)
- Design Jury / Critique Theater com 5 papéis de painel
- 19 locales com suporte RTL (ar, fa)
- Settings multi-protocolo: Anthropic, OpenAI-compat, Azure, Google, Ollama, LiteLLM
- Deploy direto para Vercel Self e Cloudflare Pages
- Sistema de pets (Mochi, custom) como companion de workspace
- Orbit — briefings agendados via skill
- Plugins (descoberta, instalação, execução via PluginLoopHome)
- Quick Switcher de projetos com histórico recente
- Export de artefatos: PDF, ZIP, HTML, JSX, imagem
- Comentários e anotações em previews
- Design Systems customizados (GitHub / local)
- Prompt Templates com preview modal
- Memória de agente inline (`MemoryModelInline`, `MemorySection`)
- Integração com Composio para conectores externos

### Capacidades planejadas

- Streaming de vídeo/áudio com modelos especializados (Seedance, Sora, Kling, Veo, ElevenLabs)
- Design Jury v2: peso do papel Designer ativado (atualmente fixo em 0.0)
- Internationalization estendida com novos locales
- GenUI Surface Renderer para geração de UI adaptativa
- Live Artifacts com auto-refresh e badges de status ao vivo
- Replay de transcripts de critique (`.ndjson.gz`)
