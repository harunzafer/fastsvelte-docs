---
description: "FastSvelte landing page template — a standalone, SEO-optimized SvelteKit marketing site included with the kit, with built-in Open Graph, Twitter cards, and self-referencing canonicals."
keywords: "fastsvelte landing page, sveltekit landing template, saas marketing site, seo, open graph, canonical, conversion landing page"
---

# Landing Page Template

FastSvelte includes a standalone, conversion-focused **landing site** in `landing/` — a separate SvelteKit app from the dashboard (`frontend/`), so your marketing pages deploy independently of the app. It's themed (light/dark) and SEO-optimized out of the box. The steps below take you from the template to your own landing page.

## 1. Find it

The landing site lives in `landing/`. The home page (`landing/src/routes/+page.svelte`) is assembled from section components in `landing/src/lib/components/landing/`. Edit a section to change its content; reorder or remove sections in `+page.svelte`.

## 2. Set your branding

- `landing/src/lib/config.ts` — `appName` and `siteUrl`.
- `Logo.svelte` — swap in your logo (light/dark variants).
- `landing/.env` — browser-exposed vars use the `PUBLIC_*` prefix (e.g. `PUBLIC_APP_NAME`, `PUBLIC_API_BASE_URL`). Run `npx svelte-kit sync` after changing them.

## 3. Arrange your sections

`+page.svelte` composes the page from these sections, in order:

```svelte
<Topbar />
<Hero />
<FeatureList />
<!-- <Showcase /> -->
<Testimonial />
<CTA />
<FAQ />
<Pricing />
<Footer />
```

Reorder, drop, or duplicate any line to change the page. `Showcase` ships commented out — uncomment it to enable. `Newsletter` is included as a component but not placed on the page; add `<Newsletter />` (and its import) wherever you want it.

## 4. Write the copy

Each section is its own component in `landing/src/lib/components/landing/`. Edit the one you want:

- **Topbar** — navigation links and the primary CTA.
- **Hero** — headline, subheading, and the main call to action.
- **Features** — the grid of product capabilities.
- **Showcase** — a "copilot in action" demo panel (off by default).
- **Testimonial** — social proof and customer quotes.
- **Pricing** — pricing tiers (this was previously named `BundleOffer`).
- **FAQ** — common questions and answers.
- **CTA** — the closing call to action.
- **Newsletter** — an email-capture signup (UI only — wire the `<form>` to your email provider; off by default).
- **Footer** — links, legal, and social.

## 5. SEO is already handled

Every route renders the `Seo` component (`landing/src/lib/components/Seo.svelte`), so titles, descriptions, self-referencing canonicals, and Open Graph / Twitter cards are set for you — no per-page URLs to hardcode. See **[SEO](seo.md)** to customize it.

## 6. Ship it

See `landing/README.md` for commands (`npm run dev`, `npm run build`). The landing site deploys independently — see [Deployment](../deployment/index.md).

## Next steps

- **[SEO](seo.md)** — sitemaps, robots, and metadata strategy.
- **[Deployment](../deployment/index.md)** — hosting the landing site.
