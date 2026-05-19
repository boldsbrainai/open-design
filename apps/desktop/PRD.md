# PRD Técnico 360° — Open Design Desktop

## 1. Overview Técnico

`@open-design/desktop` é o **Electron main process** do Open Design. Não implementa UI própria: toda interface é entregue pelo runtime web (`apps/web`) carregado num `BrowserWindow`. O papel do desktop é:

1. **Descobrir** a URL do web sidecar via JSON IPC socket.  
2. **Carregar** essa URL no `BrowserWindow` com sandbox habilitado.  
3. **Expor** pontes privilegiadas para o renderer via `contextBridge` (`preload.cts`).  
4. **Proteger** operações sensíveis (importação de pasta, abertura de caminho no FS) no main process.  
5. **Gerenciar** o ciclo de atualização do app (check, download, install).

| Atributo              | Valor                                             |
|-----------------------|---------------------------------------------------|
| Pacote                | `@open-design/desktop` v`0.7.0`                   |
| Electron              | `41.3.0`                                          |
| TypeScript            | `6.0.3`                                           |
| Target ES             | `ES2024`                                          |
| Module system         | `NodeNext` (ESM + `.cts` para preload CommonJS)   |
| Entrypoint compilado  | `dist/main/index.js`                              |
| Export público        | `./main` → `dist/main/index.js` + `.d.ts`         |

---

## 2. Processo Principal (main process)

### 2.1 Estrutura de arquivos

```
src/main/
├── index.ts          — orquestrador: bootstrapping, sidecar IPC, updater, menu
├── runtime.ts        — BrowserWindow, handlers IPC, validações de path, HMAC
├── preload.cts       — contextBridge (CommonJS — exigido pelo Electron sandbox)
├── pdf-export.ts     — exportação HTML→PDF via printToPDF do Electron
├── updater.ts        — engine de auto-update (check, download, verify, install)
└── uncaught-exception.ts — filtro defensivo de erros não tratados

tests/main/
├── hide-window-exiting-fullscreen.test.ts
├── save-print-ready-document-as-pdf.test.ts
├── uncaught-exception.test.ts
└── updater.test.ts
```

### 2.2 Ciclo de vida do processo

```
runDesktopMain(runtime, options)
  1. attachDesktopProcessErrorFilter()         — filtro uncaughtException/unhandledRejection
  2. app.whenReady()                           — Electron ready
  3. registerDesktopAuthWithDaemon()           — handshake HMAC com daemon via sidecar IPC
  4. createDesktopRuntime(options)             — registra handlers IPC + cria BrowserWindow
  5. createDesktopUpdater(config, deps)        — configura updater engine
  6. installDesktopMenu(updater)               — menu nativo (macOS + Windows/Linux)
  7. scheduleStartupUpdateCheck(updater)       — setTimeout(checkForUpdates, 5000)
  8. bootstrapSidecarRuntime()                 — inicializa contexto sidecar
  9. createJsonIpcServer()                     — escuta socket IPC do desktop
```

### 2.3 Descoberta da URL web

```typescript
// createWebDiscovery (index.ts)
const webIpc = resolveAppIpcPath({
  app: APP_KEYS.WEB,                   // "web"
  contract: OPEN_DESIGN_SIDECAR_CONTRACT,
  namespace: runtime.namespace,
});
const web = await requestJsonIpc<WebStatusSnapshot>(webIpc, {
  type: SIDECAR_MESSAGES.STATUS
}, { timeoutMs: 600 });
return web?.url ?? null;
```

O desktop faz polling no socket IPC do sidecar web até obter a URL, exibindo uma tela de espera (`createPendingHtml`) enquanto isso.

### 2.4 Monitoramento do processo pai (tools-dev)

Quando `OD_TOOLS_DEV_PARENT_PID` está presente, um `setInterval(1000)` verifica se o PID pai ainda está vivo. Quando o pai termina, o desktop chama `options.beforeShutdown()` e encerra com `process.exit(0)`.

### 2.5 Filtro de exceções não tratadas

`attachDesktopProcessErrorFilter()` instala handlers para `uncaughtException` e `unhandledRejection`. Erros que correspondem a `setTypeOfService EINVAL` (undici socket em ambientes com VPN/macOS) são suprimidos com `console.warn`; todos os demais são re-thrown via `setImmediate` para preservar o comportamento padrão de crash do Electron.

---

## 3. Renderer Process

### 3.1 Configuração do BrowserWindow

```typescript
new BrowserWindow({
  width: 1280,
  height: 900,
  minWidth: 900,
  minHeight: 600,
  show: true,
  title: "Open Design",
  // macOS: titleBarStyle: "hiddenInset", trafficLightPosition: { x: 12, y: 10 }
  webPreferences: {
    contextIsolation: true,    // obrigatório para sandbox
    nodeIntegration: false,    // nunca expor Node ao renderer
    preload: preloadPath,      // preload.cjs
    sandbox: true,             // renderer isolado do sistema
  },
})
```

### 3.2 Política de navegação

| Tipo de URL                    | Comportamento                                 |
|--------------------------------|-----------------------------------------------|
| Mesma origin que a URL atual   | Permitido (navegação normal)                  |
| `http:` / `https:` diferente   | Abre no browser externo via `shell.openExternal` |
| `blob:`                        | Permitido (downloads em renderer)             |
| `od:` (packaged)               | Permitido como child BrowserWindow (PR #911)  |
| `about:blank`                  | Permitido (PDF export fallback via `window.open`) |
| Qualquer outra                 | Negado (`{ action: "deny" }`)                 |

### 3.3 CSS de integração macOS

Inserido via `webContents.insertCSS` após `did-finish-load`. Define `-webkit-app-region: drag` no header e `no-drag` em botões, popovers e modais, habilitando a barra de título "escondida" nativa do macOS.

### 3.4 Download hook (PPTX)

`attachDownloadSaveAsDialog` intercepta `will-download` para arquivos `.pptx` e chama `item.setSaveDialogOptions()` para exibir o painel "Save As" nativo antes do download iniciar.

---

## 4. APIs e Contratos

### 4.1 contextBridge — `window.electronAPI`

Exposto via `preload.cts`:

| Método              | IPC channel              | Descrição                                                                                     |
|---------------------|--------------------------|-----------------------------------------------------------------------------------------------|
| `openExternal(url)` | `shell:open-external`    | Abre URL no browser externo. Rejeita se não for `http:` / `https:`.                          |
| `pickAndImport(init?)` | `dialog:pick-and-import` | Abre picker de pasta, minta token HMAC e POST `/api/import/folder`. Retorna response do daemon ou erro estruturado. |
| `openPath(projectId)` | `shell:open-path`        | Abre diretório do projeto no file manager do SO. Renderer informa apenas o `projectId`; caminho real é resolvido pelo daemon. |

### 4.2 contextBridge — `window.__odDesktop`

| Método                           | IPC channel   | Descrição                                                    |
|----------------------------------|---------------|--------------------------------------------------------------|
| `printPdf(html, nonce?, opts?)`  | `od:print-pdf` | Renderiza HTML numa BrowserWindow oculta e salva como PDF.  |
| `isDesktop`                      | — (constante) | `true` — permite ao renderer detectar contexto desktop.     |

### 4.3 Exports públicos do pacote (`./main`)

Funções puras exportadas para testes no workspace `apps/packaged`:

| Símbolo                       | Arquivo         | Propósito                                                              |
|-------------------------------|-----------------|------------------------------------------------------------------------|
| `isAllowedChildWindowUrl`     | `runtime.ts`    | Política de URL para `setWindowOpenHandler`                            |
| `isHttpUrl`                   | `runtime.ts`    | Detecta `http:`/`https:` para `shell:open-external`                   |
| `resolveDesktopStatusUrl`     | `runtime.ts`    | Merge `currentUrl`/`pendingUrl` para sincronização de estado           |
| `validateExistingDirectory`   | `runtime.ts`    | Valida path: absoluto, existe, isDirectory, não é `.app`               |
| `fetchResolvedProjectDir`     | `runtime.ts`    | Resolve projectId → `resolvedDir` consultando daemon                  |
| `isOpenPathAllowedForProject` | `runtime.ts`    | Verifica se projeto veio do trusted picker flow                        |
| `signDesktopImportToken`      | `runtime.ts`    | HMAC-SHA256 puro para token de importação                              |
| `pickAndImportFolder`         | `runtime.ts`    | Lógica de pick+import+retry-on-503 sem Electron                        |
| `isHarmlessSocketOptionError` | `uncaught-exception.ts` | Detecta `setTypeOfService EINVAL`                             |

---

## 5. Protocolos e Integrações (IPC, sidecar protocol)

### 5.1 JSON IPC Socket (sidecar)

O desktop se comunica com daemon e web via sockets POSIX/Named Pipe do protocolo sidecar:

```
/tmp/open-design/ipc/<namespace>/web.sock    ← consulta de URL
/tmp/open-design/ipc/<namespace>/daemon.sock ← registro do desktop auth secret
```

Mensagens usadas pelo desktop:

| Mensagem                 | Destino | Conteúdo enviado                            | Resposta esperada                     |
|--------------------------|---------|---------------------------------------------|---------------------------------------|
| `STATUS`                 | web     | `{ type: "STATUS" }`                        | `WebStatusSnapshot` com campo `url`   |
| `REGISTER_DESKTOP_AUTH`  | daemon  | `{ type, input: { secret: base64string } }` | `{ accepted: true }`                  |

### 5.2 HTTP API — daemon (via web sidecar proxy `/api/*`)

O main process faz `fetch` diretamente ao daemon (nunca via renderer):

| Endpoint                          | Método | Usado em                     | Proteção                          |
|-----------------------------------|--------|------------------------------|-----------------------------------|
| `/api/import/folder`              | POST   | `pickAndImportFolder`        | Header `X-OD-Desktop-Import-Token` (HMAC-SHA256, TTL 60 s) |
| `/api/projects/:id`               | GET    | `fetchResolvedProjectDir`    | Validação de `projectId` via regex `[A-Za-z0-9._-]{1,128}` |

### 5.3 Sidecar stamp (processo desktop)

Ao ser lançado pelo sidecar, o processo desktop recebe exatamente 5 flags:

```
--od-stamp-app=desktop
--od-stamp-mode=dev|runtime
--od-stamp-namespace=<namespace>
--od-stamp-ipc=<ipc-path>
--od-stamp-source=tools-dev|tools-pack|packaged
```

### 5.4 Protocolo de atualização

```
GET {OD_UPDATE_METADATA_URL}/<channel>/<platform>/<arch>/latest.json
  → { version, artifacts: [{ filename, url, checksums: { sha256 } }] }

GET {artifact.url}
  → binário do instalador

SHA-256 verification → state: downloaded → shell.openPath(installerPath)
```

Sentinel de propriedade do diretório de download: `.open-design-updater-root.json`.  
Estado persistido em: `state.json` (dentro de `OD_UPDATE_DOWNLOAD_ROOT`).

---

## 6. Segurança e Autenticação

### 6.1 Modelo de segurança do renderer

- `sandbox: true` — renderer isolado do processo Node/OS.
- `contextIsolation: true` — globals do renderer e do preload são separados.
- `nodeIntegration: false` — renderer não tem acesso a `require`, `process`, etc.
- `contextBridge` é a única superfície de comunicação renderer→main.

### 6.2 HMAC de importação de pasta (PR #974)

```
desktopAuthSecret = randomBytes(32)          ← gerado a cada inicialização
token = HMAC-SHA256(secret, baseDir + "\n" + nonce + "\n" + exp)
Header: X-OD-Desktop-Import-Token: <base64url-sig>~<nonce>~<exp>
TTL: 60 s
Separador de campos: ~ (nunca aparece em base64url nem ISO 8601)
```

O daemon exige o token em qualquer `POST /api/import/folder` quando um secret de desktop está registrado. O token expira e contém nonce para prevenção de replay.

### 6.3 Validação de path para `shell.openPath`

1. Renderer envia apenas `projectId` (string alfanumérica `[A-Za-z0-9._-]{1,128}`).  
2. Main process consulta `/api/projects/:id` → obtém `resolvedDir`.  
3. Verifica: `hasBaseDir && !fromTrustedPicker` → rejeita.  
4. `validateExistingDirectory`: absoluto, existe, isDirectory, sem `.app`.  
5. Apenas então chama `shell.openPath(validatedPath)`.

### 6.4 Filtragem de URLs

- `shell:open-external` só aceita `http:`/`https:`.  
- `setWindowOpenHandler` nega tudo que não seja `blob:`, `od:`, `about:blank` ou mesma origin HTTP.  
- `will-navigate` redireciona cross-origin HTTP para o browser externo.

---

## 7. Performance e Escalabilidade

| Aspecto                       | Detalhe                                                                                    |
|-------------------------------|--------------------------------------------------------------------------------------------|
| Polling de URL do web sidecar | Timeout por tentativa: 600 ms via `requestJsonIpc`                                         |
| Auto-check de update          | Disparado 5 s após `app.whenReady` com `timer.unref()` (não bloqueia quit)                 |
| Retry de auth HMAC            | Delays: `[120, 240, 480, 960, 1500]` ms; timeout por tentativa: 800 ms                    |
| Monitor de PID pai            | `setInterval(1000).unref()` — não bloqueia event loop                                     |
| BrowserWindow min size        | 900×600 — previne layout quebrado no split chat+design+preview                             |
| PDF export deck               | BrowserWindow oculto 1920×1080; `printToPDF` com `pageSize: { width: 13.33, height: 7.5 }`|
| Handlers IPC                  | `ipcMain.removeHandler` antes de cada `handle` — suporte a hot-reload sem duplicatas      |

---

## 8. Testes e Qualidade

### 8.1 Suite de testes

| Arquivo de teste                                 | O que cobre                                                              |
|--------------------------------------------------|--------------------------------------------------------------------------|
| `tests/main/uncaught-exception.test.ts`          | `isHarmlessSocketOptionError`, handlers de exceção/rejeição              |
| `tests/main/hide-window-exiting-fullscreen.test.ts` | `hideWindowExitingFullscreen` — casos fullscreen, simpleFullscreen, entering |
| `tests/main/save-print-ready-document-as-pdf.test.ts` | `savePrintReadyDocumentAsPdf` — PDF target, opções deck                |
| `tests/main/updater.test.ts`                     | `DesktopUpdater` — check, download, verify, install, channel/mode parsing |

Testes adicionais em `apps/packaged/tests/`:
- `desktop-url-allowlist.test.ts` → `isAllowedChildWindowUrl`, `isHttpUrl`, `resolveDesktopStatusUrl`
- `desktop-import-token.test.ts` → `signDesktopImportToken`, shape do token, TTL
- `pick-and-import-folder.test.ts` → `pickAndImportFolder`, retry-on-503

### 8.2 Comandos de qualidade

```bash
pnpm --filter @open-design/desktop typecheck
pnpm --filter @open-design/desktop test
pnpm --filter @open-design/desktop build
```

### 8.3 Framework

- **Runner**: Vitest `^2.1.8`, environment `node`, timeout `10_000 ms`.  
- **Cobertura de testes E2E**: `e2e/` com Playwright (não cobre UI diretamente do pacote desktop; testa via `tools-dev` + browser).

### 8.4 Invariantes documentados por testes

- Token HMAC tem exatamente 3 segmentos separados por `~`.
- `DESKTOP_IMPORT_TOKEN_FIELD_SEP` é `~` (nunca `.`).
- `pickAndImportFolder` faz exatamente **1** retry em `503 DESKTOP_AUTH_PENDING`.
- `hideWindowExitingFullscreen` não chama `hide()` enquanto ainda está `enteringFullscreen`.
