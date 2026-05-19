# BizOps Blueprint — Open Design Desktop

## 1. Visão Geral

`@open-design/desktop` é o shell Electron que empacota o runtime web do Open Design como uma aplicação nativa de desktop. Ele não possui UI própria: todo o conteúdo visual é renderizado pelo `apps/web` carregado em um `BrowserWindow`. O papel operacional do desktop é descobrir a URL do web runtime via IPC de sidecar, gerenciar o ciclo de vida da janela, fornecer pontes privilegiadas de main process para o renderer, e orquestrar o sistema de atualização da aplicação.

| Atributo         | Valor                                                          |
|------------------|----------------------------------------------------------------|
| Pacote npm       | `@open-design/desktop`                                         |
| Versão atual     | `0.7.0`                                                        |
| Runtime Electron | `41.3.0`                                                       |
| Node alvo        | `~24`                                                          |
| Plataformas      | macOS, Windows, Linux                                          |
| Entry point      | `dist/main/index.js` (compilado de `src/main/index.ts`)        |
| Canal de release | `stable`, `beta` (via `OD_UPDATE_CHANNEL`)                     |
| Release origin   | `https://releases.open-design.ai`                              |

---

## 2. Objetivos de Negócio (OKRs)

### Objetivo 1 — Experiência nativa de qualidade
- **KR 1.1** Janela inicia e carrega a UI web em < 3 s em hardware de referência.
- **KR 1.2** Zero diálogos de "JavaScript error in main process" causados por erros de socket conhecidos (filtro `setTypeOfService EINVAL`).
- **KR 1.3** Exportação para PDF disponível para decks (1920×1080) e documentos (1440×900) com diálogo "Save As" nativo.

### Objetivo 2 — Distribuição contínua e atualizações confiáveis
- **KR 2.1** Auto-check de atualização 5 s após startup quando `OD_UPDATE_AUTO_CHECK=1`.
- **KR 2.2** Download automático de artefatos com verificação de checksum SHA-256.
- **KR 2.3** Suporte a dois canais independentes (`stable` / `beta`) sem risco de cross-channel.

### Objetivo 3 — Segurança por padrão
- **KR 3.1** Renderer rodando em `sandbox: true`, `contextIsolation: true`, `nodeIntegration: false`.
- **KR 3.2** Importação de pasta via HMAC-token obrigatório (PR #974) — zero bypasses diretos de renderer para filesystem.
- **KR 3.3** `shell.openPath` restrito a `projectId` validado pelo daemon — renderer jamais nomeia caminho diretamente.

### Objetivo 4 — Integração com o ecossistema tools-dev
- **KR 4.1** Desktop encerra-se automaticamente quando o processo `tools-dev` pai terminar (monitor de PID).
- **KR 4.2** Namespace de sidecar isolado por processo para execuções paralelas de development.

---

## 3. Fluxos Operacionais

### 3.1 Inicialização completa (modo dev — tools-dev)

```
tools-dev
  └─ spawna daemon  (namespace=<ns>)
  └─ spawna web     (namespace=<ns>)
  └─ spawna desktop (namespace=<ns>, OD_TOOLS_DEV_PARENT_PID=<pid>)
        │
        ├─ attachDesktopProcessErrorFilter()    ← filtro uncaught-exception
        ├─ app.whenReady()
        ├─ registerDesktopAuthWithDaemon()      ← HMAC secret → daemon via IPC
        │      ├─ retry delays: [120, 240, 480, 960, 1500] ms
        │      └─ timeout por tentativa: 800 ms
        ├─ createDesktopRuntime()               ← registra handlers IPC
        ├─ createDesktopUpdater()               ← configura update engine
        ├─ installDesktopMenu()                 ← menu nativo com opções de update
        ├─ scheduleStartupUpdateCheck()         ← setTimeout(5000)
        └─ BrowserWindow(1280×900)
               └─ loadURL(pendingHtml)          ← tela "Waiting for web runtime URL…"
                     ↓  poll via createWebDiscovery
               └─ loadURL(webUrl)              ← URL descoberta via IPC sidecar
```

### 3.2 Inicialização completa (modo packaged)

```
apps/packaged
  └─ inicia sidecars daemon + web
  └─ chama runDesktopMain(runtime, {
       discoverWebUrl:   → od://app/       (Electron protocol)
       discoverDaemonUrl:→ http://127.0.0.1:<port>   (para Node fetch)
       update: { currentVersion, downloadRoot }
     })
```

### 3.3 Fluxo de importação de pasta (PR #974)

```
Renderer
  └─ window.electronAPI.pickAndImport(init?)
        │  IPC: dialog:pick-and-import
        ↓
Main Process
  ├─ dialog.showOpenDialog({ properties: ['openDirectory'] })
  ├─ baseDir = filePaths[0].trim()
  ├─ mintImportToken(desktopAuthSecret, baseDir)   ← HMAC-SHA256, TTL 60s
  └─ POST /api/import/folder
         Headers: X-OD-Desktop-Import-Token: <token>
         Body: { baseDir, name?, skillId?, designSystemId? }
              ↓
         Daemon verifica token → importa → retorna { project, conversationId, entryFile }
              ↓
         Em caso de 503 DESKTOP_AUTH_PENDING → re-registra secret → retenta uma vez
```

### 3.4 Fluxo de atualização

```
scheduleStartupUpdateCheck (5s após ready)
  └─ updater.checkForUpdates()
        ├─ GET OD_UPDATE_METADATA_URL (default: https://releases.open-design.ai/...)
        ├─ verifica versão disponível vs OD_UPDATE_CURRENT_VERSION
        └─ se downloaded → showUpdateResultDialog()
              ├─ "Install Update" → updater.installUpdate() → shell.openPath(installerPath)
              └─ "Later"         → descarta
```

### 3.5 Filtro de exceções não tratadas

```
process.on('uncaughtException', handler)
process.on('unhandledRejection', handler)
  ├─ isHarmlessSocketOptionError?
  │     └─ logger.warn + return silently
  └─ logger.error + removeListener + setImmediate(() => { throw error })
```

---

## 4. Métricas e KPIs

| Métrica                                  | Alvo              | Fonte                                |
|------------------------------------------|-------------------|--------------------------------------|
| Tempo até primeira renderização (web URL) | < 3 s             | logs do sidecar IPC poll             |
| Taxa de atualização bem-sucedida         | > 95% das sessões | `updater.snapshot().state`           |
| Diálogos de erro suprimidos              | 0                 | filtro `isHarmlessSocketOptionError` |
| Falhas na importação de pasta            | < 1%              | body `error.code` retornado ao renderer |
| Tentativas de re-registro HMAC           | 0 em condições normais | log `DESKTOP_AUTH_PENDING`        |
| Reinicializações forçadas por OOM        | 0                 | `uncaughtException` rate             |

---

## 5. Riscos e Mitigações

| Risco                                                         | Impacto | Probabilidade | Mitigação                                                         |
|---------------------------------------------------------------|---------|---------------|-------------------------------------------------------------------|
| Daemon não responde no handshake HMAC inicial                 | Médio   | Baixa         | Retry com backoff exponencial; fallback lazy na primeira importação |
| `setTypeOfService EINVAL` (undici + VPN/macOS)                | Alto    | Média         | `attachDesktopProcessErrorFilter` suprime a exceção harmless      |
| Renderer comprometido injetando `baseDir` arbitrário          | Alto    | Muito Baixa   | IPC `dialog:pick-and-import` une picker + import atomicamente     |
| Update baixado com checksum inválido                          | Alto    | Muito Baixa   | Verificação SHA-256 antes de marcar `state: downloaded`           |
| Janela minimizada no Windows (focus-stealing prevention)      | Baixo   | Baixa         | `ensureWindowVisible` restaura e foca a janela                    |
| Fullscreen macOS — Space orphan ao fechar                     | Médio   | Média         | `hideWindowExitingFullscreen` aguarda `leave-full-screen`         |
| Cross-channel update (stable recebendo artefato beta)         | Alto    | Muito Baixa   | `normalizeChannel` valida e rejeita canal desconhecido            |

---

## 6. Roadmap de Capacidades

| Capacidade                                          | Status         | Referência     |
|-----------------------------------------------------|----------------|----------------|
| Shell Electron com BrowserWindow                    | Estável        | `runtime.ts`   |
| Descoberta de URL via sidecar IPC                   | Estável        | `index.ts`     |
| HMAC-gated folder import (PR #974)                  | Estável        | `runtime.ts`   |
| Re-registro lazy em `DESKTOP_AUTH_PENDING`          | Estável (R5)   | `runtime.ts`   |
| PDF export (deck 1920×1080 + documento 1440×900)    | Estável        | `pdf-export.ts`|
| Save-as dialog para `.pptx` (download hook)         | Estável        | `runtime.ts`   |
| Auto-updater (package-launcher + js-incremental)   | Estável        | `updater.ts`   |
| Filtro `setTypeOfService EINVAL` (PR #895, #647)    | Estável        | `uncaught-exception.ts` |
| `hideWindowExitingFullscreen` (macOS Space safe)    | Estável        | `runtime.ts`   |
| Promoção de `uncaught-exception.ts` para `@open-design/platform` | Pendente | comentário no código |
