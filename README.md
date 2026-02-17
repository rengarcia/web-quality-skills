# Web Quality Skills

A comprehensive collection of [Agent Skills](https://agentskills.io/) for optimizing web projects based on [Google Lighthouse](https://developer.chrome.com/docs/lighthouse/overview/) guidelines, Core Web Vitals best practices, and **Sitecore JSS + Next.js performance patterns**.

Built on top of [addyosmani/web-quality-skills](https://github.com/addyosmani/web-quality-skills) — the original framework-agnostic web quality skills by [Addy Osmani](https://github.com/addyosmani). This fork extends the collection with specialized skills for **Next.js + Sitecore JSS/XM Cloud** projects deployed on **Vercel**.

## Why this fork?

The original web quality skills are framework-agnostic and cover Lighthouse audits, Core Web Vitals, accessibility, SEO, and best practices. However, Sitecore JSS + Next.js projects have unique performance challenges that generic skills don't address:

- **Layout payload bloat** — Sitecore Layout Service returns 200-500 KB of JSON per page, serialized as `__NEXT_DATA__`
- **Experience Edge latency** — Transient failures and caching quirks require specialized ISR strategies
- **Sitecore SDK bundle size** — `@sitecore-search`, `@sitecore-feaas`, and `@sitecore-cloudsdk` add significant JavaScript weight
- **Middleware chains** — Redirect, personalization, and multi-site middleware plugins add per-request overhead
- **Vercel-specific optimizations** — Edge Config, Fluid Compute, Skew Protection, and 3-tier caching

This fork adds a dedicated `nextjs-sitecore` skill that encodes real-world optimization patterns for these challenges.

## Available skills

| Skill | Description | Use when |
|-------|-------------|----------|
| **[web-quality-audit](#web-quality-audit)** | Comprehensive quality review across all categories | "Audit my site", "Review this for quality" |
| **[performance](#performance)** | Loading speed, runtime efficiency, resource optimization | "Optimize performance", "Speed up my site" |
| **[core-web-vitals](#core-web-vitals)** | LCP, INP, CLS specific optimizations | "Improve Core Web Vitals", "Fix LCP" |
| **[accessibility](#accessibility)** | WCAG compliance, screen reader support, keyboard navigation | "Improve accessibility", "WCAG audit" |
| **[seo](#seo)** | Search engine optimization, crawlability, structured data | "Optimize for SEO", "Fix meta tags" |
| **[best-practices](#best-practices)** | Security, modern APIs, code quality patterns | "Apply best practices", "Security audit" |
| **[nextjs-sitecore](#nextjs-sitecore)** | Next.js + Sitecore JSS/XM Cloud + Vercel performance | "Sitecore JSS", "XM Cloud", "Experience Edge" |

## Quick start

### Installation

#### Claude Code

```bash
# Install all skills (including Sitecore JSS)
npx add-skill rengarcia/web-quality-skills

# Or using the skills CLI
npx skills add rengarcia/web-quality-skills
```

#### Manual installation

Clone this repo and copy the skills into your project or global skills directory:

```bash
git clone https://github.com/rengarcia/web-quality-skills.git
cp -r web-quality-skills/skills/* your-project/.claude/skills/
```

Or copy only the Sitecore-specific skill:

```bash
cp -r web-quality-skills/skills/nextjs-sitecore/ your-project/.claude/skills/nextjs-sitecore/
```

#### Claude.ai / other agents

Add the `SKILL.md` contents to your project knowledge or paste directly into your conversation. For Sitecore projects, use `skills/nextjs-sitecore/SKILL.md` along with its `references/` files for full coverage.

### Usage

Skills activate automatically when your request matches their description. Examples:

```
Audit this page for web quality issues
```

```
Optimize performance and fix Core Web Vitals
```

```
Optimize this Sitecore JSS Next.js project for performance
```

```
Review the middleware chain and layout payload for this XM Cloud site
```

## Skill details

### web-quality-audit

The comprehensive skill that orchestrates all other skills. Use this for full-site audits or when you're unsure which specific area needs attention.

**Trigger phrases:** "audit my site", "quality review", "lighthouse audit", "check web quality"

**What it checks:**
- All Core Web Vitals metrics
- 50+ performance patterns
- 40+ accessibility rules
- 30+ SEO requirements
- 20+ security/best practice patterns

### performance

Deep-dive into loading and runtime performance optimization.

**Trigger phrases:** "speed up", "optimize performance", "reduce load time", "fix slow"

**Key optimizations:**
- Critical rendering path
- JavaScript bundling and code splitting
- Image optimization (formats, sizing, lazy loading)
- Font loading strategies
- Caching and preloading
- Server response optimization

### core-web-vitals

Specialized skill for the three Core Web Vitals that affect Google Search ranking.

**Trigger phrases:** "Core Web Vitals", "LCP", "INP", "CLS", "page experience"

**Metrics covered:**
- **LCP** (Largest Contentful Paint) < 2.5s
- **INP** (Interaction to Next Paint) < 200ms
- **CLS** (Cumulative Layout Shift) < 0.1

### accessibility

Comprehensive accessibility audit following WCAG 2.1 guidelines.

**Trigger phrases:** "accessibility", "a11y", "WCAG", "screen reader", "keyboard navigation"

**Categories:**
- Perceivable (text alternatives, captions, contrast)
- Operable (keyboard, timing, seizures, navigation)
- Understandable (readable, predictable, input assistance)
- Robust (compatible with assistive technologies)

### seo

Search engine optimization for better visibility and ranking.

**Trigger phrases:** "SEO", "search optimization", "meta tags", "structured data", "sitemap"

**What it covers:**
- Technical SEO (crawlability, indexability)
- On-page SEO (meta tags, headings, content structure)
- Structured data (JSON-LD, schema.org)
- Mobile-friendliness
- Performance signals

### best-practices

Modern web development standards and security practices.

**Trigger phrases:** "best practices", "security audit", "modern standards", "code quality"

**Areas covered:**
- HTTPS and security headers
- Modern JavaScript APIs
- Browser compatibility
- Error handling
- Console cleanliness

### nextjs-sitecore

Specialized performance optimization for **Next.js + Sitecore JSS/XM Cloud** projects deployed on **Vercel**. This skill covers the unique performance challenges of Sitecore headless architectures that generic web quality skills miss.

**Trigger phrases:** "Sitecore JSS", "XM Cloud", "Sitecore Next.js", "Experience Edge", "JSS optimization", "Sitecore Content SDK"

**What it covers:**

- **Layout payload optimization** — Component-level field pruning with `defineOptimizer`, system field removal, reducing `__NEXT_DATA__` from 400 KB to under 100 KB
- **ISR & data fetching** — Selective static generation with `STATIC_PATHS`, revalidation tuning per content type, transient Experience Edge failure handling
- **Bundle optimization** — Split chunks for Sitecore SDKs (`@sitecore-search`, `@sitecore-feaas`, `@sitecore-cloudsdk`), dynamic component factory imports, tree-shaking JSS imports
- **Image optimization** — `next/image` with Sitecore `remotePatterns`, LCP priority for hero banners, Sitecore media handler params (`mw`, `mh`, `as`)
- **Middleware performance** — Plugin chain ordering, Edge Config for redirects (9x faster than GraphQL lookups), personalization middleware scoping
- **Vercel deployment** — Fluid Compute, Skew Protection, 3-tier cache headers, Speed Insights, geolocation locale routing, OpenTelemetry for Experience Edge tracing
- **Component patterns** — `withDatasourceCheck`, lazy placeholders via IntersectionObserver, BYOC/Search SDK lazy loading, re-render prevention in `SitecoreContext`
- **Security** — CSP with nonces for Sitecore scripts, API key protection, Vercel WAF for editing endpoints

**Reference deep-dives:**
- [`references/LAYOUT-OPTIMIZATION.md`](skills/nextjs-sitecore/references/LAYOUT-OPTIMIZATION.md) — Full optimizer pattern with type-safe helpers and registry
- [`references/ISR-CACHING.md`](skills/nextjs-sitecore/references/ISR-CACHING.md) — Selective static generation, tag-based revalidation, webhook integration
- [`references/VERCEL-DEPLOYMENT.md`](skills/nextjs-sitecore/references/VERCEL-DEPLOYMENT.md) — Edge Config redirects, WAF rules, OpenTelemetry, multi-site architecture

## Thresholds reference

### Core Web Vitals

| Metric | Good | Needs improvement | Poor |
|--------|------|-------------------|------|
| LCP | ≤ 2.5s | 2.5s – 4.0s | > 4.0s |
| INP | ≤ 200ms | 200ms – 500ms | > 500ms |
| CLS | ≤ 0.1 | 0.1 – 0.25 | > 0.25 |

### Performance budget recommendations

| Resource type | Budget |
|---------------|--------|
| Total page weight | < 1.5 MB |
| JavaScript | < 300 KB (compressed) |
| CSS | < 100 KB (compressed) |
| Images | < 500 KB total above-fold |
| Fonts | < 100 KB |
| Third-party | < 200 KB |

### Lighthouse score targets

| Category | Target score |
|----------|--------------|
| Performance | ≥ 90 |
| Accessibility | 100 |
| Best Practices | ≥ 95 |
| SEO | ≥ 95 |

## Framework-specific notes

The core skills (performance, accessibility, SEO, best-practices) are framework-agnostic. Common patterns:

**React/Next.js:** Use `next/image`, `React.lazy()`, `Suspense`, `useCallback`/`useMemo` for INP
**Vue/Nuxt:** Use `nuxt/image`, async components, `v-once`, computed properties
**Svelte/SvelteKit:** Use `{#await}`, `svelte:image`, reactive statements
**Astro:** Use `<Image>`, partial hydration, view transitions
**Static HTML:** Use native lazy loading, `<picture>`, preconnect hints
**Sitecore JSS/Next.js:** Use the dedicated [`nextjs-sitecore`](#nextjs-sitecore) skill for specialized patterns

## Contributing

Contributions welcome! Please follow the [Agent Skills specification](https://agentskills.io/specification).

1. Fork the repository
2. Create your skill in `skills/{skill-name}/SKILL.md`
3. Keep SKILL.md under 500 lines (use `references/` for details)
4. Include practical examples and patterns
5. Submit a pull request

## Resources

- [Google Lighthouse Documentation](https://developer.chrome.com/docs/lighthouse/)
- [web.dev Learn Performance](https://web.dev/learn/performance/)
- [Core Web Vitals](https://web.dev/articles/vitals)
- [WCAG 2.1 Guidelines](https://www.w3.org/WAI/WCAG21/quickref/)
- [Agent Skills Specification](https://agentskills.io/specification)
- [Sitecore JSS Documentation](https://doc.sitecore.com/xmc/en/developers/jss/index.html)
- [Sitecore Content SDK](https://doc.sitecore.com/xmc/en/developers/content-sdk/index.html)

## Credits

The core web quality skills (performance, accessibility, SEO, best-practices, core-web-vitals, web-quality-audit) were created by [Addy Osmani](https://github.com/addyosmani) in the original [web-quality-skills](https://github.com/addyosmani/web-quality-skills) repository. Built with insights from the Chrome DevTools team, web performance experts, and accessibility advocates.

The `nextjs-sitecore` skill and Vercel deployment patterns were added by [Renan Garcia](https://github.com/rengarcia) based on real-world Sitecore JSS + XM Cloud optimization experience.

## License

MIT License - see [LICENSE](LICENSE) for details.
