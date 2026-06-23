---
description: "Add SEO metadata to your FastSvelte landing pages - use the bundled Seo component for titles, meta descriptions, self-referencing canonicals, and Open Graph / Twitter cards on every route."
keywords: "fastsvelte seo, sveltekit meta tags, canonical url, open graph, twitter cards, meta description, svelte head, sveltekit seo, page metadata"
---

# SEO & Page Metadata

[FastSvelte](https://fastsvelte.dev)'s landing ships with a reusable `Seo` component
(`src/lib/components/Seo.svelte`) that emits a correct, per-page `<head>`: a single title and
meta description, a **self-referencing canonical**, and Open Graph / Twitter card tags. The
homepage is already wired up — you only need this guide when you **add new routes**.

## Using the component

Add one `<Seo>` per route, passing that page's title and description:

```svelte
<script lang="ts">
	import Seo from '$lib/components/Seo.svelte';

	let { data } = $props();
</script>

<Seo title={data.title} description={data.description} />
```

A common pattern is to supply the values from the route's `+page.ts` load, as the homepage does:

```ts
// src/routes/pricing/+page.ts
import type { PageLoad } from './$types';

export const load: PageLoad = async ({ parent }) => {
	const { appName } = await parent();
	return {
		title: `${appName} - Pricing`,
		description: 'Simple, transparent pricing for every stage of your SaaS.'
	};
};
```

### Props

| Prop            | Required | Description                                                                    |
| --------------- | -------- | ------------------------------------------------------------------------------ |
| `title`         | yes      | Page `<title>` and default social title.                                       |
| `description`   | yes      | `<meta name="description">` and default social description.                     |
| `ogTitle`       | no       | Override the social title (defaults to `title`).                                |
| `ogDescription` | no       | Override the social description (defaults to `description`).                    |
| `image`         | no       | Absolute URL to a social-card image. When omitted, no `og:image` tag is emitted. |
| `canonical`     | no       | Override the auto-derived canonical (rarely needed).                            |

## Why one component per page (and not the layout)

It's tempting to put a default `<meta name="description">` or canonical in `+layout.svelte` so
every page inherits it. **Don't.** SvelteKit deduplicates `<title>`, but it does **not** dedupe
`<meta>` or `<link>` tags — they accumulate. Two things go wrong:

- **Duplicate descriptions.** Any page that sets its *own* description on top of the layout default
  ships *two* `<meta name="description">` tags. (A page that relies solely on the default ships one
  — the trap only springs once a page tries to override it, which most will.)
- **Leaked canonical.** A hardcoded homepage canonical in the layout is inherited by every page that
  doesn't set its own. That's a *consolidation hint* telling search engines those pages are
  duplicates of the homepage, so their ranking signals get folded into it and they don't rank as
  themselves. (It's a hint, not a `noindex` — search engines may ignore it — but you don't want to
  be fighting it.)

The `Seo` component avoids this by emitting exactly one description and a canonical derived from
the **current path**, so every route points at itself automatically with nothing to hardcode:

```
/            → https://yourdomain.com/
/pricing     → https://yourdomain.com/pricing
```

## Configure your domain

Canonical and `og:url` tags are built from `config.siteUrl`, which reads the `PUBLIC_SITE_URL`
environment variable (see `src/lib/config.ts`). Set this to your own domain so the tags don't point
at `fastsvelte.dev`:

```bash
# .env
PUBLIC_SITE_URL=https://yourdomain.com
```

`og:site_name` uses `config.appName` (`PUBLIC_APP_NAME`), so set that too if you've renamed your app.

## Social card images

The component omits `og:image` / `twitter:image` unless you pass an `image` prop. To enable rich
social previews, add a `1200×630` PNG/JPG to `static/` and pass its absolute URL:

```svelte
<Seo
	title={data.title}
	description={data.description}
	image={`${config.siteUrl}/images/og-image.png`}
/>
```

## Verify

After `npm run build` (and `npm run preview`), inspect the homepage source. You should see exactly
**one** `<meta name="description">` and **one** self-referencing `<link rel="canonical">`:

```bash
curl -s http://localhost:4173/ | grep -E 'name="description"|rel="canonical"'
```
