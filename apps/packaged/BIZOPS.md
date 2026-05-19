# BizOps Blueprint — Packaged

## 1. Visão Geral

`apps/packaged` é o **ponto de entrada thin** do runtime Electron empacotado do Open Design. Ele não contém lógica de produto: inicia os sidecars de daemon e web, registra o protocolo `od://`, e delega toda a UI e lógica de host para `@open-design/desktop`. Também expõe um modo headless (`./headless`) para o instalador Linux sem janela.

## 2. Objetivos de Negócio (OKRs)

### Objetivo 1 — Inicialização confiável em todas as plataformas
- **KR1:** Taxa de falha no cold boot < 0,5 % em macOS/Windows/Linux.
- **KR2:** Daemon sidecar pronto em < 35 s (budget `DAEMON_STATUS_TIMEOUT_MS`).
- **KR3:** Zero regressões de path entre releases (namespaces isolados).

### Objetivo 2 — Segurança dos segredos em runtime
- **KR1:** `LANGFUSE_SECRET_KEY` nunca embarcado no bundle — apenas a URL pública do relay de telemetria.
- **KR2:** Env de filhos filtrada por `PACKAGED_CHILD_ENV_ALLOWLIST`; segredos de providers (`*_API_KEY`, `*_TOKEN`) opcionalmente repassados sob controle explícito.
- **KR3:** `requireDesktopAuth: true` ligado na entrada Electron desde o primeiro request.

### Objetivo 3 — Canais de release independentes e rastreáveis
- **KR1:** Cada canal (stable, beta, preview) usa namespace de paths isolado.
- **KR2:** Identity file (`desktop-root.json`) gravado com `pid`, `ppid`, `stamp` e heartbeat a cada 5 s.
- **KR3:** `tools-pack` testa dois namespaces simultâneos sem colisão.

## 3. Fluxos Operacionais

### Inicialização (modo Electron)
1. `readPackagedConfig()` — lê `open-design-config.json` de `process.resourcesPath` (baked por `tools-pack`).
2. Resolve namespace via stamp de argv ou `config.namespace`.
3. `ensurePackagedNamespacePaths()` — cria diretórios de dados/logs/runtime.
4. `applyPackagedElectronPathOverrides()` — redireciona `userData`, `logs`, `cache`, `appData` do Electron para caminhos namespace-scoped.
5. `writePackagedDesktopIdentity()` — grava identity JSON com heartbeat.
6. `startPackagedSidecars()` — spawna daemon e web como processos filhos, aguarda IPC ready.
7. `registerOdProtocol()` — registra `od://` como protocolo privilegiado; proxies para o web sidecar.
8. `runDesktopMain()` — entrega o controle para `@open-design/desktop`.

### Inicialização (modo Headless / Linux)
- Sem Electron; entry point `./headless` resolve paths via `OD_DATA_DIR` ou `XDG_DATA_HOME`.
- Inicia daemon + web sidecars, expõe server IPC para ferramentas externas.
- `requireDesktopAuth: false` — o portão de desktop auth fica desabilitado.

### Atualização do app
- `update.*` callbacks em `runDesktopMain()` delegam para `@open-design/desktop` que executa o fluxo de auto-update.

## 4. Métricas e KPIs

| Métrica | Fonte | Meta |
|---------|-------|------|
| Tempo até daemon ready | `DAEMON_STATUS_TIMEOUT_MS` | < 35 s |
| Taxa de crash no boot | Sentry / logs | < 0,5 % |
| Cobertura de testes unitários | `vitest run` | ≥ 80 % funções exportadas |
| Build limpo sem erros TypeScript | `pnpm typecheck` | 0 erros |
| Heartbeat identity gravado | `desktop-root.json` `updatedAt` | Intervalo ≤ 5 s |

## 5. Riscos e Mitigações

| Risco | Probabilidade | Impacto | Mitigação |
|-------|-------------|---------|-----------|
| `setTypeOfService EINVAL` em macOS/VPN | Alta | Médio | `isHarmlessSocketOptionError()` filtra e loga sem crash; protocolo `od://` retorna 502 ao renderer. |
| Migração legada de dados excedendo timeout | Baixa | Alto | `DAEMON_MIGRATION_STATUS_TIMEOUT_MS` = 30 min quando `OD_LEGACY_DATA_DIR` estiver set. |
| Paths não-namespace colisão entre canais | Baixa | Alto | `resolvePackagedNamespacePaths()` sempre aninha sob `namespaceBaseRoot/namespace`. |
| Config inexistente no ambiente de dev | Média | Baixo | `readRawPackagedConfig()` retorna `{}` quando nenhum arquivo é encontrado — defaults seguros. |
| `POSTHOG_KEY` vazar em bundle | Baixa | Médio | É uma ingest key pública (`phc_*`) — write-only; padrão PostHog para apps empacotados. |

## 6. Roadmap de Capacidades

| Fase | Capacidade | Status |
|------|-----------|--------|
| Atual | Boot Electron com daemon + web sidecars | ✅ Entregue |
| Atual | Protocolo `od://` com proxy para web sidecar | ✅ Entregue |
| Atual | Modo headless Linux | ✅ Entregue |
| Atual | Identity file com heartbeat | ✅ Entregue |
| Próxima | Diagnóstico de erro de path com instruções de recovery | ✅ Entregue (`PackagedPathAccessError`) |
| Futura | Inicialização paralela de daemon e web para reduzir tempo de boot | 🔲 Backlog |
