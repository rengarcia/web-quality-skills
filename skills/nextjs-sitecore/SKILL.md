---
name: nextjs-sitecore
description: Optimize Next.js + Sitecore JSS/XM Cloud projects for performance. Use when asked about "Sitecore JSS", "XM Cloud", "Sitecore Next.js", "Experience Edge", "Sitecore performance", "JSS optimization", "Sitecore Content SDK", or "Sitecore layout optimization".
license: MIT
metadata:
  author: web-quality-skills
  version: "1.0"
---

# Next.js + Sitecore JSS performance optimization

Specialized performance optimization for Next.js projects using Sitecore JSS, XM Cloud, and Experience Edge. Complements the generic `performance` skill with Sitecore-specific patterns.

## When to use this skill

Use this skill when optimizing a Next.js project that uses:
- Sitecore JSS SDK (`@sitecore-jss/sitecore-jss-nextjs`)
- Sitecore XM Cloud or XP with headless services
- Experience Edge for content delivery
- Sitecore Content Hub for media
- Sitecore Search SDK or Sitecore Personalize

For generic Next.js or React optimization, use the `performance` skill instead.

### Sitecore Content SDK vs JSS

For **new XM Cloud projects**, consider the Sitecore Content SDK (v1.0+) over JSS. It provides smaller bundles via explicit component maps (better tree-shaking), a unified `SitecoreClient` API (fewer network calls), and App Router support. Existing JSS projects should evaluate migration for long-term benefits. Non-XM Cloud projects (XP/XM) should stay with JSS.

## Architecture performance

### Pages Router vs App Router with Sitecore

Sitecore JSS primarily targets Pages Router. If using Pages Router:
- Leverage `getStaticProps`/`getStaticPaths` for ISR
- Use `_app.tsx` sparingly — heavy providers here block every route
- Keep `_document.tsx` lean (no runtime JS)

If migrating to App Router:
- Move Sitecore context providers to a root layout `layout.tsx`
- Use Server Components for static Sitecore content
- Keep `'use client'` boundaries as low as possible in the component tree

### Component factory optimization

Use dynamic imports for heavy/below-fold components in the component factory:

```tsx
import dynamic from 'next/dynamic';
import HeroBanner from './HeroBanner'; // keep above-fold static

const components = new Map();
components.set('HeroBanner', HeroBanner);
components.set('VideoPlayer', dynamic(() => import('./VideoPlayer')));
components.set('InteractiveMap', dynamic(() => import('./InteractiveMap'), {
  loading: () => <div className="map-skeleton" />,
  ssr: false,
}));
```

### Placeholder nesting impact

- Limit placeholder nesting to 3-4 levels maximum
- Use composition over deep nesting for page layouts
- Avoid wrapping placeholders in unnecessary client components

## ISR & data fetching

Use `fallback: 'blocking'` for Sitecore pages — new pages render server-side on first request and are cached. Use the `STATIC_PATHS` env var to control which pages are pre-built at deploy time:

```bash
STATIC_PATHS=/,/about,/products/*  # Only pre-render critical pages
```

### Revalidation tuning

```tsx
export const getStaticProps: GetStaticProps = async (context) => {
  const props = await sitecorePagePropsFactory.create(context);
  if (props.notFound) {
    return { notFound: true, revalidate: 300 }; // 5 min — Edge flakiness recovery
  }
  return { props, revalidate: 1800 }; // 30 min for content pages
};
```

- **Content pages:** `revalidate: 1800` (30 min) balances freshness with Edge load
- **404 pages:** `revalidate: 300` (5 min) to recover from transient Edge failures
- **High-traffic pages:** Consider `revalidate: 3600` (1 hr) or longer
- Never use `revalidate: 1` or very low values — this overwhelms Experience Edge

### GraphQL query optimization

Fetch only needed fields when querying Experience Edge directly:

```graphql
# bad: fetching entire item with ...AllFields
# good: fetch only what the component needs
query { item(path: "/sitecore/content/home") {
  field(name: "title") { value }
  field(name: "description") { value }
}}
```

See [references/ISR-CACHING.md](references/ISR-CACHING.md) for deep-dive on ISR patterns.

## Layout payload optimization

The Sitecore Layout Service returns the full layout with all fields for all components. A typical page produces 200-500 KB of layout JSON serialized as `__NEXT_DATA__`, impacting page weight, hydration time, and memory usage.

### Component-level field pruning

Use a layout optimizer pattern to trim fields per component:

```tsx
import { defineOptimizer } from 'lib/layout-optimizer';
import { HeroBannerFields } from './HeroBanner.types';

export const heroBannerOptimizer = defineOptimizer<HeroBannerFields>(
  'HeroBanner',
  (component) => ({
    title: component.fields.title,
    subtitle: component.fields.subtitle,
    image: component.fields.image,
    // Drops all other fields from the payload
  })
);
```

### Removing system fields

Strip Sitecore system fields never used in rendering (`__Created`, `__Updated`, `__Revision`, `__Sortorder`, `__Display name`, `__Hidden`, `__Read Only`). Walk the component tree in `getStaticProps` and delete these fields.

See [references/LAYOUT-OPTIMIZATION.md](references/LAYOUT-OPTIMIZATION.md) for the full optimizer pattern with registry and type-safe helpers.

## Bundle optimization

### Split chunks for Sitecore SDKs

Sitecore SDKs are large and shared across pages. Configure webpack `splitChunks.cacheGroups`:

```js
// next.config.js — webpack config
sitecoreSearch: {
  test: /[\\/]node_modules[\\/]@sitecore-search/, name: 'sitecore-search',
  chunks: 'all', priority: 30,
},
sitecoreFeaas: {
  test: /[\\/]node_modules[\\/]@sitecore-feaas/, name: 'sitecore-feaas',
  chunks: 'all', priority: 30,
},
sitecoreCloudSdk: {
  test: /[\\/]node_modules[\\/]@sitecore-cloudsdk/, name: 'sitecore-cloudsdk',
  chunks: 'all', priority: 30,
},
heavyUi: {
  test: /[\\/]node_modules[\\/](swiper|three|@chakra-ui|@radix-ui)/,
  name: 'heavy-ui', chunks: 'all', priority: 20,
},
```

### Dynamic imports for heavy components

```tsx
const SearchResults = dynamic(() => import('components/SearchResults'), { ssr: false });
const ThreeDViewer = dynamic(() => import('components/ThreeDViewer'), {
  ssr: false, loading: () => <Skeleton height={400} />,
});
```

### Tree-shaking & analysis

```tsx
// bad: import everything
import { Text, Image, RichText, Link, Placeholder } from '@sitecore-jss/sitecore-jss-nextjs';
// good: import from subpaths if supported
import { Text } from '@sitecore-jss/sitecore-jss-nextjs/components';
```

Run bundle analyzer with `ANALYZE=true npm run build` (requires `@next/bundle-analyzer`).

## Image optimization

### `next/image` with Sitecore media

Configure `remotePatterns` for all Sitecore image domains:

```js
// next.config.js
images: {
  remotePatterns: [
    { protocol: 'https', hostname: 'edge.sitecorecloud.io' },
    { protocol: 'https', hostname: '*.sitecorecontenthub.cloud' },
    { protocol: 'https', hostname: 'xmc-*.sitecorecloud.io' },
  ],
  deviceSizes: [640, 750, 828, 1080, 1200, 1440, 1920],
}
```

### LCP images

For hero banners and above-fold images, always set `priority` and `fetchPriority="high"`:

```tsx
<ImageField field={fields.heroImage} priority={true} sizes="100vw" fetchPriority="high" />
```

### Sitecore media handler parameters

Use server-side resizing via query params for non-Edge deployments:
- `mw` — max width, `mh` — max height, `as` — allow stretch (0 or 1)
- Example: `/-/media/image.jpg?mw=800&mh=600&as=1`

## Middleware performance

### Plugin chain architecture

Order matters — put fast checks first:

```tsx
export const plugins: MiddlewarePlugin[] = [
  redirectsMiddleware,       // 1. Fast map lookup
  personalizeMiddleware,     // 2. May call Edge
  multiSiteMiddleware,       // 3. Multi-site resolution
];
```

### Key guidelines

- Only compute CSP nonces for HTML page requests (check `accept` header)
- Prefer `next.config.js` `redirects()` for static patterns over middleware
- Cache redirect maps in memory — don't fetch on every request
- Skip personalization for bot/crawler user agents
- Use `matcher` config to limit middleware to specific paths
- **On Vercel:** Replace GraphQL redirect lookups with Edge Config for ~9x faster middleware (130ms to 15ms). See [references/VERCEL-DEPLOYMENT.md](references/VERCEL-DEPLOYMENT.md)

## Font loading

Inject font classes through Sitecore's layout system:

```tsx
import { Inter } from 'next/font/google';
const inter = Inter({ subsets: ['latin'], display: 'swap', variable: '--font-inter' });

// In _app.tsx, wrap SitecoreContext with font variable class
<div className={inter.variable}>
  <SitecoreContext ...><Component {...pageProps} /></SitecoreContext>
</div>
```

- Use `display: 'swap'` to prevent invisible text
- Subset fonts to needed character sets
- Limit font families to 2-3 maximum
- Use variable fonts when available

## Caching strategy

### Recommended cache durations

| Asset type | Cache-Control | Rationale |
|-----------|---------------|-----------|
| `/_next/static/*` | `immutable, max-age=31536000` | Content-hashed, safe forever |
| Sitecore media | `max-age=86400` | URLs may change on publish |
| SDK files (FEaaS, CDP) | `max-age=2419200` (28 days) | Versioned externally |
| API routes | `no-store` or `max-age=60` | Dynamic data |
| Editing API (`/api/editing/*`) | `no-store` | Must never be cached |

### React Query with Sitecore

```tsx
const queryClient = new QueryClient({
  defaultOptions: { queries: {
    staleTime: Infinity,           // Layout data doesn't change client-side
    gcTime: 1000 * 60 * 30,       // 30 min cache
    refetchOnWindowFocus: false,
  }},
});
```

## Component performance patterns

### `withDatasourceCheck` guard

Prevents rendering components with empty datasources — avoids unnecessary DOM and hydration:

```tsx
export default withDatasourceCheck()(HeroBanner);
```

### Lazy loading below-fold placeholders

Use an `IntersectionObserver` wrapper around `<Placeholder>` for below-fold content. Render a skeleton with `minHeight` until the placeholder enters the viewport (use `rootMargin: '200px'` for preloading).

### Avoiding re-renders

- Memoize components that consume `SitecoreContext` if they only need static data
- Split frequently-changing state from Sitecore layout context
- Use `React.memo` on components inside `Placeholder` to prevent cascade re-renders

### BYOC & Search SDK

- Lazy load BYOC wrappers with `dynamic(() => import(...), { ssr: false })`
- Set explicit `width`/`height` on BYOC containers to prevent CLS
- The Search SDK is ~150 KB — always lazy load:

```tsx
const SearchWidget = dynamic(
  () => import('@sitecore-search/react').then(mod => mod.SearchWidget),
  { ssr: false }
);
```

## Security & best practices

### CSP with Sitecore

Sitecore inline scripts require nonce-based CSP. Include `https://feaas.blob.core.windows.net` in `script-src`, and Sitecore Edge/Content Hub domains in `img-src` and `connect-src`.

### Production hardening

```js
// next.config.js
module.exports = {
  poweredByHeader: false,
  productionBrowserSourceMaps: false,
};
```

- Never expose `SITECORE_API_KEY` to the client bundle
- Use `NEXT_PUBLIC_` prefix only for truly public keys (like CDP client key)
- Keep Experience Edge API keys server-side only

## Vercel deployment

When deploying to Vercel, leverage these platform-specific optimizations:

### Fluid Compute

Reuses serverless function instances across requests, eliminating cold starts. ISR revalidation and GraphQL calls to Experience Edge benefit from warm instances that maintain TCP connections. Enable in Project Settings > Functions (zero code changes). Sitecore customers report **45%+ compute savings**.

### Skew Protection

Prevents hydration mismatches during deployments when `__NEXT_DATA__` shape changes (e.g., component factory updates). Enable in Project Settings > General > Skew Protection. No code changes required.

### Edge Config for redirects

Replace the JSS redirect middleware's per-request GraphQL call to Experience Edge with Vercel Edge Config lookups (~15ms vs ~130ms):

```typescript
import { createClient } from '@vercel/edge-config';
const edgeConfig = createClient(process.env.EDGE_CONFIG_REDIRECTS);
const redirect = await edgeConfig.get(pathname); // sub-millisecond read
```

Sync redirects from Experience Edge to Edge Config via a Sitecore publish webhook. For 65,000+ redirects, use a Bloom filter for probabilistic pre-filtering.

### 3-tier cache headers

Vercel supports tiered cache control — use `Vercel-CDN-Cache-Control` for Sitecore API proxy routes to cache at the edge without polluting browser caches:

| Header | Scope | Visible to browser |
|--------|-------|-------------------|
| `Cache-Control` | Browser + CDN | Yes |
| `CDN-Cache-Control` | All CDNs | No |
| `Vercel-CDN-Cache-Control` | Vercel CDN only | No |

### Speed Insights & Analytics

```tsx
// _app.tsx — add after SitecoreContext
import { Analytics } from '@vercel/analytics/next';
import { SpeedInsights } from '@vercel/speed-insights/next';
// Filter editing paths: beforeSend={(e) => e.url.includes('/api/editing') ? null : e}
```

Segment by route pattern to identify which Sitecore templates cause performance regressions.

### Geolocation-based locale routing

Use Vercel's free `x-vercel-ip-country` header for zero-cost locale routing at the edge, before Sitecore Personalize middleware runs:

```typescript
const country = req.headers.get('x-vercel-ip-country') || 'US';
const targetLocale = COUNTRY_LOCALE_MAP[country] || 'en';
```

### Multi-site architecture

Use **one Vercel project per Sitecore application head** for isolated bundles, independent deployments, and per-site configuration (firewall, env vars, domains).

### Observability

Use `@vercel/otel` for OpenTelemetry tracing of Experience Edge calls. Add custom spans for layout fetches with attributes like `sitecore.path`, `sitecore.locale`, and `sitecore.layout.payload_bytes`. Export via Vercel Drains to Datadog/Grafana.

See [references/VERCEL-DEPLOYMENT.md](references/VERCEL-DEPLOYMENT.md) for detailed patterns.

## Performance checklist

### Build & bundle
- [ ] Heavy Sitecore SDKs split into separate chunks
- [ ] Component factory uses dynamic imports for below-fold components
- [ ] Bundle analyzer shows no unexpected large modules
- [ ] Tree-shaking verified for Sitecore JSS imports

### Data fetching
- [ ] ISR enabled with appropriate `revalidate` values
- [ ] 404 pages use short revalidation (300s) for Edge recovery
- [ ] GraphQL queries fetch only needed fields
- [ ] Layout payload is optimized (system fields stripped)

### Images
- [ ] `next/image` configured with Sitecore `remotePatterns`
- [ ] LCP image has `priority` prop
- [ ] Appropriate `sizes` attribute on responsive images

### Runtime
- [ ] Below-fold placeholders lazy loaded
- [ ] `withDatasourceCheck` used on components with datasources
- [ ] Sitecore Search SDK loaded dynamically
- [ ] Font loading uses `next/font` with `display: 'swap'`

### Caching
- [ ] Static assets have immutable cache headers
- [ ] React Query configured with `staleTime: Infinity` for layout data
- [ ] Middleware skips non-HTML requests for expensive operations

### Vercel deployment
- [ ] Fluid Compute enabled (Project Settings > Functions)
- [ ] Skew Protection enabled
- [ ] Redirects moved to Edge Config (if > 50 redirects)
- [ ] Speed Insights + Analytics installed
- [ ] OpenTelemetry configured for Experience Edge tracing

### Security
- [ ] CSP headers configured with nonces for Sitecore scripts
- [ ] `poweredByHeader: false` set
- [ ] Source maps disabled in production
- [ ] API keys not exposed to client
- [ ] Vercel WAF rules protect `/api/editing/*` endpoints
