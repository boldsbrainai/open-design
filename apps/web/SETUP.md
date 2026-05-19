# Setup Guide — Open Design Web (`apps/web`)

## Pré-requisitos

| Requisito | Versão mínima | Notas |
|---|---|---|
| Node.js | `~24` | Use `nvm` ou `fnm` para gerenciar versão |
| pnpm | `10.33.2` | Gerenciado via Corepack (ver abaixo) |
| `apps/daemon` rodando | — | O web consome todas as APIs do daemon |
| Git | qualquer recente | Para clonar o repositório |

> **Corepack**: o repositório usa `corepack` para garantir a versão correta do pnpm.
> Execute `corepack enable` uma vez após instalar o Node.js.

---

## Variáveis de Ambiente

Todas as variáveis são lidas em `apps/web/next.config.ts` no momento do build ou inicialização do servidor Next.js.

### Variáveis de runtime (lidas por `next.config.ts`)

| Variável | Tipo | Padrão | Obrigatório | Descrição |
|---|---|---|---|---|
| `OD_PORT` | `number` | `7456` | Não (dev) | Porta do daemon Express local. O web proxia `/api/*`, `/artifacts/*`, `/frames/*` para `http://127.0.0.1:$OD_PORT` em modo dev. Ignorada em produção estática. |
| `OD_WEB_OUTPUT_MODE` | `'server' \| 'standalone' \| undefined` | `undefined` | Não | Modo de output do Next.js. `undefined` = export estático (padrão produção). `'server'` = SSR para packaged desktop. `'standalone'` = Next.js standalone server. |
| `OD_WEB_DIST_DIR` | `string` | `'out'` (prod) / `'.next'` (dev) | Não | Diretório de saída do build. Pode ser absoluto ou relativo a `apps/web/`. Ignorado se `OD_WEB_PROD=1`. |
| `OD_WEB_PROD` | `'1' \| undefined` | `undefined` | Não | Força uso do `distDir` padrão (`out` ou `.next`), ignorando `OD_WEB_DIST_DIR`. Usado pelos empacotadores de produção. |
| `OD_WEB_TSCONFIG_PATH` | `string` | `undefined` | Não | Caminho customizado do `tsconfig.json` para builds especiais. Pode ser absoluto ou relativo a `apps/web/`. |
| `NODE_ENV` | `'development' \| 'production' \| 'test'` | `'development'` | Não | Controla modo dev vs prod. `'development'` ativa rewrites e Turbopack. `'production'` ativa export estático por padrão. |

### Variáveis injetadas pelo `tools-dev` (ambiente de desenvolvimento)

O `tools-dev` seta estas variáveis automaticamente antes de iniciar o web:

| Variável | Descrição |
|---|---|
| `OD_PORT` | Porta onde o daemon está escutando (default: `7456`, configurável via `--daemon-port`). |
| `OD_WEB_PORT` | Porta onde o servidor Next.js vai escutar (configurável via `--web-port`). |

> **Nota**: não use `NEXT_PORT`; o web usa `OD_WEB_PORT` injetado pelo `tools-dev`.

### Configuração do usuário (armazenada em `localStorage`)

Estas não são variáveis de ambiente — são preferências persistidas no browser pelo app:

| Chave localStorage | Tipo | Padrão | Descrição |
|---|---|---|---|
| `open-design:config` | JSON (`AppConfig`) | ver abaixo | Toda configuração do usuário. |
| `open-design:config` → `mode` | `'daemon' \| 'direct'` | `'daemon'` | Modo de execução: via daemon ou direto ao provedor IA. |
| `open-design:config` → `apiProtocol` | `'anthropic' \| 'openai' \| 'azure' \| ...` | `'anthropic'` | Protocolo do provedor de IA selecionado. |
| `open-design:config` → `model` | `string` | `'claude-sonnet-4-5'` | Modelo padrão. |
| `open-design:config` → `apiKey` | `string` | `''` | API key do provedor (quando `mode === 'direct'`). |
| `open-design:config` → `baseUrl` | `string` | `'https://api.anthropic.com'` | Base URL do provedor. |
| `open-design:config` → `theme` | `'light' \| 'dark' \| 'system'` | `'system'` | Tema da interface. |
| `open-design:config` → `accentColor` | `string` (hex) | `'#c96442'` | Cor de destaque customizável. |

---

## Instalação

### 1. Clonar o repositório (se ainda não tiver)

```bash
git clone <url-do-repositório> open-design
cd open-design
```

### 2. Habilitar Corepack

```bash
corepack enable
```

### 3. Instalar todas as dependências do monorepo

```bash
pnpm install
```

> Esse comando instala as dependências de **todos** os pacotes do workspace, incluindo `apps/web`.

---

## Configuração Inicial

### Configurar o daemon primeiro

O `apps/web` não funciona de forma isolada — ele consome APIs do `apps/daemon`. Configure e inicie o daemon antes de rodar o web.

Consulte `apps/daemon/SETUP.md` para configurar o daemon, incluindo variáveis `OD_DATA_DIR`, `OD_MEDIA_CONFIG_DIR`, e chaves de API.

### Configurar provedor de IA (opcional no build, obrigatório para usar)

A configuração do provedor de IA é feita **dentro da interface** após iniciar o app:

1. Abra o app no browser.
2. Clique no ícone de configurações (⚙️) ou acesse `Settings` via menu.
3. Em **Execution Mode**, escolha entre "Use daemon" (recomendado) ou "Direct API".
4. Selecione o protocolo (Anthropic, OpenAI, Azure, etc.).
5. Insira a API key e clique em "Test connection".

---

## Inicialização

### Modo desenvolvimento (recomendado — com daemon integrado)

O modo padrão usa `pnpm tools-dev` que inicia daemon e web juntos, com portas gerenciadas automaticamente:

```bash
# Iniciar daemon + web juntos (portas padrão)
pnpm tools-dev

# Iniciar com portas customizadas
pnpm tools-dev run web --daemon-port 17456 --web-port 17573

# Iniciar apenas o web (daemon já em execução separada)
pnpm tools-dev start web
```

O web estará disponível em `http://localhost:<OD_WEB_PORT>` (padrão: `http://localhost:3000`).

### Modo desenvolvimento (apenas o web, daemon em outro processo)

Se o daemon já estiver rodando na porta `7456`:

```bash
cd apps/web
OD_PORT=7456 pnpm dev
```

### Modo produção (export estático)

```bash
# Build de produção (gera 'apps/web/out/')
pnpm --filter @open-design/web build

# O daemon servirá os arquivos estáticos de 'out/'
# (configurado automaticamente pelo daemon)
```

### Modo packaged (desktop Electron)

Para builds packaged (desktop), use o `tools-pack`:

```bash
# macOS
pnpm tools-pack mac build --to all

# Windows
pnpm tools-pack win build --to nsis

# Linux
pnpm tools-pack linux build --to appimage
```

O web no packaged usa `OD_WEB_OUTPUT_MODE=server` e um sidecar Next.js SSR próprio.

---

## Comandos Disponíveis

### Comandos do pacote `@open-design/web`

```bash
# Servidor de desenvolvimento (Next.js com Turbopack)
pnpm --filter @open-design/web dev

# Build de produção (export estático para 'out/')
pnpm --filter @open-design/web build

# Build do sidecar (TypeScript → dist/sidecar/)
pnpm --filter @open-design/web build:sidecar

# Verificação de tipos TypeScript
pnpm --filter @open-design/web typecheck

# Rodar testes Vitest
pnpm --filter @open-design/web test
```

### Comandos do workspace (raiz)

```bash
# Verificação de tipos em todos os pacotes
pnpm typecheck

# Guard: detecta arquivos JS não-autorizados, violações de boundary
pnpm guard

# Desenvolvimento completo (daemon + web + desktop)
pnpm tools-dev

# Status dos processos em execução
pnpm tools-dev status --json

# Logs do web em execução
pnpm tools-dev logs --json

# Parar todos os processos
pnpm tools-dev stop
```

---

## Verificação

### Verificar que o web está funcionando

```bash
# 1. Checar status via tools-dev
pnpm tools-dev status --json

# 2. Checar que o daemon está acessível
curl http://localhost:7456/api/health

# 3. Abrir o browser
open http://localhost:3000   # macOS
start http://localhost:3000  # Windows
```

### Verificar build de produção

```bash
# Após 'pnpm --filter @open-design/web build'
ls apps/web/out/             # Deve ter index.html, _next/, etc.
```

### Verificar tipos TypeScript

```bash
pnpm --filter @open-design/web typecheck
# Saída esperada: sem erros
```

### Verificar testes

```bash
pnpm --filter @open-design/web test
# Saída esperada: todos os testes passando
```

---

## Troubleshooting

### O web não consegue chamar as APIs do daemon

**Sintoma**: erros de rede no console do browser, interface não carrega projetos.

**Verificações**:
1. Confirme que o daemon está rodando: `pnpm tools-dev status --json`
2. Confirme que `OD_PORT` bate com a porta do daemon: `curl http://localhost:$OD_PORT/api/health`
3. Em dev, confirme que os rewrites do `next.config.ts` estão ativos (deve aparecer no console do Next.js na inicialização).
4. Se usando porta diferente de `7456`, passe `--daemon-port <porta>` para `pnpm tools-dev`.

### Erro de tipo `crypto.randomUUID is not a function`

**Contexto**: acesso via HTTP (não HTTPS) em LAN, Docker, ou unRAID.

**Causa**: `crypto.randomUUID()` requer contexto seguro (HTTPS ou localhost). LAN IPs são não-seguros.

**Solução**: o app já tem fallback automático em `src/utils/uuid.ts`. Se o erro persistir, verifique se o fallback está sendo invocado.

### FOUC (Flash of Unstyled Content) no tema escuro/claro

**Causa**: `localStorage` lido após render.

**Solução**: já mitigado com script inline em `app/layout.tsx`. Se reaparecer, verifique se o script está presente no HTML servido (`view-source:`).

### `pnpm dev` inicia mas proxy não funciona (CORS errors)

**Sintoma**: chamadas para `/api/` retornam erros de CORS.

**Verificações**:
1. Confirme que `NODE_ENV=development` (o rewrite só existe em dev).
2. Confirme que `OD_PORT` está setado corretamente.
3. Reinicie o servidor dev após mudar variáveis de ambiente.

### Build falha com erro de TypeScript em locale

**Sintoma**: `Type 'X' is not assignable to type 'Dict'` apontando para um arquivo de locale.

**Causa**: nova chave adicionada em `src/i18n/types.ts` não foi implementada em todas as 19 locales.

**Solução**: adicione a chave faltante em **todas** as locales em `src/i18n/locales/`. O typecheck valida 100% de cobertura.

### Artefato HTML não renderiza (iframe em branco)

**Possíveis causas**:

1. **Decisão errada de render mode**: verifique `src/components/file-viewer-render-mode.ts`. Features que precisam de bridges (palette, edit, tweaks) só funcionam via `srcDoc`.
2. **Sandbox muito restritivo**: verifique se o HTML precisa de `htmlNeedsSandboxShim()`.
3. **Daemon indisponível**: o preview depende de `/artifacts/*` ou `/api/projects/:id/files/:path/preview`. Confirme que o daemon está rodando.

### Critique Theater não aparece

**Verificações**:
1. Settings → Critique Theater → confirme que está habilitado.
2. O toggle é persistido em `localStorage` (`open-design:config`) e sincronizado via `CustomEvent('open-design:critique-theater-toggle')`.
3. Verifique se o projeto tem um artefato HTML gerado (o Theater só ativa após geração de artefato).

### Instalação de dependências falha com versão errada do pnpm

**Causa**: pnpm fora da versão esperada (`10.33.2`).

**Solução**:
```bash
corepack enable
corepack prepare pnpm@10.33.2 --activate
pnpm install
```
