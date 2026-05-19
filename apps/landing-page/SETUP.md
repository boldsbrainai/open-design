# Setup Guide — Landing Page

## Pré-requisitos

| Requisito | Versão | Observação |
|-----------|--------|-----------|
| Node.js | `~24` | Obrigatório — engine declarado no `package.json` |
| pnpm | `10.33.2` | Via Corepack (`corepack enable`) |
| Playwright | `^1.59.1` | Somente para geração de previews — não é necessário para `dev`/`build` |

## Variáveis de Ambiente

| Variável | Tipo | Padrão | Obrigatório | Descrição |
|----------|------|--------|-------------|-----------|
| `OD_LANDING_SITE` | `string` | `https://open-design.ai` | Não | URL canônica do site. Usado por `Astro.site`, pelo `@astrojs/sitemap` e pelas tags `og:url` / `canonical`. Deve ser definido em deployments de preview para que URLs do sitemap e OG não apontem para o domínio de produção. |

> Nenhuma outra variável de ambiente é necessária para build/dev local.  
> A GitHub API é consumida client-side no navegador sem autenticação — nenhuma chave é necessária.

## Instalação

```bash
# A partir da raiz do monorepo
pnpm install
```

As dependências do workspace são ligadas automaticamente pelo pnpm. Não é necessário instalar dependências separadamente dentro de `apps/landing-page/`.

## Configuração Inicial

A aplicação lê os diretórios de conteúdo do monorepo **em build-time** via Astro Content Collections:

| Collection | Diretório |
|------------|-----------|
| `skills` | `<repo-root>/skills/*/SKILL.md` |
| `systems` | `<repo-root>/design-systems/*/DESIGN.md` |
| `craft` | `<repo-root>/craft/*.md` |
| `templates` | `<repo-root>/templates/live-artifacts/*/README.md` |
| `blog` | `apps/landing-page/app/content/blog/*.md` |

Nenhuma configuração manual é necessária: os globs são resolvidos automaticamente pelo Astro.

### Geração de Previews (opcional)

Previews são imagens estáticas geradas offline com Playwright. Se não estiverem presentes em `public/previews/`, as páginas de catálogo simplesmente não exibirão thumbnails.

```bash
# Instala browsers do Playwright (apenas uma vez por máquina)
npx playwright install chromium

# Gera os previews
pnpm --filter @open-design/landing-page previews
```

As imagens são salvas em `apps/landing-page/public/previews/<bucket>/<slug>.{png,webp}`. Commitar após geração.

## Inicialização

### Modo de desenvolvimento

```bash
pnpm --filter @open-design/landing-page dev
```

Servidor de dev em: `http://127.0.0.1:17574`

> O hot reload do Astro reflete mudanças em arquivos `.astro`, `.ts`, `.tsx` e `.css` automaticamente.

### Build de produção

```bash
pnpm --filter @open-design/landing-page build
```

Equivalente ao comando no `package.json`: `astro check && astro build`.  
Saída gerada em `apps/landing-page/out/`.

### Preview do build local

```bash
pnpm --filter @open-design/landing-page preview
```

Serve o conteúdo de `out/` em: `http://127.0.0.1:17574`

## Comandos Disponíveis

| Comando | Descrição |
|---------|-----------|
| `pnpm --filter @open-design/landing-page dev` | Servidor de desenvolvimento com HMR |
| `pnpm --filter @open-design/landing-page build` | Type-check + build estático para `out/` |
| `pnpm --filter @open-design/landing-page preview` | Serve o build de `out/` localmente |
| `pnpm --filter @open-design/landing-page previews` | Gera thumbnails de skills/templates via Playwright |
| `pnpm --filter @open-design/landing-page typecheck` | Executa apenas `astro check` (sem build) |

## Verificação

Após o build, confirme que os seguintes artefatos existem em `apps/landing-page/out/`:

```
out/
  index.html          ← Homepage
  sitemap-index.xml   ← Sitemap principal
  rss.xml             ← Feed RSS do blog
  blog/               ← Posts do blog
  skills/             ← Catálogo de skills
  systems/            ← Catálogo de design systems
  craft/              ← Craft rules
  templates/          ← Catálogo de templates
```

### Typecheck

```bash
pnpm --filter @open-design/landing-page typecheck
# Deve terminar com: "Found 0 errors."
```

## Troubleshooting

### `astro check` falha com erros de tipo nas Content Collections
- Verifique se os diretórios `skills/`, `design-systems/`, `craft/` e `templates/` existem na raiz do repositório.
- Execute `pnpm install` na raiz para garantir que links do workspace estão corretos.

### `pnpm dev` mostra catálogo vazio
- O Astro lê os arquivos em build/start-time. Reinicie o servidor após adicionar novos `SKILL.md` ou `DESIGN.md`.

### Previews não aparecem nas páginas de catálogo
- Execute `pnpm --filter @open-design/landing-page previews` para gerar os thumbnails.
- Verifique se `public/previews/<bucket>/` contém os arquivos `.png` ou `.webp`.

### Build falha com `LANGFUSE_*` ou outras variáveis não encontradas
- A landing page não usa essas variáveis. Verifique se não há arquivo `.env` com configurações conflitantes.

### `OD_LANDING_SITE` errado em preview deployment
- Configure a variável de ambiente no painel do Cloudflare Pages para o ambiente de Preview com a URL correta do deployment.
