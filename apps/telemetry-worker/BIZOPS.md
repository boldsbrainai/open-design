# BizOps Blueprint — Telemetry Worker

## 1. Visão Geral

`apps/telemetry-worker` é um **Cloudflare Worker relay** de telemetria opt-in para o Open Design. O cliente desktop envia batches de ingestão Langfuse redatados ao Worker, que valida, aplica rate limits e os encaminha ao Langfuse com autenticação server-side. O objetivo central é manter `LANGFUSE_SECRET_KEY` fora dos bundles empacotados: clientes de release conhecem apenas a URL pública `https://telemetry.open-design.ai/api/langfuse`.

## 2. Objetivos de Negócio (OKRs)

### Objetivo 1 — Coleta confiável de telemetria opt-in
- **KR1:** Uptime do Worker ≥ 99,9 % (SLA Cloudflare Workers).
- **KR2:** Latência de relay P95 < 500 ms para batches válidos.
- **KR3:** Zero vazamento de credenciais Langfuse em bundles de release.

### Objetivo 2 — Proteção contra abuso e spam
- **KR1:** Rate limit por client/user-id: 120 req/min (`TELEMETRY_CLIENT_RATE_LIMITER`).
- **KR2:** Rate limit por IP: 600 req/min (`TELEMETRY_IP_RATE_LIMITER`).
- **KR3:** Rejeição de payloads inválidos antes do forward (validação estrutural completa).

### Objetivo 3 — Operabilidade e observabilidade
- **KR1:** Endpoint `/health` retorna JSON de status — monitorável por uptime checks.
- **KR2:** Deploy automatizado via `pnpm --filter @open-design/telemetry-worker deploy`.
- **KR3:** Segredos gerenciados via `wrangler secret put` — nunca em `.env` commitados.

## 3. Fluxos Operacionais

### Fluxo de ingestão de telemetria
1. Cliente daemon envia `POST /api/langfuse` com header `X-Open-Design-Telemetry: langfuse-ingestion-v1`.
2. Worker verifica o marker header (rejeita sem ele: 400).
3. Lê body com limite de 1 MB (`MAX_BODY_BYTES`); rejeita se exceder (413).
4. Valida estrutura JSON: `batch` array, 1–100 eventos, cada um com `id` (string ≤ 200 chars), `type` (allowlist) e `body` (objeto).
5. Aplica rate limits: por `userId` do trace-create (120/min) e por `CF-Connecting-IP` (600/min).
6. Monta `Authorization: Basic <base64(publicKey:secretKey)>` e faz forward para `LANGFUSE_BASE_URL/api/public/ingestion`.
7. Retorna a resposta do Langfuse ao cliente.

### Fluxo de health check
- `GET /health` ou `GET /api/langfuse` (GET) → retorna `{ status: "ok", relay: "langfuse-ingestion-v1" }`.

### Deploy
```bash
pnpm --filter @open-design/telemetry-worker deploy
```
Após o deploy, configurar a variável de repo `OPEN_DESIGN_TELEMETRY_RELAY_URL` com a rota do Worker.

### Rotação de segredos
```bash
pnpm --dir apps/telemetry-worker dlx wrangler secret put LANGFUSE_PUBLIC_KEY
pnpm --dir apps/telemetry-worker dlx wrangler secret put LANGFUSE_SECRET_KEY
```

## 4. Métricas e KPIs

| Métrica | Fonte | Meta |
|---------|-------|------|
| Uptime do Worker | Cloudflare Dashboard | ≥ 99,9 % |
| Requests bloqueados por rate limit | Workers Analytics | Monitorar picos |
| Requests rejeitados por validação | Workers Analytics | < 1 % em uso normal |
| Latência P95 de relay | Workers Analytics | < 500 ms |
| Testes passando | `vitest run` | 100 % |

## 5. Riscos e Mitigações

| Risco | Probabilidade | Impacto | Mitigação |
|-------|-------------|---------|-----------|
| Worker indisponível | Baixa | Baixo | Daemon retria, loga falha e **não bloqueia** o fluxo do usuário. |
| Abuso de terceiros via URL pública | Média | Médio | Rate limits duplos (client-id + IP) + validação do marker header. |
| Comprometimento de `LANGFUSE_SECRET_KEY` | Baixa | Alto | Armazenado apenas em Cloudflare Worker Secrets — nunca em código ou `.env`. |
| Langfuse API fora do ar | Baixa | Baixo | Worker retorna erro 5xx ao cliente; daemon trata como falha transitória. |
| Batch com eventos de tipo desconhecido | Média | Baixo | `ALLOWED_EVENT_TYPES` allowlist rejeita explicitamente com 400. |

## 6. Roadmap de Capacidades

| Fase | Capacidade | Status |
|------|-----------|--------|
| Atual | Relay de ingestão Langfuse com autenticação server-side | ✅ Entregue |
| Atual | Rate limiting duplo (client + IP) via Cloudflare bindings | ✅ Entregue |
| Atual | Validação estrutural completa do batch | ✅ Entregue |
| Atual | Health endpoint `/health` | ✅ Entregue |
| Próxima | Métricas de relay expostas via Workers Analytics API | 🔲 Planejado |
| Futura | Suporte a múltiplos backends de observabilidade além do Langfuse | 🔲 Backlog |
