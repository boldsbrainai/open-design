# Setup Guide — Daemon (`@open-design/daemon`)

## Pré-requisitos

| Dependência | Versão mínima | Como verificar |
|---|---|---|
| Node.js | `~24` (qualquer 24.x) | `node --version` |
| pnpm | `10.33.2` (via Corepack) | `pnpm --version` |
| Git | qualquer | `git --version` |
| (opcional) Claude Code CLI | qualquer | `claude --version` |
| (opcional) OpenAI Codex CLI | qualquer | `codex --version` |
| (opcional) GitHub Copilot CLI | qualquer | `github-copilot-cli --version` |

> **Importante**: Use Corepack para garantir a versão de pnpm fixada no `package.json` raiz:
> ```bash
> corepack enable
> corepack prepare
> ```

---

## Variáveis de Ambiente

### Servidor HTTP

| Variável | Tipo | Padrão | Obrigatório | Descrição |
|---|---|---|---|---|
| `OD_PORT` | `number` | `7456` | Não | Porta HTTP do daemon |
| `OD_BIND_HOST` | `string` | `127.0.0.1` | Não | Endereço de bind (loopback por padrão) |
| `OD_WEB_PORT` | `number` | — | Não | Porta da web UI (usado na validação de origem) |
| `OD_ALLOWED_ORIGINS` | `string` | — | Não | Origens extras permitidas, separadas por vírgula (ex: `http://localhost:3000`) |

### Paths de Dados

| Variável | Tipo | Padrão | Obrigatório | Descrição |
|---|---|---|---|---|
| `OD_DATA_DIR` | `string` | `<projectRoot>/.od` | Não | Raiz de todos os dados de runtime (SQLite, projetos, config). Suporta `~/` e `$HOME` |
| `OD_MEDIA_CONFIG_DIR` | `string` | `<OD_DATA_DIR>` | Não | Override para localização de `media-config.json` apenas |
| `OD_RESOURCE_ROOT` | `string` | — | Não | Override para recursos read-only (skills/, design-systems/, frames/). Deve estar dentro do workspace root |
| `OD_USER_STATE_DIR` | `string` | `~/.open-design` | Não | Diretório para tokens de deploy (vercel.json, cloudflare-pages.json) |
| `OD_LEGACY_DATA_DIR` | `string` | — | Não | Fonte para migração de dados legados |

### Daemon

| Variável | Tipo | Padrão | Obrigatório | Descrição |
|---|---|---|---|---|
| `OD_DAEMON_CLI_PATH` | `string` | auto-resolvido | Não | Path explícito para o binário `od` (cli.js) |
| `OD_BIN` | `string` | auto-resolvido | Não | Alternativa a `OD_DAEMON_CLI_PATH` |

### Plugin System

| Variável | Tipo | Padrão | Obrigatório | Descrição |
|---|---|---|---|---|
| `OD_MAX_DEVLOOP_ITERATIONS` | `number` | `10` | Não | Máximo de iterações do devloop de plugin por stage |
| `OD_SNAPSHOT_UNREFERENCED_TTL_DAYS` | `number` | `30` | Não | Dias até GC de snapshots não referenciados |
| `OD_SNAPSHOT_RETENTION_DAYS` | `number` | — | Não | Retenção máxima de snapshots referenciados (sem limite se não definido) |
| `OD_SNAPSHOT_GC_INTERVAL_MS` | `number` | `21600000` (6h) | Não | Intervalo do worker de GC de snapshots |

### Critique Theater (Design Jury)

| Variável | Tipo | Padrão | Obrigatório | Descrição |
|---|---|---|---|---|
| `OD_CRITIQUE_ENABLED` | `boolean` | `false` | Não | Habilita o Critique Theater |
| `OD_CRITIQUE_MAX_ROUNDS` | `number` | veja `defaultCritiqueConfig()` | Não | Máximo de rounds por sessão |
| `OD_CRITIQUE_SCORE_THRESHOLD` | `number` | veja `defaultCritiqueConfig()` | Não | Score mínimo para encerrar (escala 0..OD_CRITIQUE_SCORE_SCALE) |
| `OD_CRITIQUE_SCORE_SCALE` | `number` | veja `defaultCritiqueConfig()` | Não | Denominador da escala de score |
| `OD_CRITIQUE_PER_ROUND_TIMEOUT_MS` | `number` | veja `defaultCritiqueConfig()` | Não | Timeout por round em ms |
| `OD_CRITIQUE_TOTAL_TIMEOUT_MS` | `number` | veja `defaultCritiqueConfig()` | Não | Timeout total da sessão em ms |
| `OD_CRITIQUE_PARSER_MAX_BLOCK_BYTES` | `number` | veja `defaultCritiqueConfig()` | Não | Tamanho máximo de um bloco do parser |
| `OD_CRITIQUE_FALLBACK_POLICY` | `string` | veja `defaultCritiqueConfig()` | Não | Política quando adapter degrada: `abort` ou `continue` |

### Telemetria (opt-in, desativado por padrão)

| Variável | Tipo | Padrão | Obrigatório | Descrição |
|---|---|---|---|---|
| `POSTHOG_KEY` | `string` | — | Não | Chave PostHog para analytics. Sem ela, PostHog é no-op |
| `POSTHOG_HOST` | `string` | `https://us.i.posthog.com` | Não | Host PostHog customizado |
| `OPEN_DESIGN_TELEMETRY_RELAY_URL` | `string` | — | Não | URL do relay de telemetria Open Design (Langfuse via relay) |
| `LANGFUSE_PUBLIC_KEY` | `string` | — | Não | Chave pública Langfuse (acesso direto, sem relay) |
| `LANGFUSE_SECRET_KEY` | `string` | — | Não | Chave secreta Langfuse (acesso direto, sem relay) |

> **Nota de privacidade**: nenhuma telemetria é enviada sem `prefs.metrics = true` no app-config. Langfuse adicionalmente requer `prefs.content = true`.

### Provedores de Mídia

Todas as chaves abaixo são opcionais. A ausência de uma chave desativa o provedor correspondente.

| Variável | Provedor |
|---|---|
| `OD_OPENAI_API_KEY` / `OPENAI_API_KEY` / `AZURE_API_KEY` / `AZURE_OPENAI_API_KEY` | OpenAI / Azure OpenAI |
| `OD_VOLCENGINE_API_KEY` / `ARK_API_KEY` / `VOLCENGINE_API_KEY` | Volcengine ARK |
| `OD_GROK_API_KEY` / `XAI_API_KEY` | xAI Grok |
| `OD_NANOBANANA_API_KEY` / `GOOGLE_API_KEY` / `GEMINI_API_KEY` | Google / NanoBanana |
| `OD_IMAGEROUTER_API_KEY` / `IMAGEROUTER_API_KEY` | ImageRouter |
| `OD_CUSTOM_IMAGE_API_KEY` / `CUSTOM_IMAGE_API_KEY` | Custom Image provider |
| `OD_BFL_API_KEY` / `BFL_API_KEY` | Black Forest Labs (FLUX) |
| `OD_FAL_KEY` / `FAL_KEY` | fal.ai |
| `OD_REPLICATE_API_TOKEN` / `REPLICATE_API_TOKEN` | Replicate |
| `OD_GOOGLE_API_KEY` | Google (Imagen/Veo) |
| `OD_KLING_API_KEY` / `KLING_API_KEY` | Kling (vídeo) |
| `OD_MIDJOURNEY_API_KEY` | Midjourney |
| `OD_MINIMAX_API_KEY` / `MINIMAX_API_KEY` | MiniMax |
| `OD_SUNO_API_KEY` | Suno (áudio) |
| `OD_UDIO_API_KEY` | Udio (áudio) |
| `OD_ELEVENLABS_API_KEY` / `ELEVENLABS_API_KEY` | ElevenLabs (TTS/voz) |
| `OD_MEDIA_MODEL_ALIASES` | JSON string de aliases de modelo (ex: `'{"model-a":"model-b"}'`) |

### Agentes de IA

As chaves abaixo são lidas pelos CLIs de agente via env passado no spawn. Podem ser definidas diretamente no ambiente ou via `agentCliEnv` no app-config.

| Variável | Agente |
|---|---|
| `ANTHROPIC_BASE_URL` + `ANTHROPIC_API_KEY` | Claude Code (proxy) |
| `OPENAI_BASE_URL` + `OPENAI_API_KEY` | Codex (proxy) |
| `CODEX_HOME` | Codex — diretório home (para `generated_images/`) |

---

## Instalação

```bash
# 1. Clone o repositório (se ainda não tiver)
git clone https://github.com/nexu-io/open-design.git
cd open-design

# 2. Ative o pnpm correto via Corepack
corepack enable
corepack prepare

# 3. Instale as dependências do workspace
pnpm install

# 4. Build do daemon (e das dependências de workspace)
pnpm --filter @open-design/daemon typecheck
pnpm --filter @open-design/daemon build
```

> O comando `pnpm --filter @open-design/daemon typecheck` também builda `@open-design/contracts` e `@open-design/registry-protocol` automaticamente, pois estão no script `typecheck`.

---

## Configuração Inicial

### 1. Arquivo de dados (opcional)

Por padrão, todos os dados ficam em `<projectRoot>/.od/`. Para usar um diretório diferente (ex: isolamento por namespace ou instala empacotada):

```bash
export OD_DATA_DIR=/caminho/para/meu-od-dir
```

### 2. Chaves de API de mídia (opcional)

As chaves podem ser definidas de duas formas:

**Via variável de ambiente** (recomendado para CI/automação):
```bash
export OPENAI_API_KEY=sk-...
export ELEVENLABS_API_KEY=...
```

**Via Settings da UI** (recomendado para uso interativo):
- Abra a UI web → Settings → Media Providers → insira as chaves

As chaves inseridas via UI são salvas em `<OD_DATA_DIR>/media-config.json`.

### 3. Tokens de deploy (opcional)

Os tokens Vercel e Cloudflare Pages são salvos em `~/.open-design/` ao conectar via UI de deploy. Podem ser armazenados em diretório alternativo com:

```bash
export OD_USER_STATE_DIR=/caminho/customizado
```

---

## Inicialização

### Modo desenvolvimento (recomendado)

Use sempre o `tools-dev` como ponto de entrada para desenvolvimento local. Ele gerencia portas, namespaces e env corretamente:

```bash
# Inicia daemon + web juntos (recomendado)
pnpm tools-dev

# Inicia apenas o daemon em porta customizada
pnpm tools-dev run web --daemon-port 17456 --web-port 17573

# Verifica status
pnpm tools-dev status --json
```

### Modo direto (baixo nível)

```bash
# Build + start direto (sem abrir browser)
pnpm --filter @open-design/daemon build
node apps/daemon/dist/cli.js --no-open

# Com porta customizada
node apps/daemon/dist/cli.js --port 7456 --no-open

# Com host customizado (cuidado: não exponha fora do loopback em produção)
node apps/daemon/dist/cli.js --host 127.0.0.1 --port 7456 --no-open
```

### Modo produção / packaged

No modo packaged, o daemon é iniciado como sidecar pelo `apps/packaged`. Não inicie manualmente em produção empacotada — use o Electron app.

### Modo teste

```bash
pnpm --filter @open-design/daemon test
```

Os testes usam `OD_DATA_DIR` para isolar dados de test em diretório temporário.

---

## Comandos Disponíveis

Todos os scripts do `package.json` do daemon:

| Comando | Descrição |
|---|---|
| `pnpm --filter @open-design/daemon build` | Compila TypeScript → `dist/` via `tsc` |
| `pnpm --filter @open-design/daemon test` | Roda testes com Vitest |
| `pnpm --filter @open-design/daemon typecheck` | Typechecking sem emit (inclui build de contracts) |

Scripts do `od` bin (após build):

| Comando | Descrição |
|---|---|
| `node dist/cli.js` | Inicia o daemon e abre o browser |
| `node dist/cli.js --no-open` | Inicia o daemon sem abrir o browser |
| `node dist/cli.js --port <N>` | Porta HTTP customizada |
| `node dist/cli.js --host <H>` | Bind host customizado |
| `node dist/cli.js media generate ...` | Gera mídia via daemon em execução |
| `node dist/cli.js mcp` | Inicia MCP server stdio |
| `node dist/cli.js artifacts ...` | CLI de artifacts |
| `node dist/cli.js plugin ...` | CLI de plugins |
| `node dist/cli.js research search --query "..."` | CLI de pesquisa |
| `node dist/cli.js connectors-tools ...` | CLI de connectors |
| `node dist/cli.js live-artifacts-tools ...` | CLI de live artifacts |

---

## Verificação

### Confirmar que o daemon está rodando

```bash
# Health check
curl http://localhost:7456/api/health
# Resposta esperada: {"ok":true}

# Versão
curl http://localhost:7456/api/version

# Agentes disponíveis
curl http://localhost:7456/api/agents

# Skills carregadas
curl http://localhost:7456/api/skills
```

### Via tools-dev

```bash
pnpm tools-dev status --json
# Procure por "daemon": { "status": "running", "url": "http://127.0.0.1:7456" }
```

### Logs

```bash
pnpm tools-dev logs --json
# Ou com namespace específico:
pnpm tools-dev logs --namespace <name> --json
```

Logs do daemon ficam em `.tmp/tools-dev/<namespace>/daemon.log`.

---

## Troubleshooting

### Daemon não inicia — `EADDRINUSE`

A porta já está em uso. Verifique com `lsof -i :7456` (macOS/Linux) ou `netstat -ano | findstr :7456` (Windows) e encerre o processo conflitante, ou use outra porta:

```bash
node dist/cli.js --port 7457 --no-open
```

### `OD_DATA_DIR is not writable`

O diretório configurado em `OD_DATA_DIR` não tem permissão de escrita. Verifique:
```bash
ls -la $(dirname $OD_DATA_DIR)
chmod 755 $OD_DATA_DIR
```

### Agente não encontrado — `detectAgents()` retorna lista vazia

O CLI do agente não está no PATH. Instale-o conforme a documentação do agente:

- Claude Code: `npm install -g @anthropic-ai/claude-code`
- Codex: `npm install -g @openai/codex`
- Copilot CLI: via extensão GitHub Copilot CLI

Após instalar, reinicie o daemon para nova detecção.

### Erro `AGENT_PROMPT_TOO_LARGE`

O prompt excede o orçamento de argv/stdin do agente. Causas comuns:
- Muitos arquivos no projeto
- System prompt muito longo (muitos craft sections + skill body)

Mitigações:
- Reduza o número de arquivos no projeto
- Use um skill com body menor
- Separe em múltiplas conversas

### Chaves de provedor de mídia rejeitadas

Verifique se a chave está correta e ativa. As chaves de env têm precedência sobre `media-config.json`. Para depurar:

```bash
curl http://localhost:7456/api/media/config
# Verifica quais provedores têm chave configurada (mascarada)
```

### Banco SQLite corrompido

```bash
# Backup e reset
cp .od/app.sqlite .od/app.sqlite.bak
rm .od/app.sqlite
# Reinicie o daemon — o schema será recriado automaticamente
```

> Projetos e arquivos ficam em `.od/projects/` — não são afetados pela deleção do `app.sqlite`.

### `OD_RESOURCE_ROOT must be under the workspace root`

A variável `OD_RESOURCE_ROOT` aponta para um diretório fora do workspace. Verifique se o path está correto e é relativo ou absoluto dentro do repositório.

### Port em conflito com outro namespace

Ao usar `tools-dev` com múltiplos namespaces, cada um precisa de porta própria:

```bash
pnpm tools-dev run web --daemon-port 17456 --web-port 17573  # namespace 1
pnpm tools-dev run web --daemon-port 17457 --web-port 17574  # namespace 2
```

### Erros de TypeScript ao buildar

```bash
# Certifique-se de buildar as dependências primeiro
pnpm --filter @open-design/contracts build
pnpm --filter @open-design/registry-protocol build
# Depois
pnpm --filter @open-design/daemon build
```

Ou simplesmente use `pnpm --filter @open-design/daemon typecheck` que já faz isso automaticamente.
