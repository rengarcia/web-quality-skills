# Vercel deployment optimization for Sitecore Next.js

## Edge Config for redirects

Sitecore's default redirect middleware makes a GraphQL call to Experience Edge on every request (~130ms). Replacing this with Vercel Edge Config drops latency to ~15ms.

### Architecture

On Sitecore publish, a webhook syncs redirect data to Edge Config via the Vercel API. Middleware reads from Edge Config instead of calling Experience Edge.

```bash
npm install @vercel/edge-config
```

### Middleware implementation

```typescript
import { NextResponse } from 'next/server';
import type { NextRequest } from 'next/server';
import { createClient } from '@vercel/edge-config';

export default async function middleware(req: NextRequest) {
  const pathname = req.nextUrl.pathname.replace(/\/$/, '').toLowerCase();
  const store = createClient(process.env.EDGE_CONFIG_REDIRECTS);
  const redirect = await store.get(pathname);

  if (redirect && pathname !== redirect.destination) {
    const statusCode = redirect.permanent ? 308 : 307;
    return NextResponse.redirect(new URL(redirect.destination, req.url), statusCode);
  }
  return NextResponse.next();
}
```

### Edge Config JSON structure

```json
{
  "/old-page": { "destination": "/new-page", "permanent": true },
  "/old-page/item-1": { "destination": "/new-page/item-1", "permanent": false }
}
```

### Scaling to 65,000+ redirects

Use a Bloom filter in Edge Config for sub-millisecond probabilistic lookups. Only fetch the full redirect entry when the Bloom filter indicates a probable match.

### Sync webhook

Create a Sitecore webhook handler that fetches redirects from Experience Edge and PATCHes them to Edge Config via the Vercel REST API on `publish:end` events.

## 3-tier cache headers

Vercel processes three levels of cache control with different precedence:

```typescript
// pages/api/sitecore-proxy.ts
export default async function handler(req: NextApiRequest, res: NextApiResponse) {
  const data = await fetchFromExperienceEdge(req.query);

  res.setHeader('Cache-Control', 'public, max-age=60');                    // Browser: 1 min
  res.setHeader('CDN-Cache-Control', 'public, max-age=300');               // All CDNs: 5 min
  res.setHeader('Vercel-CDN-Cache-Control',
    'public, max-age=3600, stale-while-revalidate=86400');                  // Vercel: 1 hr + SWR

  res.json(data);
}
```

Use `Vercel-CDN-Cache-Control` for Sitecore API proxy routes to cache at the edge without affecting browser caches. This prevents stale browser caches from serving outdated content after publishes.

**Caveat:** Cache headers do NOT work if `Authorization` header or `set-cookie` is present in the response. This matters for Sitecore preview/editing endpoints.

## Vercel WAF & bot protection

Use `vercel.json` firewall rules to IP-restrict `/api/editing` endpoints and rate-limit Edge proxy routes. Use `@vercel/functions` `verifyBotId()` to validate form submissions before forwarding to Sitecore — invisible, AI-driven bot detection with zero user friction.

## OpenTelemetry instrumentation

```bash
npm install @opentelemetry/api @vercel/otel
```

```typescript
// instrumentation.ts
import { registerOTel } from '@vercel/otel';

export function register() {
  registerOTel({
    serviceName: 'sitecore-nextjs-head',
    instrumentationConfig: {
      fetch: {
        propagateContextUrls: ['edge.sitecorecloud.io', '*.sitecorecloud.io'],
        dontPropagateContextUrls: ['feaas.blob.core.windows.net'],
        ignoreUrls: ['vercel.live', '_next/static'],
      },
    },
  });
}
```

### Custom spans for Sitecore operations

```typescript
import { trace } from '@opentelemetry/api';
const tracer = trace.getTracer('sitecore-operations');

export async function fetchLayoutWithTracing(path: string, locale: string) {
  const span = tracer.startSpan('sitecore.layout.fetch');
  try {
    span.setAttributes({ 'sitecore.path': path, 'sitecore.locale': locale });
    const result = await fetchLayout(path, locale);
    span.setAttributes({
      'sitecore.layout.component_count': countComponents(result),
      'sitecore.layout.payload_bytes': JSON.stringify(result).length,
    });
    return result;
  } catch (error) {
    span.recordException(error as Error);
    throw error;
  } finally {
    span.end();
  }
}
```

Export traces via Vercel Drains (Pro/Enterprise) to Datadog APM, Honeycomb, Grafana Tempo, or New Relic.

## Geolocation-based locale routing

Use Vercel's free geolocation headers instead of calling Sitecore Personalize for locale detection:

```typescript
// middleware/plugins/geoLocalePlugin.ts
const COUNTRY_LOCALE_MAP: Record<string, string> = {
  IN: 'en-IN', DE: 'de-DE', FR: 'fr-FR', JP: 'ja-JP',
};

export function geoLocalePlugin(req: NextRequest): NextResponse | null {
  const country = req.headers.get('x-vercel-ip-country') || 'US';
  const currentLocale = req.nextUrl.locale;
  if (currentLocale !== 'default') return null;

  const targetLocale = COUNTRY_LOCALE_MAP[country] || 'en';
  const url = req.nextUrl.clone();
  url.locale = targetLocale;
  return NextResponse.redirect(url);
}
```

Pair with Sitecore's multi-site resolver. Geo detection happens at the edge (0ms cost) before PersonalizeMiddleware, reducing CDP calls.

## Multi-site architecture on Vercel

Use **one Vercel project per application head**:

```
Sitecore XM Cloud Instance
  ├── Vercel Project: brand-a.com
  │     └── Domain: brand-a.com
  ├── Vercel Project: brand-b.com
  │     └── Domain: brand-b.com
  └── Vercel Project: careers.brand-a.com
        └── Domain: careers.brand-a.com
```

Benefits:
- **Isolated bundles** — only CSS/JS for that site ships
- **Independent deploys** — release one site without affecting others
- **Per-site config** — firewall rules, env vars, logging, domains
- **Rendering flexibility** — SSR, SSG, or ISR per site

## Middleware optimization matrix

Systematically evaluate which middleware plugins are needed:

| Plugin | Action | Impact |
|--------|--------|--------|
| Redirect middleware | Replace with Edge Config | Eliminates Edge GraphQL per request |
| Multisite middleware | Remove if single-site | Eliminates site resolution overhead |
| Personalize middleware | Restrict `matcher` to specific paths | Reduces CDP calls |
| CSP middleware | Only compute for `text/html` requests | Skip for assets/API |
