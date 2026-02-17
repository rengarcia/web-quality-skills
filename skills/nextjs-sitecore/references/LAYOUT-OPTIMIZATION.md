# Layout payload optimization

## The problem

Sitecore Layout Service returns the full page layout as a JSON response. This includes every field of every component, plus referenced items and system metadata. A typical page can produce 200-500 KB of layout JSON.

This JSON is embedded as `__NEXT_DATA__` in the HTML, meaning:
- Every byte is downloaded by the browser
- React must parse and hydrate the entire payload
- Large payloads delay Time to Interactive and increase memory usage

## Measuring payload size

Check `__NEXT_DATA__` size in browser DevTools:

```js
// Run in browser console
const data = document.getElementById('__NEXT_DATA__')?.textContent;
console.log(`Layout payload: ${(data?.length / 1024).toFixed(1)} KB`);
```

Target: keep `__NEXT_DATA__` under 100 KB for good performance.

## The optimizer pattern

### Core concept

Each component declares which fields it actually needs. A layout optimizer walks the component tree and strips everything else before the data reaches `__NEXT_DATA__`.

### `defineOptimizer` helper

```tsx
// lib/layout-optimizer/define-optimizer.ts
import { ComponentRendering } from '@sitecore-jss/sitecore-jss-nextjs';

type OptimizerFn<T> = (component: ComponentRendering) => Partial<T>;

export function defineOptimizer<T>(
  componentName: string,
  optimize: OptimizerFn<T>
) {
  return { componentName, optimize };
}
```

### `defineOwnFieldsOptimizer` helper

For components that only use their own declared fields (most common case):

```tsx
// lib/layout-optimizer/define-own-fields-optimizer.ts
export function defineOwnFieldsOptimizer<T>(
  componentName: string,
  fieldNames: (keyof T)[]
) {
  return defineOptimizer<T>(componentName, (component) => {
    const result: Partial<T> = {};
    for (const field of fieldNames) {
      if (component.fields?.[field as string] !== undefined) {
        result[field] = component.fields[field as string];
      }
    }
    return result;
  });
}
```

### Example: HeroBanner optimizer

```tsx
// components/HeroBanner/HeroBanner.optimizer.ts
import { defineOwnFieldsOptimizer } from 'lib/layout-optimizer';
import { HeroBannerFields } from './HeroBanner.types';

export const heroBannerOptimizer = defineOwnFieldsOptimizer<HeroBannerFields>(
  'HeroBanner',
  ['title', 'subtitle', 'backgroundImage', 'ctaLink', 'ctaText']
);

// The original payload might include 20+ fields from the datasource
// After optimization, only these 5 fields remain
```

### Optimizer registry

```tsx
// lib/layout-optimizer/registry.ts
import { heroBannerOptimizer } from 'components/HeroBanner/HeroBanner.optimizer';
import { cardListOptimizer } from 'components/CardList/CardList.optimizer';
// ... import all component optimizers

const optimizerRegistry = new Map<string, ReturnType<typeof defineOptimizer>>();

// Register all optimizers
[heroBannerOptimizer, cardListOptimizer].forEach((opt) => {
  optimizerRegistry.set(opt.componentName, opt);
});

export { optimizerRegistry };
```

### Running the optimizer

Apply optimizers in `getStaticProps` or a layout service plugin:

```tsx
// lib/layout-optimizer/optimize-layout.ts
import { LayoutServiceData, ComponentRendering } from '@sitecore-jss/sitecore-jss-nextjs';
import { optimizerRegistry } from './registry';

export function optimizeLayout(layoutData: LayoutServiceData): LayoutServiceData {
  if (!layoutData?.sitecore?.route) return layoutData;

  visitComponents(layoutData.sitecore.route, (component) => {
    const optimizer = optimizerRegistry.get(component.componentName);
    if (optimizer) {
      component.fields = optimizer.optimize(component) as Record<string, unknown>;
    }
    stripSystemFields(component);
  });

  return layoutData;
}

function visitComponents(
  route: ComponentRendering,
  visitor: (component: ComponentRendering) => void
) {
  if (route.placeholders) {
    Object.values(route.placeholders).forEach((placeholder) => {
      placeholder.forEach((component) => {
        if ('componentName' in component) {
          visitor(component as ComponentRendering);
          visitComponents(component as ComponentRendering, visitor);
        }
      });
    });
  }
}

const SYSTEM_FIELDS = [
  '__Created', '__Updated', '__Revision', '__Sortorder',
  '__Display name', '__Hidden', '__Read Only', '__Renderings',
];

function stripSystemFields(component: ComponentRendering) {
  if (!component.fields) return;
  SYSTEM_FIELDS.forEach((field) => delete component.fields![field]);
}
```

### Integration with `getStaticProps`

```tsx
// lib/page-props-factory/plugins/layout-optimizer.ts
import { optimizeLayout } from 'lib/layout-optimizer/optimize-layout';

export class LayoutOptimizerPlugin {
  order = 100; // Run after layout data is fetched

  async exec(props: SitecorePageProps) {
    if (props.layoutData) {
      props.layoutData = optimizeLayout(props.layoutData);
    }
    return props;
  }
}
```

## Impact

Typical results from implementing layout optimization:

| Metric | Before | After |
|--------|--------|-------|
| `__NEXT_DATA__` size | 380 KB | 95 KB |
| HTML document size | 420 KB | 135 KB |
| Time to Interactive | 4.2s | 2.8s |
| Hydration time | 180ms | 65ms |

## Guidelines for adding optimizers

1. **Every new component should have an optimizer** — add it alongside the component
2. **Co-locate optimizer files** — `ComponentName.optimizer.ts` next to `ComponentName.tsx`
3. **Start with `defineOwnFieldsOptimizer`** — covers 90% of cases
4. **Use `defineOptimizer` for complex cases** — when you need to transform or reshape fields
5. **Test with large pages** — verify the optimizer doesn't strip fields that are actually needed
6. **Monitor `__NEXT_DATA__` size** — add it to your performance monitoring
