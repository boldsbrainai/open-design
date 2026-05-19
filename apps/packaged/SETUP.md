# Setup Guide — Packaged

## Pré-requisitos

| Requisito | Versão | Observação |
|-----------|--------|-----------|
| Node.js | `~24` | Obrigatório — engine declarado no `package.json` |
| pnpm | `10.33.2` | Via Corepack (`corepack enable`) |
| Electron | `41.3.0` | Instalado como devDependency — necessário para build e testes |
| `@open-design/desktop` | workspace | Deve ser buildado antes do typecheck deste pacote |

> **Atenção:** `apps/packaged` é um pacote buildado — seu output em `dist/` deve existir antes de ser usado pelo desktop. Não execute diretamente com `node`; use `pnpm tools-pack` ou `pnpm tools-dev` para o ciclo de vida completo.

## Variáveis de Ambiente

### Variáveis lidas em runtime (pelo processo packaged)

| Variável | Tipo | Padrão | Obrigatório | Descrição |
|----------|------|--------|-------------|-----------|
| `OD_PACKAGED_CONFIG_PATH` | `string` | — | Não | Path explícito para o arquivo `open-design-config.json`. Sobrescreve a busca padrão em `process.resourcesPath`. |
| `OD_PACKAGED_NAMESPACE` | `string` | `SIDECAR_DEFAULTS.namespace` | Não | Namespace de runtime. Isola paths de dados/logs/cache entre canais de release. |
| `OD_PACKAGED_ALLOW_WEB_OUTPUT_MODE_OVERRIDE` | `string` | — | Não | Permite sobrescrever o `webOutputMode` via `OD_WEB_OUTPUT_MODE`. |
| `OD_WEB_STANDALONE_ROOT` | `string` | — | Não | Raiz do standalone output do Next.js (modo `standalone`). |
| `OD_WEB_OUTPUT_MODE` | `"server" \| "standalone"` | `"server"` | Não | Modo de output do web sidecar. Requer `OD_PACKAGED_ALLOW_WEB_OUTPUT_MODE_OVERRIDE`. |
| `OD_DATA_DIR` | `string` | — | Não | Realoca **todo** o runtime data do daemon para este diretório. Aceita `~/` e paths relativos ao `projectRoot`. |
| `OD_RESOURCE_ROOT` | `string` | Relativo ao `node_modules` | Não | Raiz de recursos read-only do daemon (skills, design-systems, frames). No modo headless, default é `<__dirname>/../../../open-design`. |
| `OD_LEGACY_DATA_DIR` | `string` | — | Não | Quando definido, o budget de timeout do daemon é estendido para 30 min para suportar migração de dados legados. |
| `OPEN_DESIGN_TELEMETRY_RELAY_URL` | `string` | — | Não | URL pública do telemetry worker relay. Repassado ao daemon sidecar. |
| `POSTHOG_KEY` | `string` | — | Não | Chave pública PostHog (`phc_*`). Baked em `open-design-config.json` pelo `tools-pack`. No modo headless, lido de env. |
| `POSTHOG_HOST` | `string` | — | Não | Host PostHog customizado. |
| `XDG_DATA_HOME` | `string` | `~/.local/share` | Não | **Modo headless Linux apenas.** Base para `namespaceBaseRoot` quando `OD_DATA_DIR` não está definido. |
| `OD_DESKTOP_LOG_ECHO` | `string` | — | Não | Quando definido, ecoa os logs do desktop para stdout além de gravar no arquivo de log. |

### Variáveis injetadas em processos filhos (sidecars)

O `packaged` cria um env filtrado para daemon e web sidecars via `PACKAGED_CHILD_ENV_ALLOWLIST`:

```
HOME, HTTP_PROXY, HTTPS_PROXY, LANG, LC_ALL, LOGNAME, NO_PROXY, TMPDIR, USER, VP_HOME
```

Variáveis `*_API_KEY` e `*_TOKEN` são repassadas apenas quando `includeProviderSecrets = true`.

## Instalação

```bash
# A partir da raiz do monorepo
pnpm install
```

## Configuração Inicial

### Build do pacote

```bash
pnpm --filter @open-design/packaged build
```

Gera:
- `dist/index.mjs` — entry Electron
- `dist/headless.mjs` — entry headless Linux
- `dist/index.d.ts` — declarações TypeScript

### Pré-requisito para typecheck

O typecheck do `packaged` depende dos tipos de `@open-design/desktop`. Build o desktop primeiro:

```bash
pnpm --filter @open-design/desktop build
pnpm --filter @open-design/packaged typecheck
```

## Inicialização

### Modo normal (via tools-dev)

Para desenvolvimento, use o ciclo de vida completo do monorepo:

```bash
pnpm tools-dev
# ou com ports explícitos:
pnpm tools-dev run web --daemon-port 17456 --web-port 17573
```

### Modo empacotado (via tools-pack)

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

### Modo headless Linux (sem Electron)

```bash
# O entrypoint headless é exportado como ./headless no package
node -e "import('./dist/headless.mjs')"

# Com namespace customizado:
OD_PACKAGED_NAMESPACE=my-namespace node -e "import('./dist/headless.mjs')"
```

## Comandos Disponíveis

| Comando | Descrição |
|---------|-----------|
| `pnpm --filter @open-design/packaged build` | Build com esbuild + declarações TypeScript |
| `pnpm --filter @open-design/packaged test` | Testes unitários com Vitest |
| `pnpm --filter @open-design/packaged typecheck` | Type-check (requer desktop buildado) |
| `pnpm tools-pack mac build --to all` | Build completo macOS (DMG + zip) |
| `pnpm tools-pack win build --to nsis` | Build Windows (instalador NSIS) |
| `pnpm tools-pack linux build --to appimage` | Build Linux AppImage |
| `pnpm tools-pack linux build --containerized` | Build Linux em container |

## Verificação

Após o build, confirme:

```
apps/packaged/dist/
  index.mjs       ← Entry Electron
  headless.mjs    ← Entry headless
  index.d.ts      ← Declarações TypeScript
```

### Testes

```bash
pnpm --filter @open-design/packaged test
# Todos os testes devem passar (verde)
```

### Status do runtime (apps empacotados)

```bash
pnpm tools-dev inspect desktop status --json
# Deve retornar JSON com status dos sidecars
```

### Verificar log do desktop

```bash
pnpm tools-dev logs --json
# Logs em: <namespaceRoot>/logs/desktop/latest.log
```

## Troubleshooting

### `PackagedPathAccessError: Open Design cannot access its data folder`
- O diretório de dados foi criado por outro usuário (ex: com `sudo`).
- Corrija as permissões conforme as instruções exibidas na mensagem de erro:
  ```bash
  sudo chown -R "$USER":staff "<parent-path>"
  chmod -R u+rwX "<parent-path>"
  ```

### `setTypeOfService EINVAL` em macOS com VPN
- Erro benigno — o daemon registra e continua normalmente.
- Se aparecer em diálogo Electron em vez de log, verifique se `isHarmlessSocketOptionError()` está sendo aplicado na versão instalada.

### Daemon não inicia em 35 s
- Se estiver fazendo migração de dados legados, defina `OD_LEGACY_DATA_DIR` para ampliar o budget para 30 min.
- Verifique os logs do daemon em `<namespaceRoot>/logs/daemon/latest.log`.

### `pnpm typecheck` falha com erros em tipos do desktop
- Execute `pnpm --filter @open-design/desktop build` antes do typecheck do packaged.

### Colisão de paths entre canais (stable/beta/preview)
- Verifique se `OD_PACKAGED_NAMESPACE` é diferente para cada canal.
- Namespace padrão: `SIDECAR_DEFAULTS.namespace` (geralmente `"default"`).
