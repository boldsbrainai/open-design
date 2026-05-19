# PRD Técnico 360° — Open Design Web (`apps/web`)

## 1. Overview Técnico

### Stack completa

| Camada | Tecnologia | Versão |
|---|---|---|
| Framework web | Next.js | `^16.2.5` |
| UI runtime | React | `^18.3.1` |
| Linguagem | TypeScript | `^5.6.3` |
| Estilização | Tailwind CSS | `^4.1.17` |
| Bundler (dev) | Turbopack (via Next.js) | — |
| SDK IA (Anthropic) | `@anthropic-ai/sdk` | `^0.32.1` |
| SDK IA (OpenAI-compat) | `openai` | `^6.36.0` |
| Ícones | `lucide-react` | `^1.14.0` |
| Analytics | PostHog (`posthog-js`) | `^1.205.0` |
| Testes | Vitest | `^2.1.8` |
| Testes de componente | `@testing-library/react` | `^16.3.2` |
| Runtime DOM em testes | `jsdom` | `29.1.0` |
| Node.js | `~24` | — |
| Gestor de pacotes | pnpm | `10.33.2` (via Corepack) |

### Arquitetura geral

```
┌─────────────────────────────────────────────────┐
│                  Next.js 16 App Router           │
│  app/layout.tsx  (I18nProvider + AnalyticsProvider)│
│  app/[[...slug]]/page.tsx  (shell + SPA mount)   │
└───────────────────────┬─────────────────────────┘
                        │ hidrata
                        ▼
┌─────────────────────────────────────────────────┐
│                  src/App.tsx  (Client Shell)     │
│  - router (pushState)                            │
│  - config (localStorage + daemon sync)           │
│  - rota → view (HomeView / ProjectView / etc.)   │
└───────────────────────┬─────────────────────────┘
                        │ HTTP / SSE
                        ▼
              apps/daemon (Express + SQLite)
```

**Modos de output (controlados por `OD_WEB_OUTPUT_MODE`):**

| Modo | `output` Next.js | Uso |
|---|---|---|
| `export` (padrão prod) | `'export'` (static) | Daemon serve os arquivos estáticos |
| `server` | nenhum / SSR | Packaged desktop — sidecar web SSR |
| `standalone` | `'standalone'` | Sidecar com servidor Next.js rastreado |
| dev | nenhum + rewrites | `tools-dev`, portas proxiadas |

---

## 2. Backend / Servidor (Next.js Server Components e API Routes)

### Server Components vs Client Components

`apps/web` é predominantemente uma **SPA client-side**. A separação é:

| Arquivo | Tipo | Motivo |
|---|---|---|
| `app/layout.tsx` | Server Component | Metadata, viewport, script de tema anti-FOUC |
| `app/[[...slug]]/page.tsx` | Client Component (shell) | Monta `<App />` que usa hooks e estado |
| `src/App.tsx` | Client Component | Estado global, router, efeitos |
| Todos os componentes em `src/components/` | Client Components (`'use client'`) | Interatividade, SSE, localStorage |

### API Routes Next.js

`apps/web` **não expõe API Routes próprias** (`app/api/` não existe). Todas as APIs são servidas pelo `apps/daemon`. Em dev, o `next.config.ts` cria rewrites transparentes:

```ts
{ source: '/api/:path*',       destination: 'http://127.0.0.1:<OD_PORT>/api/:path*' }
{ source: '/artifacts/:path*', destination: 'http://127.0.0.1:<OD_PORT>/artifacts/:path*' }
{ source: '/frames/:path*',    destination: 'http://127.0.0.1:<OD_PORT>/frames/:path*' }
```

Em produção estática, o daemon serve diretamente sob a mesma origem.

### Server Actions

Não utilizadas. Toda mutação de dados é via `fetch()` client-side contra o daemon.

---

## 3. Frontend / Cliente

### Componentes principais e responsabilidades

#### Shell e navegação

| Componente | Responsabilidade |
|---|---|
| `App.tsx` | Shell principal. Bootstraps config, daemon health, estado global de projetos, agentes, design systems, templates, skills. Roteamento para views. |
| `EntryShell.tsx` | Layout da tela inicial (nav rail + hero). Delega a sub-views. |
| `EntryView.tsx` | Wrapper fino que passa dados e callbacks para `EntryShell`. |
| `EntryNavRail.tsx` | Rail de navegação esquerda entre Home, Projects, Tasks, Plugins, Design Systems, Integrations. |
| `WorkspaceTabsBar.tsx` | Barra de abas de projetos abertos simultâneos. |
| `QuickSwitcher.tsx` | Palette de projeto rápido (`Ctrl+K` / `Cmd+K`). |

#### Views principais

| Componente | Responsabilidade |
|---|---|
| `HomeView.tsx` | Hero da home com recent projects, starters e design templates. |
| `ProjectView.tsx` | Workspace de projeto. Orquestra chat, file viewer, critique theater. |
| `FileViewer.tsx` | Visualizador de artefatos (HTML inline, imagem, markdown, React component, live artifact). |
| `PluginsView.tsx` / `PluginDetailView.tsx` | Marketplace e detalhes de plugins. |
| `DesignSystemsTab.tsx` | Listagem e gerenciamento de design systems. |
| `IntegrationsView.tsx` | Configuração de conectores externos. |
| `TasksView.tsx` | Automações / runs agendados. |
| `MarketplaceView.tsx` | Marketplace público de plugins. |

#### Chat

| Componente | Responsabilidade |
|---|---|
| `ChatPane.tsx` | Lista de mensagens, starters, agrupamento por dia, jump button. |
| `ChatComposer.tsx` | Input de chat com suporte a anexos, menções, ctrl+enter, drag & drop. |
| `AssistantMessage.tsx` | Renderização de resposta do agente (texto, thinking, tool cards). Oculta AskUserQuestion duplicado e grupos TodoWrite. |
| `ToolCard.tsx` | Cards de tool_use / tool_result. Inclui `AskUserQuestionCard` (chips persistentes) e `TodoCard` (pinned task list). |
| `ContextChipStrip.tsx` | Chips de contexto acima do composer (arquivos, skill, design system). |

#### Artefatos e preview

| Componente | Responsabilidade |
|---|---|
| `FileViewer.tsx` | Decisão url-load vs srcDoc via `file-viewer-render-mode.ts`. Dual-iframe para evitar flash ao trocar modo. |
| `GenUISurfaceRenderer.tsx` | Renderizador de superfície GenUI adaptativa. |
| `PreviewModal.tsx` | Preview fullscreen com export controls. |
| `PreviewDrawOverlay.tsx` | Overlay de desenho/anotação sobre preview. |
| `PaletteTweaks.tsx` | Painel de paleta de cores para tweaks ao vivo no artefato. |
| `ManualEditPanel.tsx` | Edição manual de código de artefato via diff patch. |

#### Design Jury (Critique Theater)

| Componente/Arquivo | Responsabilidade |
|---|---|
| `Theater/state/reducer.ts` | Máquina de estados pura. Ações: `PanelEvent` do contrato + `__reset__`. |
| `Theater/state/sse.ts` | Gerenciador do canal SSE. `sseToPanelEvent` + validação por variante. |
| `Theater/hooks/useCritiqueStream.ts` | Assina SSE do projeto → alimenta reducer. |
| `Theater/hooks/useCritiqueReplay.ts` | Reproduz reducer a partir de `.ndjson.gz`. |
| `Theater/CritiqueTheaterMount.tsx` | Mount drop-in no ProjectView. Handshake de kill. |
| `Theater/TheaterStage.tsx` | Stage ao vivo: 5 PanelistLanes + ScoreTicker + RoundDivider + InterruptButton. |
| `Theater/TheaterCollapsed.tsx` | Badge pós-run: shipped / interrupted / failed. |
| `Theater/TheaterTranscript.tsx` | Surface de replay read-only. |

#### Configuração e settings

| Componente | Responsabilidade |
|---|---|
| `SettingsDialog.tsx` | Dialog de configurações multi-seção (API, modelo, tema, pets, notificações, orbit, privacidade). |
| `InlineModelSwitcher.tsx` | Troca rápida de modelo/protocolo inline no header. |
| `LanguageMenu.tsx` | Seletor de locale. |
| `AvatarMenu.tsx` | Menu de perfil/conta. |

### State management

`apps/web` **não usa Redux, Zustand nem Context avançado** para estado global. A estratégia é:

| Mecanismo | Uso |
|---|---|
| `useState` / `useReducer` em `App.tsx` | Estado de config, projetos, conversas, design systems, skills, agents |
| `localStorage` (`open-design:config`) | Persistência de preferências do usuário (tema, modelo, API key, etc.) |
| `useReducer` em `Theater/state/` | Estado da máquina do Critique Theater (puro, sem efeitos colaterais) |
| Props drilling com callbacks | Passagem de estado de `App.tsx` → views → componentes |
| Custom hooks (`useCritiqueStream`, `useProjectFileEvents`, etc.) | Encapsulamento de streams SSE |
| `src/router.ts` (`pushState`) | Estado de navegação (URL como source of truth) |

### UI/UX flows principais

1. **Onboarding**: `config.onboardingCompleted === false` → modal de boas-vindas → escolha de provedor.
2. **Criar projeto**: Home → NewProjectPanel → POST /api/projects → redirect para ProjectView.
3. **Chat com agente**: `ChatComposer` → submit → `streamViaDaemon` → SSE → `AssistantMessage` incremental.
4. **AskUserQuestion interativo**: agente emite `tool_use` → `AskUserQuestionCard` com chips → usuário responde → `submitChatRunToolResult` (POST `/api/runs/:id/tool-result`).
5. **Design Jury**: artefato pronto → CritiqueTheaterMount → SSE → 5 painelistas julgam → score composto.
6. **Deploy**: FileViewer → "Deploy" → dialog → `deployProjectFile()` → link público.

---

## 4. APIs e Contratos

Todos os contratos de request/response são definidos em `packages/contracts` (`@open-design/contracts`). A web **nunca define seus próprios DTOs** de API — importa sempre de `@open-design/contracts`.

### APIs consumidas do daemon

#### Projetos e conversas

| Endpoint | Método | Descrição |
|---|---|---|
| `/api/projects` | GET | Listar projetos |
| `/api/projects` | POST | Criar projeto |
| `/api/projects/:id` | GET | Detalhes do projeto |
| `/api/projects/:id` | PATCH | Atualizar projeto (nome, metadata) |
| `/api/projects/:id` | DELETE | Excluir projeto |
| `/api/projects/:id/conversations` | GET | Listar conversas |
| `/api/projects/:id/conversations` | POST | Criar conversa |
| `/api/projects/:id/conversations/:cid` | PATCH | Atualizar conversa |
| `/api/projects/:id/conversations/:cid` | DELETE | Excluir conversa |
| `/api/projects/:id/conversations/:cid/messages` | GET | Listar mensagens |
| `/api/projects/:id/conversations/:cid/messages` | POST | Salvar mensagem |
| `/api/projects/:id/tabs` | GET / PUT | Estado de abas abertas |

#### Chat runs (SSE)

| Endpoint | Método | Descrição |
|---|---|---|
| `/api/runs` | POST | Iniciar run de chat (`ChatRequest`) → SSE stream |
| `/api/runs/:id` | GET | Reattach a run em andamento |
| `/api/runs/:id/status` | GET | Status do run (`ChatRunStatusResponse`) |
| `/api/runs/:id/tool-result` | POST | Enviar `tool_result` para run `stream-json` ativo |
| `/api/runs` (list active) | GET | Listar runs ativos do projeto |

**Eventos SSE do run:**

| `event` | Descrição |
|---|---|
| `status` | Status do run (starting, running, done, error) |
| `text_delta` | Chunk de texto do agente |
| `thinking_delta` | Chunk de thinking interno |
| `tool_use` | Chamada de ferramenta pelo agente |
| `tool_result` | Resultado de ferramenta |
| `usage` | Contagem de tokens input/output |
| `raw` | Evento bruto do agente (debug) |

#### Arquivos de projeto

| Endpoint | Método | Descrição |
|---|---|---|
| `/api/projects/:id/files` | GET | Listar arquivos |
| `/api/projects/:id/files/:path` | GET | Conteúdo raw do arquivo |
| `/api/projects/:id/files/:path/preview` | GET | Preview HTML processado |
| `/api/projects/:id/files/:path/text` | GET / PUT | Ler/escrever texto |
| `/api/projects/:id/files/:path/rename` | POST | Renomear arquivo |
| `/api/projects/:id/upload` | POST | Upload de arquivos |

#### Eventos SSE de projeto

| Endpoint | Descrição |
|---|---|
| `/api/projects/:id/events` | Stream de eventos (file-changed, live-artifact, conversation-created) |

Backoff exponencial: 1s → 30s. Reset ao receber evento `ready`.

#### Design Systems, Skills, Agents, Templates

| Endpoint | Descrição |
|---|---|
| `/api/design-systems` | Listar design systems disponíveis |
| `/api/design-systems/:id` | Detalhes do design system |
| `/api/skills` | Listar skills |
| `/api/skills/:id` | Detalhes da skill |
| `/api/agents` | Listar agentes disponíveis |
| `/api/design-templates` | Listar templates de design |
| `/api/templates` | Listar templates de projeto |
| `/api/prompt-templates` | Listar prompt templates |

#### Live Artifacts

| Endpoint | Método | Descrição |
|---|---|
| `/api/projects/:id/live-artifacts` | GET | Listar live artifacts |
| `/api/projects/:id/live-artifacts/:aid` | GET | Detalhes |
| `/api/projects/:id/live-artifacts/:aid/code` | GET | Código fonte |
| `/api/projects/:id/live-artifacts/:aid/refresh` | POST | Forçar refresh |
| `/api/projects/:id/live-artifacts/:aid/refreshes` | GET | Histórico de refreshes |

#### Deploy

| Endpoint | Método | Descrição |
|---|---|
| `/api/projects/:id/deploy` | GET | Config de deploy |
| `/api/projects/:id/deploy` | PUT | Atualizar config |
| `/api/projects/:id/deploy/:provider/files/:path` | POST | Deploy de arquivo |
| `/api/projects/:id/deployments` | GET | Histórico de deploys |
| `/api/cloudflare-pages/zones` | GET | Zonas CF Pages disponíveis |

#### Critique Theater

| Endpoint | Método | Descrição |
|---|---|
| `/api/projects/:id/critique/:runId/interrupt` | POST | Interromper run do Design Jury |

#### Configuração e utilitários

| Endpoint | Método | Descrição |
|---|---|
| `/api/config` | GET / PATCH | Config do daemon (provedor de mídia, Composio, etc.) |
| `/api/health` | GET | Liveness check do daemon |
| `/api/version` | GET | Versão do daemon e app |
| `/api/connection-test` | POST | Testar conexão com provedor IA |

### Schemas principais (de `@open-design/contracts`)

```ts
// Iniciar um run de chat
interface ChatRequest {
  projectId: string;
  conversationId: string;
  messages: ChatMessage[];
  systemPrompt?: string;
  agentId?: string | null;
  skillId?: string | null;
  designSystemId?: string | null;
  researchOptions?: ResearchOptions;
  // ...
}

// Evento SSE de início de run
interface ChatSseStartPayload {
  runId: string;
  streamFormat: 'agent' | 'stdout';
}

// Evento SSE de agente (formato 'agent')
type ChatSseEvent = 
  | { type: 'status'; status: ChatRunStatus }
  | { type: 'text_delta'; text: string }
  | { type: 'thinking_delta'; thinking: string }
  | { type: 'tool_use'; toolUseId: string; name: string; input: unknown }
  | { type: 'tool_result'; toolUseId: string; content: string }
  | { type: 'usage'; inputTokens: number; outputTokens: number }
  | { type: 'error'; error: SseErrorPayload };
```

---

## 5. Protocolos e Integrações

### Comunicação com o daemon (HTTP + SSE)

- **HTTP REST**: `fetch()` puro, sem biblioteca. Todas as chamadas em `src/providers/registry.ts`, `src/state/projects.ts`, `src/providers/daemon.ts`.
- **SSE de runs**: `fetch()` com leitura manual de stream (`response.body.getReader()`), parser em `src/providers/sse.ts` (`parseSseFrame`). Não usa `EventSource` nativo para runs (permite controle de reconexão e suporte a body).
- **SSE de eventos de projeto**: `EventSource` nativo (`/api/projects/:id/events`) com backoff exponencial gerenciado em `src/providers/project-events.ts`.
- **Headers de analytics**: injetados em toda chamada `same-origin /api/` via `globalThis.fetch` monkey-patch no `AnalyticsProvider`.

### Provedores de IA suportados

| Arquivo | Protocolo | Exemplos |
|---|---|---|
| `providers/anthropic.ts` | Anthropic Messages API | Claude Sonnet/Opus/Haiku |
| `providers/openai-compatible.ts` | OpenAI Chat Completions | OpenAI, Groq, Together, Perplexity, DeepSeek, LiteLLM |
| `providers/azure-compatible.ts` | Azure OpenAI | Azure ChatCompletion endpoints |
| `providers/google-compatible.ts` | Google Generative AI | Gemini |
| `providers/ollama-compatible.ts` | Ollama API | Modelos locais |
| `providers/anthropic-compatible.ts` | Anthropic-compat | Vertex, Bedrock, DeepSeek |

**Modo daemon** (`config.mode === 'daemon'`, padrão): todo streaming passa pelo daemon via `/api/runs`.  
**Modo direto** (`config.mode === 'direct'`): web chama diretamente o provedor de IA (sem daemon para o LLM).

### i18n — sistema de internacionalização

- **19 locales**: `ar`, `de`, `en`, `es-ES`, `fa`, `fr`, `hu`, `id`, `it`, `ja`, `ko`, `pl`, `pt-BR`, `ru`, `th`, `tr`, `uk`, `zh-CN`, `zh-TW`.
- **Arquivo central de tipos**: `src/i18n/types.ts` — interface `Dict` com todas as chaves dot-namespaced.
- **Dicionários**: `src/i18n/locales/<locale>.ts` — cada locale implementa 100% de `Dict`.
- **Provider**: `I18nProvider` em `app/layout.tsx` (Server Component wrapper).
- **Hook de uso**: `useT()` — retorna função `t(key, vars?)`.
- **Detecção**: via `navigator.language` com fallback para `'en'`.
- **RTL**: suporte a árabe (`ar`) e persa (`fa`) via CSS logical properties.

### Sistema de temas e design systems

- **Temas**: `light` | `dark` | `system` via `data-theme` em `<html>`.
- **Anti-FOUC**: script inline em `app/layout.tsx` aplica tema antes da hidratação.
- **Accent color**: CSS custom properties `--accent`, `--accent-strong`, `--accent-soft`, `--accent-tint`, `--accent-hover` via `color-mix()`. Padrão: `#c96442`.
- **Design Systems**: importados do daemon (`/api/design-systems`), injetados no system prompt do agente como contexto de marca.

### Integração com Electron (desktop)

- `src/types/electron.d.ts` declara `window.electronAPI`.
- O web não detecta porta do daemon — recebe via contexto de sidecar.
- `sidecar/index.ts` bootstrapa o runtime Next.js SSR para o desktop packaged.

### Telemetria (PostHog)

- `src/analytics/provider.tsx` — `AnalyticsProvider` com PostHog.
- `src/analytics/events.ts` — typed wrappers para cada evento rastreado.
- Opt-out via `config.telemetry.metrics === false` (`setConsent(false)`).
- `anonymousId` baseado em `installationId` do daemon; rotação via `setIdentity()`.

---

## 6. Segurança e Autenticação

### Modelo de autenticação

`apps/web` **não tem autenticação própria** — é uma SPA local que assume acesso à instância daemon do usuário. O modelo de segurança é:

- **Isolamento por processo**: daemon roda localmente; web e daemon compartilham a mesma máquina do usuário.
- **API keys**: armazenadas apenas em `localStorage` (cliente) e repassadas ao daemon via PATCH `/api/config`. **Nunca logadas, nunca em URLs.**
- **Contexto não-HTTPS**: suporte explícito para self-hosting em LAN (Docker, unRAID) sem quebrar `crypto.randomUUID()`.

### Proteções implementadas

| Proteção | Implementação |
|---|---|
| Headers de analytics nunca vazam para terceiros | `isSameOriginApiCall()` em `analytics/provider.tsx` |
| Sanitização de HTML em artefatos | `renderMarkdownToSafeHtml()` em `artifacts/markdown.ts` |
| Sandbox de iframes | `srcDoc` com `sandbox` attribute; `htmlNeedsSandboxShim()` para HTML externo |
| XSS no script de tema | `dangerouslySetInnerHTML` comentado com `biome-ignore` justificado (anti-FOUC) |
| Sem credenciais em logs ou URL params | Apenas `localStorage` + HTTP body PATCH |

---

## 7. Performance e Escalabilidade

### Estratégias de output

| Modo | Estratégia | Detalhes |
|---|---|---|
| Produção estática | `output: 'export'` | Daemon serve HTML/JS/CSS estático. Sem SSR overhead. |
| Packaged server | SSR Next.js | Sidecar web própio com SSR completo para desktop. |
| Dev | Turbopack | Bundler rápido, HMR, proxy de `/api/*` integrado. |

### Lazy loading e code splitting

- Next.js App Router faz code splitting automático por rota.
- Componentes pesados (Theater, FileViewer, ManualEditPanel) são importados sob demanda.

### Estratégias de UI

- **Dual-iframe no FileViewer**: dois iframes montados simultaneamente; visibilidade trocada via CSS para evitar flash de recarga ao alternar modo de render.
- **Componentes montados com toggle CSS**: elementos condicionais (ex: `chat-jump-btn`) ficam montados e exibidos/ocultados por classe CSS para preservar transição de saída.
- **Anti-FOUC**: script inline aplica tema antes de qualquer paint.
- **Debounce de config**: `syncConfigToDaemon` aplica debounce para não chamar PATCH a cada keystroke.
- **Truncagem de histórico de chat**: mensagens anteriores truncadas a 12.000 chars antes de enviar ao agente (`MAX_TRANSCRIPT_MESSAGE_CHARS`).
- **Aviso de token alto**: alerta quando input > 200.000 tokens (`HIGH_INPUT_TOKEN_WARNING_THRESHOLD`).

### Animações

- Ease padrão: `cubic-bezier(0.23, 1, 0.32, 1)`.
- Duração: entrada ~200ms, saída ~140ms.
- Accordion: `grid-template-rows: 0fr → 1fr` com classe `.accordion-collapsible`.
- `scale` nunca inicia de 0; mínimo `scale(0.9)`.

---

## 8. Testes e Qualidade

### Estratégia de testes

| Camada | Ferramenta | Localização | Ambiente |
|---|---|---|---|
| Unitários / integração | Vitest `^2.1.8` | `apps/web/tests/**/*.test.{ts,tsx}` | `node` |
| Componentes React | `@testing-library/react ^16.3.2` + jsdom `29.1.0` | `apps/web/tests/` | `node` + jsdom |
| E2E / UI automation | Playwright | `e2e/ui/` (fora deste app) | browser |

**Configuração (`vitest.config.ts`):**
```ts
export default defineConfig({
  test: {
    environment: 'node',
    include: ['tests/**/*.test.{ts,tsx}'],
  },
});
```

### Checagens de qualidade

| Checagem | Comando |
|---|---|
| TypeScript | `pnpm --filter @open-design/web typecheck` |
| Testes | `pnpm --filter @open-design/web test` |
| Build | `pnpm --filter @open-design/web build` |
| Guard (JS ilegal, etc.) | `pnpm guard` (raiz) |
| Typecheck global | `pnpm typecheck` (raiz) |

### Invariantes de qualidade

- `src/` é source-only: **nenhum** `*.test.ts` / `*.test.tsx` dentro de `src/`.
- `apps/web` não importa `apps/daemon/src/**` — integração somente via HTTP.
- Toda string visível ao usuário passa por `t(key)` — hardcoded English reprova review.
- Toda locale implementa 100% do `Dict` — falta de chave gera erro de typecheck.
- Novos arquivos `.js`/`.mjs`/`.cjs` precisam de razão explícita e passam `pnpm guard`.
