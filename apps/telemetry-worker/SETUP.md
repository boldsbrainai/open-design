# Setup Guide — Telemetry Worker

## Pré-requisitos

| Requisito | Versão | Observação |
|-----------|--------|-----------|
| Node.js | `~24` | Obrigatório — engine declarado no `package.json` |
| pnpm | `10.33.2` | Via Corepack (`corepack enable`) |
| Wrangler CLI | via `pnpm dlx` | Não é instalado como dep — executado via `pnpm dlx wrangler` |
| Conta Cloudflare | — | Necessária para deploy; Account ID: `64ad4569ffd912432d6b86d5656484c4` |
| Credenciais Langfuse | — | `LANGFUSE_PUBLIC_KEY` e `LANGFUSE_SECRET_KEY` do projeto Langfuse |

## Variáveis de Ambiente

### Variáveis de ambiente Wrangler (configuradas em `wrangler.toml` → `[vars]`)

| Variável | Tipo | Padrão | Obrigatório | Descrição |
|----------|------|--------|-------------|-----------|
| `LANGFUSE_BASE_URL` | `string` | `https://us.cloud.langfuse.com` | Não | URL base do Langfuse. Para instâncias self-hosted ou na região EU, alterar aqui. |

### Secrets Cloudflare Worker (nunca em código ou `.env`)

| Secret | Obrigatório | Descrição |
|--------|-------------|-----------|
| `LANGFUSE_PUBLIC_KEY` | **Sim** | Chave pública do projeto Langfuse. Usada no header `Authorization: Basic`. |
| `LANGFUSE_SECRET_KEY` | **Sim** | Chave secreta do projeto Langfuse. **Nunca embutir em código.** |

### Bindings de rate limiting (configurados em `wrangler.toml` → `[[ratelimits]]`)

| Binding | `namespace_id` | Limite | Período | Descricão |
|---------|----------------|--------|---------|-----------|
| `TELEMETRY_CLIENT_RATE_LIMITER` | `1001` | 120 req | 60 s | Rate limit por `userId` extraído do batch |
| `TELEMETRY_IP_RATE_LIMITER` | `1002` | 600 req | 60 s | Rate limit por `CF-Connecting-IP` |

### Variável de repositório (CI/CD)

| Variável | Valor de produção | Descrição |
|----------|-------------------|-----------|
| `OPEN_DESIGN_TELEMETRY_RELAY_URL` | `https://telemetry.open-design.ai/api/langfuse` | URL pública do relay; baked pelo `tools-pack` no `open-design-config.json`. |

## Instalação

```bash
# A partir da raiz do monorepo
pnpm install
```

> O Worker não tem dependências de runtime além das APIs nativas do Cloudflare Workers. As `devDependencies` (`typescript`, `vitest`) são instaladas pelo pnpm workspace.

## Configuração Inicial

### 1. Autenticar no Cloudflare

```bash
pnpm --dir apps/telemetry-worker dlx wrangler login
```

### 2. Configurar os secrets

Os secrets **devem ser configurados antes do primeiro deploy**:

```bash
# Chave pública Langfuse
pnpm --dir apps/telemetry-worker dlx wrangler secret put LANGFUSE_PUBLIC_KEY

# Chave secreta Langfuse (nunca aparece em logs ou outputs)
pnpm --dir apps/telemetry-worker dlx wrangler secret put LANGFUSE_SECRET_KEY
```

Cada comando abre um prompt interativo. Cole o valor e pressione Enter.

> **Segurança:** Nunca passe secrets como argumentos de linha de comando (`--value`) pois eles ficam no histórico do shell.

### 3. Verificar `wrangler.toml`

Confirme que `account_id` e a rota de custom domain estão corretos:

```toml
name = "open-design-telemetry-relay"
account_id = "64ad4569ffd912432d6b86d5656484c4"
routes = [
  { pattern = "telemetry.open-design.ai", custom_domain = true }
]
```

## Inicialização

### Desenvolvimento local

```bash
pnpm --filter @open-design/telemetry-worker dev
# ou
pnpm --dir apps/telemetry-worker dlx wrangler dev
```

O Wrangler expõe o Worker localmente (geralmente em `http://127.0.0.1:8787`).

> Em modo local, `LANGFUSE_PUBLIC_KEY` e `LANGFUSE_SECRET_KEY` não estão disponíveis automaticamente. Use um arquivo `.dev.vars` (não commitado) para testes locais:
> ```
> # apps/telemetry-worker/.dev.vars  — NÃO COMMITAR
> LANGFUSE_PUBLIC_KEY=pk-lf-...
> LANGFUSE_SECRET_KEY=sk-lf-...
> ```

## Comandos Disponíveis

| Comando | Descrição |
|---------|-----------|
| `pnpm --filter @open-design/telemetry-worker dev` | Servidor de desenvolvimento local |
| `pnpm --filter @open-design/telemetry-worker test` | Testes unitários com Vitest |
| `pnpm --filter @open-design/telemetry-worker typecheck` | Type-check do Worker |
| `pnpm --filter @open-design/telemetry-worker deploy` | Deploy para produção |
| `pnpm --dir apps/telemetry-worker dlx wrangler secret put LANGFUSE_PUBLIC_KEY` | Configurar chave pública Langfuse |
| `pnpm --dir apps/telemetry-worker dlx wrangler secret put LANGFUSE_SECRET_KEY` | Configurar chave secreta Langfuse |
| `pnpm --dir apps/telemetry-worker dlx wrangler tail` | Stream de logs do Worker em produção |

## Verificação

### Após deploy

```bash
# Health check
curl https://telemetry.open-design.ai/health
# Esperado: {"status":"ok","relay":"langfuse-ingestion-v1"}

# Verificar que GET /api/langfuse também retorna health
curl https://telemetry.open-design.ai/api/langfuse
# Esperado: {"status":"ok","relay":"langfuse-ingestion-v1"}
```

### Testes locais

```bash
pnpm --filter @open-design/telemetry-worker test
# Todos os testes devem passar (verde)
```

### Typecheck

```bash
pnpm --filter @open-design/telemetry-worker typecheck
# Deve terminar sem erros
```

### Envio de batch de teste

```bash
curl -X POST https://telemetry.open-design.ai/api/langfuse \
  -H "Content-Type: application/json" \
  -H "X-Open-Design-Telemetry: langfuse-ingestion-v1" \
  -d '{"batch":[{"id":"test-1","type":"trace-create","body":{"name":"test"}}]}'
# Esperado: resposta 200 do Langfuse (ou 500 se secrets não configurados)
```

## Troubleshooting

### `500` ou `error: "missing credentials"` no relay
- `LANGFUSE_PUBLIC_KEY` ou `LANGFUSE_SECRET_KEY` não foram configurados via `wrangler secret put`.
- Execute os comandos de configuração de secrets na seção **Configuração Inicial**.

### `400 bad request` ao enviar batch
- Verifique se o header `X-Open-Design-Telemetry: langfuse-ingestion-v1` está presente.
- Verifique se `batch` é um array não vazio com objetos válidos (`id`, `type`, `body`).
- Tipos permitidos: `trace-create`, `span-create`, `generation-create`, `event-create`, `score-create`.

### `413` — body muito grande
- O batch excede 1 MB. Reduza o número de eventos por requisição.

### `429` — rate limit
- Aguarde o próximo período (60 s) ou verifique se há clients enviando mais de 120 req/min.

### Worker não encontrado em `telemetry.open-design.ai`
- Verifique se o custom domain está configurado no Cloudflare Dashboard.
- Verifique se o deploy foi bem-sucedido: `pnpm --dir apps/telemetry-worker dlx wrangler tail`.

### Daemon não envia telemetria
- Verifique se `OPEN_DESIGN_TELEMETRY_RELAY_URL` está corretamente baked em `open-design-config.json`.
- Para desenvolvimento local: defina `LANGFUSE_PUBLIC_KEY` + `LANGFUSE_SECRET_KEY` diretamente no daemon (bypass do relay).
