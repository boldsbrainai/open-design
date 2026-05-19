# PRD Técnico 360° — Packaged

## 1. Overview Técnico

| Campo | Valor |
|-------|-------|
| Pacote | `@open-design/packaged` v0.8.0 |
| Runtime | Node `~24`, Electron 41.3.0 |
| Linguagem | TypeScript 6.0.3 (ESM) |
| Bundler | esbuild 0.27.7 — target `node24`, `format: esm`, `packages: external` |
| Exports | `./` → `dist/index.mjs` (Electron entry); `./headless` → `dist/headless.mjs` |
| Tipo de declarações | `dist/index.d.ts` via `tsc --emitDeclarationOnly` |

Este pacote é um **thin orchestrator**: não contém lógica de produto. Toda a UI vive em `@open-design/desktop`, todo o backend em `@open-design/daemon`, e todo o frontend em `@open-design/web`. O papel do `packaged` é inicializar o ambiente Electron correto, spawnar os sidecars, e entregar o controle.

---

## 2. Backend / Servidor

Não há servidor HTTP próprio. O `packaged` spawna dois servidores como processos filhos:

### Daemon sidecar
- Entry: `config.daemonSidecarEntry` (path absoluto baked em `open-design-config.json`).
- Porta: dinâmica (negociada via IPC); nunca embarcada em paths.
- Timeout de ready: 35 s (`DAEMON_STATUS_TIMEOUT_MS`); 30 min se `OD_LEGACY_DATA_DIR` estiver definido.
- Env passada: `PACKAGED_CHILD_ENV_ALLOWLIST` + `*_API_KEY`/`*_TOKEN` opcionalmente.

### Web sidecar
- Entry: `config.webSidecarEntry`.
- Modo: `"server"` (SSR Next.js) ou `"standalone"` (Next standalone output).
- URL reportada via IPC após boot.

### IPC
- POSIX sockets em `/tmp/open-design/ipc/<namespace>/<app>.sock`.
- `bootstrapSidecarRuntime()` de `@open-design/sidecar` gerencia a lógica de IPC.
- `requestJsonIpc()` é usado para consultar status dos sidecars.

---

## 3. Frontend / Cliente

Não há frontend próprio. O `packaged`:
1. Registra o protocolo `od://` no Electron antes de qualquer renderer ser criado.
2. Delega a criação de janelas para `@open-design/desktop/main` via `runDesktopMain()`.
3. Fornece `discoverWebUrl()` → retorna `od://app/` (entrada do protocolo).
4. Fornece `discoverDaemonUrl()` → retorna a URL HTTP real do daemon sidecar.

### Protocolo `od://`
- `OD_SCHEME = "od"`, `OD_ENTRY_URL = "od://app/"`.
- Privilegiado: `corsEnabled`, `secure`, `standard`, `stream`, `supportFetchAPI`.
- Cada request de renderer é proxiado para `webRuntimeUrl` (URL real do web sidecar) via `handleOdRequest()`.
- Em caso de falha de socket (`EINVAL` / VPN), retorna 502 JSON em vez de crashar.

---

## 4. APIs e Contratos

### Config de pacote (`open-design-config.json`)
Arquivo baked por `tools-pack` em `process.resourcesPath`. Campos:

| Campo | Tipo | Descrição |
|-------|------|-----------|
| `appVersion` | `string \| null` | Versão do app exibida na UI |
| `daemonCliEntryRelative` | `string` | Path relativo ao CLI do daemon |
| `daemonSidecarEntryRelative` | `string` | Path relativo ao sidecar do daemon |
| `webSidecarEntryRelative` | `string` | Path relativo ao sidecar web |
| `namespace` | `string` | Namespace padrão do runtime |
| `namespaceBaseRoot` | `string` | Raiz base dos namespaces |
| `resourceRoot` | `string` | Raiz de recursos read-only (skills, design-systems, frames) |
| `telemetryRelayUrl` | `string \| null` | URL pública do telemetry worker |
| `posthogKey` | `string \| null` | Chave pública PostHog (ingest-only) |
| `posthogHost` | `string \| null` | Host PostHog (opcional) |
| `nodeCommandRelative` | `string` | Path relativo ao binário Node |
| `webOutputMode` | `"server" \| "standalone"` | Modo de output do Next.js |
| `webStandaloneRoot` | `string \| null` | Raiz do standalone output |

### Identity File (`desktop-root.json`)
Gravado em `<namespaceRoot>/runtime/desktop-root.json` com heartbeat a cada 5 s:

| Campo | Tipo |
|-------|------|
| `appPath` | `string` — caminho do `.app` no macOS |
| `executablePath` | `string` |
| `logPath` | `string` |
| `namespaceRoot` | `string` |
| `pid` / `ppid` | `number` |
| `stamp` | `SidecarStamp` (5 campos: `app`, `ipc`, `mode`, `namespace`, `source`) |
| `startedAt` / `updatedAt` | `string` ISO 8601 |
| `version` | `1` |

### SidecarStamp (5 campos obrigatórios)
```ts
{ app, ipc, mode, namespace, source }
```

---

## 5. Protocolos e Integrações

### `@open-design/sidecar-proto`
- `APP_KEYS`, `SIDECAR_MODES`, `SIDECAR_SOURCES`, `OPEN_DESIGN_SIDECAR_CONTRACT`
- `normalizeNamespace()`, `SIDECAR_DEFAULTS`

### `@open-design/sidecar`
- `bootstrapSidecarRuntime()` — contexto IPC do runtime.
- `createSidecarLaunchEnv()` — env vars dos filhos.
- `resolveAppIpcPath()` — path do socket IPC.
- `requestJsonIpc()` — consultas IPC.
- `writeJsonFile()` / `removeFile()` — I/O para identity files.

### `@open-design/platform`
- `readProcessStamp()` — lê stamp dos argv.
- `createProcessStampArgs()` / `stopProcesses()` / `waitForProcessExit()`.
- `wellKnownUserToolchainBins()` — binários Node do toolchain do usuário.

### `@open-design/desktop/main`
- `runDesktopMain(runtime, options)` — janela Electron, auto-update, shortcuts.

---

## 6. Segurança e Autenticação

| Aspecto | Implementação |
|---------|--------------|
| Segredos Langfuse | Nunca embarcados — apenas `telemetryRelayUrl` (URL pública). |
| Env dos processos filhos | Filtrada por `PACKAGED_CHILD_ENV_ALLOWLIST`; segredos de providers repassados apenas quando `includeProviderSecrets = true`. |
| `requireDesktopAuth` | Sempre `true` na entrada Electron; `false` no modo headless. |
| Protocolo `od://` | Privilegiado e isolado do exterior; proxia apenas para `127.0.0.1` (web sidecar local). |
| Erros de socket | `isHarmlessSocketOptionError()` — apenas `setTypeOfService EINVAL` é silenciado; outros erros propagam. |
| Identity heartbeat | Sobrescreve a cada 5 s — detecta processos zumbis ao comparar `updatedAt`. |

---

## 7. Performance e Escalabilidade

| Dimensão | Detalhe |
|---------|---------|
| Tempo de boot | Daemon ready em < 35 s; web sidecar em < 30 s normalmente. |
| Migração legada | Budget estendido para 30 min quando `OD_LEGACY_DATA_DIR` ativo. |
| Bundle size | esbuild com `packages: external` — apenas glue code no bundle; deps resolvidas em runtime. |
| Dois entrypoints independentes | `index.mjs` (Electron) e `headless.mjs` — sem código Electron no bundle headless. |
| Namespaces isolados | Múltiplos canais (stable/beta/preview) podem rodar simultaneamente sem colisão de paths. |

---

## 8. Testes e Qualidade

| Ferramenta | Comando | Escopo |
|-----------|---------|--------|
| Vitest | `pnpm test` | Testes unitários em `tests/` |
| TypeScript | `pnpm typecheck` | Valida `tsconfig.json` + `tsconfig.tests.json` |
| Build check | `pnpm build` | esbuild + `tsc --emitDeclarationOnly` |

### Testes notáveis (baseados nos módulos exportados)
- `handleOdRequest()` — testável sem Electron via `fetchImpl` stub.
- `isHarmlessSocketOptionError()` — testa shape exata do erro `setTypeOfService EINVAL`.
- `resolvePackagedNamespacePaths()` — valida todos os subpaths gerados.
- `validateIngestionBody()` (telemetry-worker) — série de casos de borda para batch.

### Pré-requisito de typecheck
`pnpm typecheck` depende de `pnpm --filter @open-design/desktop build` — o desktop deve ser buildado antes para os tipos estarem disponíveis.
