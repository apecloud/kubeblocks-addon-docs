# kubeblocks-docs Dev Server Troubleshooting

## Symptom

After starting `yarn dev` (or `node_modules/.bin/next dev --port <PORT>`), doc pages under `/docs/<version>/<category>/...` return **500 Internal Server Error** with the message `missing required error components, refreshing...` in the browser.

Server logs show:

```
⨯ TypeError: Cannot read properties of undefined (reading 'call')
   at eval (webpack-internal:///(rsc)/./src/app/[locale]/docs/[version]/[category]/layout.tsx:11:114)
   at <unknown> (rsc)/./src/app/[locale]/docs/[version]/[category]/layout.tsx
     (/Users/.../kubeblocks-docs/.next/server/app/[locale]/docs/[version]/[category]/[[...paths]]/page.js:...) {
  type: 'TypeError',
  page: '/en/docs/preview/user_docs/overview/introduction'
}
```

Line 11 of `layout.tsx` is the import of `next-international/server`:

```ts
import { setStaticParamsLocale } from 'next-international/server';
```

## Root Cause

The `.next` build cache contains a stale compiled reference to `next-international/server`. After a Node.js or package update, or when the cache was built under a different environment, webpack's module graph in the cached `.next/server/app/...page.js` holds a broken function pointer — resulting in `undefined.call` at runtime.

This is **not** an MDX content error, a sidebar YAML error, or a `generate-docs-index` failure. The docs index may be fully valid (328 docs indexed) while this error still occurs.

## Fix

```bash
# 1. Kill the dev server
pkill -f "next dev"

# 2. Clear the stale build cache
rm -rf /path/to/kubeblocks-docs/.next

# 3. Restart (regenerates index + compiles fresh)
cd /path/to/kubeblocks-docs
yarn install && yarn generate-docs-index
node_modules/.bin/next dev --port 3100
```

After a fresh compile (~2–3 min for first page load), all routes return 200.

## Verification

```bash
curl -s -o /dev/null -w "%{http_code}" \
  http://localhost:3100/docs/preview/user_docs/overview/introduction
# expect: 200
```

## Notes

- The `yarn dev` script in `package.json` runs `generate-docs-index` automatically, but does **not** clear `.next`. Clear it manually if you see this error after a package update or environment change.
- The route structure is `/[locale]/docs/[version]/[category]/[[...paths]]`. The `en` locale is stripped by middleware (307 redirect), so the canonical URL is `/docs/preview/user_docs/...` without the `/en/` prefix.
