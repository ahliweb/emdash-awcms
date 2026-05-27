**Follow-up: implemented and tested in AWCMS-Micro**

Since opening this discussion, I've gone ahead and implemented the plugin-owned translation approach in our AWCMS-Micro fork. It's working well with both native and sandboxed plugin boundaries. Here's what we built and what we learned:

### What we implemented

**1. Manifest-based i18n catalog shipped with the plugin**

Each plugin declares its own translation catalog inline in the manifest:

```typescript
// runtime.ts
export const AWCMS_EXAMPLE_MANIFEST = {
  id: "awcms-micro-example",
  // ...
  i18n: {
    defaultLocale: "en",
    supportedLocales: ["en", "id"],
    messages: {
      en: {
        "awcms.nav.group.dashboard": "Dashboard",
        "awcms.nav.overview": "Overview",
        "awcms.nav.access": "Access Control",
        // ...
      },
      id: {
        "awcms.nav.group.dashboard": "Dasbor",
        "awcms.nav.overview": "Ikhtisar",
        "awcms.nav.access": "Kontrol Akses",
        // ...
      }
    }
  }
};
```

**2. Namespaced keys with a `resolveLabel` fallback chain**

```typescript
// resolve-label.ts
export function resolveLabel(
  labelKey: string,
  fallbackLabel: string,
  messages: TranslationMessages | undefined,
  requestedLocale: string,
  defaultLocale: string = "en"
): string {
  if (messages) {
    if (messages[requestedLocale]?.[labelKey]) return messages[requestedLocale][labelKey];
    if (messages[defaultLocale]?.[labelKey]) return messages[defaultLocale][labelKey];
  }
  return fallbackLabel || labelKey;
}
```

Fallback order: requested locale → default locale → `fallbackLabel` → `labelKey` itself.

**3. Navigation manifest pattern: `labelKey` + `fallbackLabel`**

Every nav item carries both a translation key and an English fallback so nothing breaks if a locale is missing:

```typescript
{
  id: "overview",
  labelKey: "awcms.nav.overview",
  fallbackLabel: "Overview",
  path: "/overview",
  permission: "awcms:example:dashboard:read",
}
```

**4. `PluginLocalNav` component consumes locale + messages**

```tsx
// render-plugin-local-nav.tsx
<PluginLocalNav
  groups={normalizedGroups}
  currentPath={currentPath}
  locale={locale}              // from Lingui i18n.locale
  messages={manifest.i18n?.messages}
  defaultLocale="en"
/>
```

**5. Integration with Lingui in the admin**

The plugin admin pages use `useLingui()` to get the active locale, then pass it down to plugin-owned navigation and UI components:

```tsx
const { i18n } = useLingui();
const locale = i18n.locale;

<PluginLocalNav locale={locale} messages={AWCMS_EXAMPLE_MANIFEST.i18n?.messages} />
```

### What works

- **Decoupling confirmed**: plugin translations ship with the plugin. No core catalog changes needed when a plugin adds a language.
- **Sandbox-compatible**: the `resolveLabel` function is pure JS — works identically in native React admin pages and sandboxed server-side routes.
- **Two locales shipped**: English (source) and Indonesian. Adding a third locale is just adding a new key to `messages`.
- **No Lingui dependency in sandboxed plugins**: sandboxed plugins use the plain `resolveLabel` API; native plugins can use Lingui macros if they want.

### What we'd change if starting over

- The inline `messages` object works fine for small-to-medium plugins, but larger plugins might benefit from shipping `.json` files instead of inline objects. The `resolveLabel` API would stay the same.
- We considered a `ctx.translate(key)` API for sandboxed plugin routes but haven't needed it yet — the manifest pattern covers all our use cases so far.

### Templates

We stuck with Astro's i18n for templates, exactly as discussed. EmDash starter templates can follow Astro's routing + i18n patterns without the core CMS needing a separate template translation layer.

### Summary

The approach we proposed works in practice. The key insight is that **plugins should own their translation catalogs entirely** — EmDash only needs to provide the locale resolution context (which locale is active) and a small lookup utility (`resolveLabel`). Everything else — catalog format, key naming, supported locales — stays with the plugin author.
