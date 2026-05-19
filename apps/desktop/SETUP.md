# Setup Guide — Open Design Desktop

## Pré-requisitos

| Dependência     | Versão mínima | Observação                                              |
|-----------------|---------------|---------------------------------------------------------|
| Node.js         | `~24`         | Use Corepack para garantir a versão correta             |
| pnpm            | `10.33.2`     | Gerenciado via Corepack (`corepack enable`)             |
| Electron        | `41.3.0`      | Instalado como devDependency — não instalar globalmente |
| TypeScript      | `6.0.3`       | Compilado pelo build do pacote, não instalado globalmente |
| Sistema operacional | macOS, Windows, Linux | Electron suporta as 3 plataformas        |

> **Nota:** O `apps/desktop` nunca é executado diretamente. Ele é iniciado pelo `apps/packaged` (modo release) ou pelo `tools-dev` (modo desenvolvimento local). Siga as seções abaixo conforme o modo desejado.

---

## Variáveis de Ambiente

### Variáveis do sidecar (injetadas pelo orquestrador)

Estas variáveis são injetadas automaticamente pelo `tools-dev` ou `apps/packaged`. Normalmente **não precisam ser definidas manualmente**.

| Variável                 | Obrigatória | Exemplo                                  | Descrição                                                                 |
|--------------------------|-------------|------------------------------------------|---------------------------------------------------------------------------|
| `OD_SIDECAR_BASE`        | Sim         | `/home/user/.tmp/tools-dev/default`      | Diretório base dos arquivos de runtime do sidecar                         |
| `OD_SIDECAR_NAMESPACE`   | Sim         | `default`                                | Namespace de isolamento — permite múltiplas instâncias simultâneas         |
| `OD_SIDECAR_SOURCE`      | Sim         | `tools-dev` \| `tools-pack` \| `packaged` | Fonte de lançamento do processo                                           |
| `OD_SIDECAR_IPC_BASE`    | Não         | `/tmp/open-design/ipc`                   | Diretório base dos sockets IPC POSIX                                      |
| `OD_SIDECAR_IPC_PATH`    | Não         | `/tmp/open-design/ipc/default/desktop.sock` | Caminho direto do socket IPC deste processo                            |
| `OD_TOOLS_DEV_PARENT_PID`| Não         | `12345`                                  | PID do processo `tools-dev` pai. Quando definido, desktop encerra automaticamente quando o pai morre. |

### Variáveis do sistema de atualização

Controlam o comportamento do `DesktopUpdater`. Úteis para ambientes de CI, beta testing e builds customizados.

| Variável                   | Tipo      | Padrão                                  | Descrição                                                                 |
|----------------------------|-----------|-----------------------------------------|---------------------------------------------------------------------------|
| `OD_UPDATE_ENABLED`        | `boolean` | `1` (packaged), `0` (dev)               | Habilita/desabilita o sistema de update. Aceita `1/0/true/false/yes/no`.  |
| `OD_UPDATE_CHANNEL`        | `string`  | `stable`                                | Canal de atualização. Valores: `stable`, `beta`.                          |
| `OD_UPDATE_MODE`           | `string`  | `package-launcher`                      | Modo de instalação. Valores: `package-launcher`, `js-incremental`.        |
| `OD_UPDATE_CURRENT_VERSION`| `string`  | versão do `package.json` (`0.7.0`)      | Versão atual usada para comparação com o servidor de updates.              |
| `OD_UPDATE_DOWNLOAD_ROOT`  | `string`  | `<runtimeBase>/updater`                 | Diretório local onde os artefatos baixados ficam armazenados.             |
| `OD_UPDATE_METADATA_URL`   | `string`  | `https://releases.open-design.ai/...`  | URL para buscar metadados de release (`latest.json`).                     |
| `OD_UPDATE_AUTO_CHECK`     | `boolean` | `1`                                     | Habilita verificação automática 5 s após a inicialização.                 |
| `OD_UPDATE_AUTO_DOWNLOAD`  | `boolean` | `0`                                     | Habilita download automático quando nova versão disponível.               |
| `OD_UPDATE_AUTO_OPEN`      | `boolean` | `0`                                     | Abre o instalador automaticamente após download.                          |
| `OD_UPDATE_ARCH`           | `string`  | `process.arch` (`x64`, `arm64`)         | Arquitetura do artefato a baixar. Inferida automaticamente em builds normais. |
| `OD_UPDATE_PLATFORM`       | `string`  | `process.platform` (`darwin`, `win32`, `linux`) | Plataforma do artefato. Inferida automaticamente.               |
| `OD_UPDATE_OPEN_DRY_RUN`   | `boolean` | `0`                                     | Simula a abertura do instalador sem realmente abri-lo (útil em CI).       |

### Variáveis herdadas (usadas internamente pelo daemon/web, não pelo desktop)

| Variável      | Descrição                                                             |
|---------------|-----------------------------------------------------------------------|
| `OD_PORT`     | Porta HTTP do daemon. O desktop não a lê diretamente — obtém via IPC do sidecar web. |
| `OD_WEB_PORT` | Porta do servidor web Next.js. Idem — obtida via IPC.                 |

---

## Instalação

### 1. Clonar o repositório e instalar dependências

```bash
git clone https://github.com/nexu-io/open-design.git
cd open-design

# Ativar Corepack para usar a versão correta do pnpm
corepack enable

# Instalar todas as dependências do monorepo
pnpm install
```

### 2. Compilar o pacote desktop

```bash
pnpm --filter @open-design/desktop build
```

Isso executa `tsc -p tsconfig.json` e gera `apps/desktop/dist/main/index.js` (e `.d.ts`).

> **Atenção:** O `preload.cts` é compilado para `preload.cjs` (CommonJS) pelo TypeScript. Isso é obrigatório — o Electron requer que preload scripts sejam CommonJS mesmo em projetos ESM.

---

## Configuração Inicial

O `apps/desktop` não possui arquivo de configuração próprio. Toda configuração é recebida via:

1. **Variáveis de ambiente** injetadas pelo orquestrador (`tools-dev` ou `apps/packaged`).
2. **Sidecar stamp flags** (`--od-stamp-app`, `--od-stamp-namespace`, etc.) lidos pelo `@open-design/platform`.
3. **Opções de runtime** (`DesktopMainOptions`) passadas pelo entry point do `apps/packaged`.

Para desenvolvimento local, **nenhuma configuração adicional é necessária** — o `tools-dev` injeta tudo automaticamente.

---

## Inicialização

### Modo desenvolvimento (recomendado)

Use exclusivamente o `tools-dev` como ponto de entrada. Nunca inicie o desktop isoladamente em desenvolvimento.

```bash
# Inicia daemon + web + desktop com namespace padrão
pnpm tools-dev

# Inicia com portas customizadas
pnpm tools-dev run web --daemon-port 17456 --web-port 17573

# Verifica status dos processos
pnpm tools-dev status --json

# Inspeciona o desktop
pnpm tools-dev inspect desktop status --json

# Captura screenshot da janela desktop
pnpm tools-dev inspect desktop screenshot --path /tmp/open-design.png
```

### Modo packaged (release)

O desktop é iniciado pelo `apps/packaged` via `tools-pack`:

```bash
# macOS
pnpm tools-pack mac build --to all
pnpm tools-pack mac install

# Windows
pnpm tools-pack win build --to nsis
pnpm tools-pack win install

# Linux
pnpm tools-pack linux build --to appimage
pnpm tools-pack linux install
```

---

## Comandos Disponíveis

### Comandos do pacote `@open-design/desktop`

```bash
# Compilar TypeScript → dist/
pnpm --filter @open-design/desktop build

# Verificar tipos sem emitir arquivos
pnpm --filter @open-design/desktop typecheck

# Executar suite de testes (Vitest)
pnpm --filter @open-design/desktop test
```

### Comandos do monorepo relevantes para o desktop

```bash
# Typecheck global
pnpm typecheck

# Guard (valida convenções do monorepo)
pnpm guard

# Iniciar ambiente de desenvolvimento completo
pnpm tools-dev

# Parar todos os processos
pnpm tools-dev stop

# Logs em tempo real
pnpm tools-dev logs --json

# Logs de namespace específico
pnpm tools-dev logs --namespace <name> --json
```

---

## Verificação

Após `pnpm --filter @open-design/desktop build`, verifique:

```bash
# 1. Confirmar que o dist foi gerado
ls apps/desktop/dist/main/
# Esperado: index.js, index.d.ts, index.d.ts.map, runtime.js, updater.js,
#           pdf-export.js, uncaught-exception.js, preload.cjs

# 2. Verificar que não há erros de tipo
pnpm --filter @open-design/desktop typecheck
# Esperado: saída vazia (sem erros)

# 3. Executar os testes
pnpm --filter @open-design/desktop test
# Esperado: todos os testes passando em tests/main/

# 4. Iniciar o ambiente completo e verificar o desktop
pnpm tools-dev
pnpm tools-dev inspect desktop status --json
# Esperado: { "running": true, "url": "http://127.0.0.1:<port>" }
```

---

## Troubleshooting

### Desktop não abre / janela não aparece (Windows)

**Causa:** Windows focus-stealing prevention pode deixar a janela minimizada ou oculta.  
**Solução:** O desktop chama `ensureWindowVisible()` automaticamente. Se ainda assim não aparecer, verifique `pnpm tools-dev inspect desktop status --json` para confirmar que o processo está rodando.

### Diálogo "JavaScript error in main process" ao conectar em VPN (macOS)

**Causa:** undici lança `setTypeOfService EINVAL` em kernels que recusam definir o byte IP_TOS.  
**Solução:** Já tratado pelo `attachDesktopProcessErrorFilter`. Se o diálogo ainda aparecer, verifique se o build é recente (a partir do PR #895 / #647).

### Pasta não importa — erro `desktop auth secret not registered`

**Causa:** O handshake HMAC inicial com o daemon não completou.  
**Solução:** Na primeira tentativa de importação, o desktop tenta o registro lazy automaticamente (`DESKTOP_AUTH_PENDING` → re-register → retry). Se persistir, reinicie o `tools-dev` para garantir que daemon e desktop se inicializem na mesma janela de tempo.

### PDF não é gerado

**Causa:** O `od:print-pdf` IPC lança erro se o HTML for inválido ou se o `printToPDF` falhar.  
**Solução:** Verifique no DevTools do renderer (View → Toggle DevTools) se `window.__odDesktop.printPdf` está disponível. Em `isDesktop` deve ser `true`.

### Update travado em `checking` ou `downloading`

**Causa:** `OD_UPDATE_METADATA_URL` inacessível ou artefato com checksum diferente.  
**Solução:**
1. Verifique conectividade com `https://releases.open-design.ai`.
2. Limpe o `OD_UPDATE_DOWNLOAD_ROOT` (padrão: `<runtimeBase>/updater`).
3. Defina `OD_UPDATE_ENABLED=0` para desabilitar temporariamente.

### Erro `unsupported desktop update channel` ou `unsupported desktop update mode`

**Causa:** `OD_UPDATE_CHANNEL` ou `OD_UPDATE_MODE` com valor inválido.  
**Valores aceitos:**
- `OD_UPDATE_CHANNEL`: `stable` ou `beta`
- `OD_UPDATE_MODE`: `package-launcher` ou `js-incremental`

### TypeScript não encontra tipos do Electron

**Causa:** `electron` está como `devDependency` — não é instalado em `node_modules` de outros pacotes.  
**Solução:** Execute `pnpm install` na raiz do monorepo para garantir que o workspace link está atualizado.
