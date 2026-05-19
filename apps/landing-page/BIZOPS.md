# BizOps Blueprint — Landing Page

## 1. Visão Geral

A `apps/landing-page` é o site público de marketing do **Open Design**, hospedado em `open-design.ai` via Cloudflare Pages (projeto `open-design-landing`). É uma aplicação Astro v5 com output estático, React 18 no servidor para pré-renderização e zero JavaScript de runtime padrão no cliente. Serve como porta de entrada para descoberta do produto, catálogo de skills, design systems, craft rules, templates e blog.

## 2. Objetivos de Negócio (OKRs)

### Objetivo 1 — Conversão de visitantes em usuários
- **KR1:** Aumentar cliques no CTA de download via `/releases` em 20% MoM.
- **KR2:** Taxa de bounce < 60% em páginas de catálogo.
- **KR3:** Stars no GitHub exibidas em tempo real na hero section (via API do GitHub).

### Objetivo 2 — Autoridade de conteúdo no nicho de design com IA
- **KR1:** Publicar ao menos 2 posts de blog por mês refletindo novos recursos.
- **KR2:** Todas as páginas de catálogo indexadas e com `<link rel="canonical">` correto.
- **KR3:** Sitemap atualizado a cada deploy com `lastmod` dos posts de blog.

### Objetivo 3 — Descoberta do catálogo de capabilities
- **KR1:** Exibir counts atualizados de skills e design systems na hero (dados reais de `getCatalogCounts()`).
- **KR2:** Páginas `/skills/`, `/systems/`, `/craft/` e `/templates/` com previews gerados por `pnpm previews`.
- **KR3:** Feed RSS funcional em `/rss.xml`.

## 3. Fluxos Operacionais

### Deploy
1. Push para `main` → CI constrói com `astro check && astro build` → Cloudflare Pages publica `out/`.
2. Deployments de preview: `OD_LANDING_SITE` é injetado pela Cloudflare para stampar a URL correta sem forkar a config.
3. Atualizações do catálogo: Skills, design-systems, craft e templates são lidos diretamente dos diretórios-raiz do monorepo via Content Collections do Astro durante o build.

### Geração de Previews
1. Executar `pnpm --filter @open-design/landing-page previews` localmente.
2. O script `scripts/generate-previews.ts` usa Playwright para capturar `example.html` de cada skill/template.
3. Imagens salvas em `public/previews/<bucket>/<slug>.{png,webp}`.
4. Commitar as imagens antes do próximo deploy de produção.

### Blog
- Posts em `app/content/blog/*.md` com frontmatter `date: YYYY-MM-DD`.
- O sitemap lê as datas em tempo de build; posts sem data não recebem `lastmod`.

## 4. Métricas e KPIs

| Métrica | Fonte | Meta |
|---------|-------|------|
| Pageviews mensais | Cloudflare Analytics | +15% MoM |
| Core Web Vitals (LCP) | Web Vitals / Lighthouse | < 2.5 s |
| Cobertura de previews | `public/previews/` vs catálogo | 100 % skills |
| GitHub stars exibidas | API `nexu-io/open-design` | Atualização diária |
| Erros de typecheck no CI | `astro check` | 0 |

## 5. Riscos e Mitigações

| Risco | Probabilidade | Impacto | Mitigação |
|-------|-------------|---------|-----------|
| GitHub API rate-limit zerando stars | Média | Baixo | `getGithubRepoMeta()` retorna `null`; UI oculta o campo graciosamente. |
| Previews desatualizadas vs. skills | Alta | Médio | Pipeline de CI deve rodar `pnpm previews` periodicamente. |
| `OD_LANDING_SITE` errado em preview | Baixa | Médio | Variável configurada por ambiente no Cloudflare Pages. |
| Conteúdo `/og/` indexado por crawlers | Baixa | Baixo | `noindex` no `<head>` + `Disallow` no `robots.txt` + filtro no sitemap. |
| Build quebra por mudança em SKILL.md schema | Média | Alto | `skillSchema` usa `.passthrough()` — campos extras não causam falha. |

## 6. Roadmap de Capacidades

| Fase | Capacidade | Status |
|------|-----------|--------|
| Atual | Catálogo estático de skills, systems, craft, templates | ✅ Entregue |
| Atual | Blog com RSS e sitemap | ✅ Entregue |
| Atual | Previews gerados por Playwright | ✅ Entregue |
| Próxima | Internacionalização (i18n) das páginas de marketing | 🔲 Planejado |
| Próxima | Busca client-side no catálogo | 🔲 Planejado |
| Futura | CDN de previews via Cloudflare Images | 🔲 Backlog |
