# ISR & caching strategies for Sitecore Next.js

## Selective static generation

### The `STATIC_PATHS` pattern

Large Sitecore sites may have thousands of pages. Building all of them at deploy time is slow and often unnecessary. Use the `STATIC_PATHS` environment variable to control which pages are pre-rendered:

```bash
# Only pre-render critical pages
STATIC_PATHS=/,/about,/products,/contact

# Use glob patterns for page groups
STATIC_PATHS=/,/products/*,/blog/featured-*

# Disable static generation entirely (pure ISR)
STATIC_PATHS=none
```

### Implementation

```tsx
// lib/sitemap-fetcher/plugins/static-paths-filter.ts
export class StaticPathsFilterPlugin {
  exec(paths: StaticPath[]): StaticPath[] {
    const staticPaths = process.env.STATIC_PATHS;

    if (!staticPaths || staticPaths === 'none') {
      return []; // All pages served via ISR on first request
    }

    const patterns = staticPaths.split(',').map((p) => p.trim());

    return paths.filter((path) => {
      const fullPath = '/' + path.params.path.join('/');
      return patterns.some((pattern) => matchPath(fullPath, pattern));
    });
  }
}

function matchPath(path: string, pattern: string): boolean {
  if (pattern.endsWith('*')) {
    return path.startsWith(pattern.slice(0, -1));
  }
  return path === pattern;
}
```

### Locale-aware static generation

For multi-language sites, generate static pages only for the primary language and let ISR handle the rest:

```tsx
export class LocaleFilterPlugin {
  exec(paths: StaticPath[]): StaticPath[] {
    const primaryLocale = process.env.PRIMARY_LOCALE || 'en';

    // Only statically generate pages in the primary locale
    return paths.filter((path) => path.params.locale === primaryLocale);
  }
}
```

This approach reduces build time proportionally to the number of locales.

## Multi-site path filtering

Sitecore XM Cloud supports multi-site with a `_site_` prefix convention:

```tsx
// Pages are stored as: /sitecore/content/_site_brand-a/home/about
// But served as: /about on brand-a.com

export class MultiSitePathFilter {
  exec(paths: StaticPath[], siteInfo: SiteInfo): StaticPath[] {
    const sitePrefix = `_site_${siteInfo.name}`;

    return paths.filter((path) => {
      const firstSegment = path.params.path[0];
      return firstSegment === sitePrefix || !firstSegment?.startsWith('_site_');
    });
  }
}
```

## Transient upstream failure detection

Experience Edge can return temporary errors during content publishing or under load. Treating these as real 404s causes pages to disappear.

Detect transient failures by health-checking Experience Edge (`/healthz` with a 2s timeout). If Edge is unhealthy or unreachable, use `revalidate: 60` (1 min); for valid 404s, use `revalidate: 300` (5 min).

### Distinguishing error types

| Scenario | Recommended `revalidate` | Rationale |
|----------|-------------------------|-----------|
| Edge returns valid 404 | 300s (5 min) | Page may be published soon |
| Edge returns 5xx | 60s (1 min) | Transient — retry quickly |
| Network timeout | 60s (1 min) | Infrastructure issue |
| Edge returns valid content | 1800s (30 min) | Normal content caching |
| High-traffic page | 3600s (1 hr) | Reduce Edge load |

## Revalidation timing strategies

### Content type-based revalidation

```tsx
function getRevalidateTime(layoutData: LayoutServiceData): number {
  const templateName = layoutData?.sitecore?.route?.templateName;

  switch (templateName) {
    case 'News Article':
      return 600;   // 10 min — frequently updated
    case 'Product Page':
      return 1800;  // 30 min — moderate updates
    case 'Legal Page':
      return 7200;  // 2 hr — rarely changes
    case 'Homepage':
      return 900;   // 15 min — balance freshness with traffic
    default:
      return 1800;  // 30 min default
  }
}
```

### On-demand revalidation (Pages Router)

For content that must update immediately on publish, use Next.js on-demand revalidation with a Sitecore webhook:

```tsx
// pages/api/revalidate.ts
export default async function handler(req: NextApiRequest, res: NextApiResponse) {
  if (req.headers['x-revalidation-secret'] !== process.env.REVALIDATION_SECRET) {
    return res.status(401).json({ message: 'Invalid secret' });
  }
  const { pages, locale } = req.body;
  const siteName = process.env.SITECORE_SITE_NAME;
  for (const page of pages) {
    await res.revalidate(`/${locale}/_site_${siteName}${page}`);
  }
  return res.json({ revalidated: true, now: Date.now() });
}
```

### Tag-based revalidation (App Router)

With App Router, use `next: { tags }` on fetch calls and `revalidateTag()` for surgical invalidation:

```typescript
// Fetch with cache tags
const navData = await fetch(EDGE_URL, {
  next: { tags: ['navigation'], revalidate: 3600 },
});

// API route triggered by Sitecore webhook
import { revalidateTag } from 'next/cache';
export async function POST(req: Request) {
  const { tags } = await req.json(); // e.g., ['navigation', 'product-123']
  for (const tag of tags) revalidateTag(tag);
  return Response.json({ revalidated: true });
}
```

### Sitecore webhook setup

Create a Webhook Event Handler in Sitecore under `/system/Webhooks`, selecting the `publish:end` event. Point to your revalidation endpoint.

**Key caveat:** Experience Edge can take up to 15 minutes to reflect published content. Use the Experience Edge webhook (fires after Edge is updated) rather than the CM `publish:end` webhook to avoid revalidating against stale Edge data.

## Monitoring ISR cache performance

Key metrics to track: **cache hit rate**, **revalidation frequency**, **stale page age**, and **Edge error rate**.

Log structured JSON in `getStaticProps` with `event: 'isr_revalidation'`, path, locale, duration, and `hasLayout` boolean. Query in your observability platform (Vercel Logs/Drains, Datadog, etc.).
