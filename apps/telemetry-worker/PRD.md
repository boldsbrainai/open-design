# PRD Técnico 360° — Telemetry Worker

## 1. Overview Técnico

| Campo | Valor |
|-------|-------|
| Pacote | `@open-design/telemetry-worker` v0.0.0 |
| Runtime | Cloudflare Workers (Compatibility Date `2026-05-01`) |
| Linguagem | TypeScript 5.6 (ESM) |
| Worker name | `open-design-telemetry-relay` |
| Account ID | `64ad4569ffd912432d6b86d5656484c4` |
| Domínio | `telemetry.open-design.ai` (custom domain) |
| `workers_dev` | `false` — exposto apenas via domínio customizado |
| Entry | `src/index.ts` |

O Worker é um **relay de segurança**: intercepta batches de telemetria do daemon, valida-os, e faz forward autenticado ao Langfuse. O cliente nunca precisa conhecer as credenciais Langfuse.

---

## 2. Backend / Servidor

O Worker **é** o servidor. Não há infraestrutura adicional.

### Handlers implementados

| Método | Path | Comportamento |
|--------|------|--------------|
| `POST` | `/api/langfuse` | Ingestão de telemetria (fluxo principal) |
| `GET` | `/api/langfuse` | Health check — retorna JSON de status |
| `GET` | `/health` | Health check — retorna JSON de status |
| `*` | `*` | 404 JSON |

### Interface `Env`
```ts
interface Env {
  LANGFUSE_PUBLIC_KEY?: string;       // Wrangler secret
  LANGFUSE_SECRET_KEY?: string;       // Wrangler secret
  LANGFUSE_BASE_URL?: string;         // Var em wrangler.toml
  TELEMETRY_CLIENT_RATE_LIMITER?: RateLimitBinding;
  TELEMETRY_IP_RATE_LIMITER?: RateLimitBinding;
}
```

---

## 3. Frontend / Cliente

Não há frontend. Os únicos "clientes" são:
1. **Daemon Open Design** (`apps/daemon`) — envia `POST /api/langfuse` com batches opt-in.
2. **Ferramentas de monitoramento** — fazem `GET /health` para verificar disponibilidade.

---

## 4. APIs e Contratos

### `POST /api/langfuse` — Ingestão de telemetria

**Headers obrigatórios:**
```
X-Open-Design-Telemetry: langfuse-ingestion-v1
Content-Type: application/json
```

**Body (JSON):**
```ts
{
  batch: Array<{
    id: string;        // ≤ 200 chars, não vazio
    type: EventType;   // ver allowlist abaixo
    body: object;      // objeto arbitrário
  }>
}
```

**Allowlist de tipos (`ALLOWED_EVENT_TYPES`):**
```
trace-create | span-create | generation-create | event-create | score-create
```

**Limites:**
- Body máximo: `1 MB` (`MAX_BODY_BYTES = 1024 * 1024`)
- Máximo de eventos por batch: `100` (`MAX_BATCH_EVENTS`)

**Respostas:**

| Status | Condição |
|--------|----------|
| `200` | Relay bem-sucedido para Langfuse |
| `400` | Header marker ausente, body inválido, tipo não permitido |
| `413` | Body excede 1 MB |
| `429` | Rate limit excedido |
| `5xx` | Erro ao conectar no Langfuse ou credenciais ausentes |

### `GET /health`

**Resposta `200`:**
```json
{ "status": "ok", "relay": "langfuse-ingestion-v1" }
```

---

## 5. Protocolos e Integrações

### Langfuse (upstream)
- URL base: `LANGFUSE_BASE_URL` (padrão: `https://us.cloud.langfuse.com`)
- Endpoint alvo: `<LANGFUSE_BASE_URL>/api/public/ingestion`
- Autenticação: `Authorization: Basic base64(LANGFUSE_PUBLIC_KEY:LANGFUSE_SECRET_KEY)`
- Método: `POST` com o body original repassado integralmente após validação.

### Cloudflare Rate Limiting Bindings
Dois rate limiters independentes configurados em `wrangler.toml`:

| Binding | `namespace_id` | Limite | Período | Key |
|---------|----------------|--------|---------|-----|
| `TELEMETRY_CLIENT_RATE_LIMITER` | `1001` | 120 req | 60 s | `client:<userId>` do trace-create |
| `TELEMETRY_IP_RATE_LIMITER` | `1002` | 600 req | 60 s | `ip:<CF-Connecting-IP>` |

### Extração de `userId` para rate limiting
`findTraceUserId()` percorre o batch procurando o primeiro evento `trace-create` com `body.userId` string não vazia. Trunca para 200 chars. Se não encontrado, o rate limit por client é pulado.

### Daemon ↔ Worker
- O daemon só conhece `OPEN_DESIGN_TELEMETRY_RELAY_URL` (baked em `open-design-config.json` pelo `tools-pack`).
- Se o Worker estiver indisponível: daemon retria, loga e continua sem bloquear o usuário.
- Modo de desenvolvimento local: daemon pode usar `LANGFUSE_PUBLIC_KEY` + `LANGFUSE_SECRET_KEY` diretamente, bypassando o relay.

---

## 6. Segurança e Autenticação

| Aspecto | Implementação |
|---------|--------------|
| Credenciais Langfuse | Armazenadas exclusivamente em Cloudflare Worker Secrets. |
| Marker header | `X-Open-Design-Telemetry: langfuse-ingestion-v1` — barreira básica contra requests aleatórios. |
| Validação estrutural | Rejeita qualquer body que não siga o schema Langfuse antes de tocar no rate limiter. |
| Rate limiting duplo | Protege contra abuso por user-id e por IP independentemente. |
| Limite de tamanho | Body > 1 MB rejeitado com 413 antes de parsing. |
| `workers_dev: false` | Worker não exposto em `*.workers.dev` — apenas via domínio customizado. |
| Forwarding do body | Body original (não reprocessado) é encaminhado ao Langfuse após validação — sem risco de injection via reformatação. |

---

## 7. Performance e Escalabilidade

| Dimensão | Detalhe |
|---------|---------|
| Latência | Cloudflare Workers edge — < 10 ms de overhead antes do forward para Langfuse. |
| Escalabilidade | Workers escala automaticamente; sem configuração de instâncias. |
| Rate limits | Configurados com headroom para uso normal; limites conservadores protegem Langfuse de burst. |
| Body parsing | Leitura com `MAX_BODY_BYTES` guard antes de `JSON.parse` — evita OOM por payloads gigantes. |
| Encode de auth | `basicAuthHeader()` usa `TextEncoder` + `btoa()` — sem dependências externas. |

---

## 8. Testes e Qualidade

| Ferramenta | Comando | Escopo |
|-----------|---------|--------|
| Vitest | `pnpm test` | Testes unitários em `tests/` |
| TypeScript | `pnpm typecheck` | `tsc -p tsconfig.json --noEmit` |
| Wrangler dev | `pnpm dev` | Desenvolvimento local com Worker runtime simulado |

### Funções candidatas a cobertura de testes
- `validateIngestionBody()` — muitos casos de borda (batch vazio, batch > 100, tipo inválido, id muito longa).
- `enforceRateLimits()` — mock de `RateLimitBinding` para simular limite atingido.
- `findTraceUserId()` — batch sem trace-create, userId não-string, userId vazia.
- `basicAuthHeader()` — verifica encoding Base64 correto.
- `jsonResponse()` — verifica headers `Cache-Control: no-store` e `Content-Type: application/json`.

### Deploy checklist
1. `pnpm typecheck` — 0 erros.
2. `pnpm test` — 100% passando.
3. `pnpm --filter @open-design/telemetry-worker deploy`.
4. Configurar `OPEN_DESIGN_TELEMETRY_RELAY_URL` no repositório como `https://telemetry.open-design.ai/api/langfuse`.
5. Verificar `GET /health` na URL de produção.
