---
description: "FastSvelte landing page template — a standalone, SEO-optimized SvelteKit marketing site included with the kit, with built-in Open Graph, Twitter cards, and self-referencing canonicals."
keywords: "fastsvelte landing page, sveltekit landing template, saas marketing site, seo, open graph, canonical, conversion landing page"
---

# Landing Page Template

FastSvelte includes a standalone, conversion-focused **landing site** in `landing/` — a separate SvelteKit app from the dashboard (`frontend/`), so your marketing pages deploy independently of the app. It's themed (light/dark) and SEO-optimized out of the box.

## What's included

The landing page (`landing/src/routes/+page.svelte`) is assembled from section components in `landing/src/lib/components/landing/`:

- `Topbar`, `Hero`, `Features`, `Testimonial`, `CTA`, `FAQ`, `BundleOffer`, `Footer`
- `Showcase` is included but commented out — enable it for a screenshot section.

Branding and theming live in `Logo.svelte`, `ThemeToggle.svelte`, and `landing/src/lib/config.ts` (`appName`, `siteUrl`).

## SEO built in

Every route renders the `Seo` component (`landing/src/lib/components/Seo.svelte`), which emits:

- `<title>` and `<meta name="description">`.
- A **self-referencing canonical** auto-derived from the current path — each route points at itself, with no per-page URL to hardcode (override via the `canonical` prop only if needed).
- **Open Graph** tags (`og:type`, `og:url`, `og:title`, `og:description`, `og:site_name`, optional `og:image`).
- **Twitter** card tags (`summary_large_image`, title, description, optional image).

Usage:

```svelte
<Seo
  title="Your Product — tagline"
  description="One-sentence value proposition."
  image="https://yourdomain.com/og-card.png"
/>
```

`ogTitle` / `ogDescription` override the social text; `image` is omitted from output when unset. The canonical and `og:site_name` come from `config.siteUrl` / `config.appName`. See **[SEO](seo.md)** for the full guidance.

## Customizing

1. Edit the section components in `landing/src/lib/components/landing/` (copy, order, which sections render in `+page.svelte`).
2. Set branding in `landing/src/lib/config.ts` and swap `Logo.svelte`.
3. Configure browser-exposed vars in `landing/.env` (`PUBLIC_*` prefix, e.g. `PUBLIC_APP_NAME`, `PUBLIC_API_BASE_URL`). Run `npx svelte-kit sync` after changing them.

## Running

See `landing/README.md` for commands (`npm run dev`, `npm run build`). The landing site deploys independently — see [Deployment](../deployment/index.md).

## Next steps

- **[SEO](seo.md)** — sitemaps, robots, and metadata strategy.
- **[Deployment](../deployment/index.md)** — hosting the landing site.
