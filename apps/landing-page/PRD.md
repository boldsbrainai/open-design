# PRD Técnico 360° — Landing Page

## 1. Overview Técnico

| Campo | Valor |
|-------|-------|
| Pacote | `@open-design/landing-page` v0.7.0 |
| Framework | Astro v5.15 — output `static` |
| Linguagem | TypeScript 5.6 (ESM), React 18.3 (server-only, sem hydration padrão) |
| Node runtime | `~24` |
| Deployment | Cloudflare Pages (projeto `open-design-landing`) |
| Domínio | `https://open-design.ai` |
| Source dir | `app/` |
| Output dir | `out/` |

A landing page é gerada inteiramente em build-time como HTML/CSS estático. Não há API routes ativas em runtime — toda a dinamicidade (stars do GitHub, versão do release) é carregada client-side por um script inline minimalista.

---

## 2. Backend / Servidor

Não há servidor de runtime. Todo o "backend" é **build-time**:

### Content Collections (Astro)
Definidas em `app/content.config.ts`:

| Collection | Glob | Tipo |
|------------|------|------|
| `skills` | `../../skills/*/SKILL.md` | `skillSchema` (zod, passthrough) |
| `systems` | `../../design-systems/*/DESIGN.md` | passthrough |
| `craft` | `../../craft/*.md` | passthrough |
| `templates` | `../../templates/live-artifacts/*/README.md` | passthrough |
| `blog` | `app/content/blog/*.md` | frontmatter com `date` |

### Funções Cloudflare Pages
Diretório `functions/` — presente na estrutura para futuras integrações server-side se necessário.

### Script de geração de previews
`scripts/generate-previews.ts` usa **Playwright** para abrir `example.html` de cada skill/template e capturar screenshots em `public/previews/<bucket>/<slug>.{png,webp}`. Executado offline, não em CI padrão.

---

## 3. Frontend / Cliente

### Renderização
- `app/page.tsx` — componente React raiz pré-renderizado via `renderToStaticMarkup()` dentro do template Astro.
- CSS: `app/globals.css` + `app/sub-pages.css`.
- Plugins de página registrados em `app/plugin-registry.ts`.

### Script inline no cliente
Único bloco `<script is:inline>` em `app/pages/index.astro` responsável por:
- Formatar stars do GitHub (`formatStars(count)`)
- Formatar versão do release (`formatVersion(release)`)
- Chamar `enhanceHeader()` após `DOMContentLoaded`
- Buscar `https://api.github.com/repos/nexu-io/open-design/releases/latest` e `https://api.github.com/repos/nexu-io/open-design` client-side para exibir versão e stars em tempo real.

### Páginas geradas
| Rota | Origem |
|------|--------|
| `/` | `app/pages/index.astro` |
| `/blog/` | `app/pages/blog/` |
| `/blog/<slug>/` | Content Collection `blog` |
| `/skills/` | Content Collection `skills` |
| `/systems/` | Content Collection `systems` |
| `/craft/` | Content Collection `craft` |
| `/templates/` | Content Collection `templates` |
| `/og/` | `app/pages/og.astro` — Open Graph images (noindex) |
| `/rss.xml` | `app/pages/rss.xml.ts` — Feed RSS via `@astrojs/rss` |

### Open Graph
- Hero image: `app/image-assets.ts` → `heroImage`.
- Cada página: `<meta property="og:image">` com URL canônica construída via `Astro.site`.

---

## 4. APIs e Contratos

### APIs consumidas (client-side, em runtime)
| API | Endpoint | Uso |
|-----|----------|-----|
| GitHub REST API | `GET /repos/nexu-io/open-design` | Stars count |
| GitHub REST API | `GET /repos/nexu-io/open-design/releases/latest` | Versão do último release |

Ambas chamadas são best-effort; ausência de resposta não quebra o layout.

### Feeds produzidos
| Recurso | URL | Formato |
|---------|-----|---------|
| RSS Feed | `/rss.xml` | RSS 2.0 via `@astrojs/rss` |
| Sitemap | `/sitemap-index.xml` | XML Sitemap via `@astrojs/sitemap` |

---

## 5. Protocolos e Integrações

### Cloudflare Pages
- Deploy automático via Git integration.
- Variáveis de ambiente configuradas no painel Cloudflare.
- Headers e redirects definidos via Cloudflare Pages config ou `_headers`/`_redirects`.

### Sitemap
Gerado em build via `@astrojs/sitemap`. Configuração em `astro.config.ts`:
- `/og/` filtrado do sitemap.
- Prioridades: `/` = 1.0, `/blog/` = 0.9, posts = 0.8, demais = 0.5 (padrão Astro).
- `lastmod` inferido das datas dos posts de blog.

### `trailingSlash: 'always'`
Todas as rotas terminam com `/` — URLs canônicas sempre incluem trailing slash.

---

## 6. Segurança e Autenticação

- Site 100% estático e público — sem autenticação.
- Sem cookies, sessões ou dados pessoais coletados na landing page.
- Headers de segurança recomendados (CSP, X-Frame-Options) devem ser configurados via Cloudflare Pages `_headers` ou regras de transformação.
- `/og/` tem `<meta name="robots" content="noindex">` e está em `robots.txt` `Disallow` — impede indexação de superfície de screenshot.
- Script inline não executa código arbitrário de terceiros — apenas chamadas à GitHub API pública.

---

## 7. Performance e Escalabilidade

| Dimensão | Estratégia |
|---------|-----------|
| Output estático | Zero compute em runtime — servido direto da CDN Cloudflare. |
| React sem hydration | `renderToStaticMarkup()` gera HTML puro; sem bundle JS de React no cliente. |
| Script inline minimalista | Único bloco inline, sem dependências externas, sem `async` pesado. |
| Previews estáticos | Imagens `.webp`/`.png` servidas de `public/` — cacheáveis indefinidamente. |
| Sitemap com `lastmod` | Permite que crawlers priorizem apenas páginas modificadas. |
| Astro output `static` | Todo o HTML gerado em build-time; sem SSR overhead. |

Build esperado: < 60 s para catálogo completo. Nenhuma limitação de escalabilidade — Cloudflare CDN global.

---

## 8. Testes e Qualidade

| Ferramenta | Comando | Escopo |
|-----------|---------|--------|
| `astro check` | `pnpm typecheck` | Type-check de `.astro`, `.ts` e `.tsx` |
| TypeScript | Incluso no `astro check` | Tipos de Content Collections |
| Playwright | `scripts/generate-previews.ts` | Captura de previews (não é test runner) |

Não há suíte de testes unitários dedicada nesta versão. A qualidade é garantida por:
1. TypeScript strict no source.
2. `astro check` no CI antes do build.
3. Validação visual via previews gerados por Playwright.

### Critérios de aceitação de build
- `astro check` sem erros de tipo.
- `astro build` termina sem erros.
- `out/` contém `index.html`, `sitemap-index.xml`, `rss.xml`.
