# Code Quality Report

## Project: `cms-map`

**Review Date:** December 11, 2025

---

## 1. Configuration Files

### `package.json`

| Line | Issue | Category |
|------|-------|----------|
| 14-16 | `@webflow/webflow-cli` is in `dependencies` but should be in `devDependencies` | **Bad Practice** |

The project's own cursor rules explicitly state: *"Ensure that the dependency `@webflow/webflow-cli` is always installed in the package.json as a **dev dependency**"*

---

### `index.html`

| Line | Issue | Category |
|------|-------|----------|
| 7 | External stylesheet without **Subresource Integrity (SRI)** hash | **🚨 Security Vulnerability** |

```html
<link href="https://cdn.prod.website-files.com/..." rel="stylesheet" type="text/css"/>
```

This is a supply chain attack vector. If `cdn.prod.website-files.com` is compromised, attackers can inject malicious CSS.

**Fix:**
```html
<link href="..." rel="stylesheet" integrity="sha384-..." crossorigin="anonymous"/>
```

---

| Line | Issue | Category |
|------|-------|----------|
| 13-20 | CSS in `<body>` instead of `<head>` | **Bad Practice** |
| 14-15, 17-18 | `corner-shape` is **non-standard CSS** (Chrome 138+ only, behind flag) | **Browser Compatibility** |

```html
<style>
  .map-holder{
    corner-shape: superellipse(-1.8);
  }
  .marker-popup-img{
    corner-shape: squircle;
  }
</style>
```

This will silently fail in 99% of browsers. No Firefox, Safari, or older Chrome support.

---

## 2. CSS Files

### `src/components/CMSMap/CMSMap.css`

| Line | Issue | Category |
|------|-------|----------|
| 18-29 | `will-change: transform` on potentially **hundreds of markers** | **🐌 Performance** |

```css
.custom-marker {
  /* ... */
  will-change: transform;
  backface-visibility: hidden;
}
```

`will-change` reserves GPU memory for *each* marker. With many markers, this causes memory bloat and can actually *hurt* performance. Use it sparingly, or apply only on hover.

---

| Line | Issue | Category |
|------|-------|----------|
| 48-64 | 8 `!important` declarations | **Code Smell** |

```css
.mapboxgl-ctrl-top-right {
  top: var(--controls-vertical-padding, 0px) !important;
  right: var(--controls-horizontal-padding, 0) !important;
}
/* ... similar for other corners */
```

Indicates CSS specificity wars with Mapbox's styles. Consider using a more specific selector or CSS layers instead.

---

## 3. React Code

### `src/main.tsx`

| Line | Issue | Category |
|------|-------|----------|
| 5 | Non-null assertion `!` without error handling | **Bad Practice** |
| 5 | Missing `<StrictMode>` wrapper | **Bad Practice** |

```tsx
createRoot(document.getElementById("root")!).render(<App />);
```

**Fix:**
```tsx
const root = document.getElementById("root");
if (!root) throw new Error("Root element not found");
createRoot(root).render(<StrictMode><App /></StrictMode>);
```

---

### `src/App.tsx`

| Line | Issue | Category |
|------|-------|----------|
| 6 | **`SlotExmaple`** → should be **`SlotExample`** | **🔤 Typo** |
| 9-11 | `dangerouslySetInnerHTML` with large HTML blob | **Code Smell** |

```tsx
const SlotExmaple = () => {
  return (
    <div
      dangerouslySetInnerHTML={{
        __html: `...`,
      }}
    ></div>
  );
};
```

While the HTML is hardcoded here, this pattern is risky if ever made dynamic.

---

### `src/components/CMSMap/CMSMap.tsx`

| Line | Issue | Category |
|------|-------|----------|
| 36-37 | **Latitude and Longitude are SWAPPED** | **🚨 Bug** |

```tsx
const CMSMap = ({
  centerLat = -90.5795,  // ❌ Invalid! Latitude range is -90 to +90
  centerLng = 39.8283,   // ❌ This looks like a latitude value
```

These default values are Kansas, USA (`39.8283, -98.5795`), but the labels are backwards!

---

| Line | Issue | Category |
|------|-------|----------|
| 18 | **`maker-pop-up`** → should be **`marker-pop-up`** | **🔤 Typo** |

```tsx
const popup = element.querySelector(".maker-pop-up");
```

This typo appears in multiple places (App.tsx HTML also uses `maker-pop-up`). Consistent typo, but still wrong.

---

| Line | Issue | Category |
|------|-------|----------|
| 134-136 | Empty fragment `<></>` instead of `null` | **Code Smell** |

```tsx
) : (
  <></>
)}
```

---

| Line | Issue | Category |
|------|-------|----------|
| 77-87 | Inline styles object recreated every render | **🐌 Performance** |

```tsx
const containerStyle = {
  flex: 1,
  display: "flex" as const,
  // ... more styles
};
```

This object is not memoized. With `useMemo`, it would only recalculate when padding values change.

---

### `src/components/CMSMap/mapUtils.ts`

| Line | Issue | Category |
|------|-------|----------|
| 35 | Accessing **private property** `popup._content` | **Bad Practice** |
| 35 | `console.log` in production code | **Code Smell** |

```tsx
console.log(popup._content);
const element =
  popup.getElement() ||
  popup._content?.querySelector(".marker-popup-content");
```

`_content` is an internal Mapbox API. This will break without warning when Mapbox updates.

---

| Line | Issue | Category |
|------|-------|----------|
| 12-13 | Hardcoded external URL for marker image | **Bad Practice** |

```tsx
el.style.backgroundImage =
  "url(https://docs.mapbox.com/mapbox-gl-js/assets/custom_marker.png)";
```

If Mapbox removes this asset from their docs, all markers break. Should be a local asset or configurable prop.

---

### `src/components/CMSMap/useCMSCollectionItems.ts`

| Line | Issue | Category |
|------|-------|----------|
| 39 | Using `.current` in dependency array | **Bad Practice** |

```tsx
}, [cmsCollectionComponentSlotRef.current, items]);
```

Ref `.current` changes don't trigger re-renders. This is a subtle bug—the effect won't re-run when the ref updates.

**Fix:** Use the ref object itself, or use a callback ref pattern.

---

### `src/components/CMSMap/useMapbox.ts`

| Line | Issue | Category |
|------|-------|----------|
| 119-122 | Empty dependency array ignores `center`, `zoom`, `mapKey` | **🚨 Bug** |

```tsx
return () => cleanupMap(mapRef, markersRef, initializedRef);
}, []); // Empty dependency array - only run once!
```

The comment acknowledges this is intentional, but it means:
- Changing `mapKey` won't reinitialize the map
- Changing `center`/`zoom` won't update the view

ESLint's `react-hooks/exhaustive-deps` would flag this.

---

### `src/components/CMSMap/mapInitializer.ts`

| Line | Issue | Category |
|------|-------|----------|
| 114, 122, 138 | Multiple `console.log/warn/error` statements in production | **Code Smell** |

```tsx
console.error("Map error:", e);
// ...
console.log("Map loaded successfully!");
// ...
console.log("Cleaning up map...");
```

These should be removed or wrapped in a debug flag for production.

---

### `src/hooks/useShadowGlobalStyles.tsx`

| Line | Issue | Category |
|------|-------|----------|
| N/A | File extension `.tsx` but contains **no JSX** | **Code Smell** |

Should be renamed to `.ts`.

---

## Summary

| Category | Count |
|----------|-------|
| 🚨 Security Vulnerabilities | 2 |
| 🚨 Bugs | 3 |
| ❌ Bad Practices | 10 |
| ⚠️ Code Smells | 8 |
| 🔤 Typos | 3 |
| 🐌 Performance Issues | 3 |

**Total Issues Found: 28**

---

## Recommendations

### High Priority (Fix Immediately)
1. Add SRI hash to external stylesheet in `index.html`
2. Fix swapped `centerLat`/`centerLng` default values in `CMSMap.tsx`
3. Remove usage of private Mapbox API (`popup._content`)
4. Fix the ref `.current` anti-pattern in `useCMSCollectionItems.ts`

### Medium Priority
1. Move `@webflow/webflow-cli` to devDependencies
2. Remove or conditionally gate all `console.log` statements
3. Fix the `useMapbox` hook to properly handle prop changes
4. Replace hardcoded marker image URL with local asset

### Low Priority
1. Fix typos (`SlotExmaple` → `SlotExample`, `maker-pop-up` → `marker-pop-up`)
2. Rename `useShadowGlobalStyles.tsx` to `.ts`
3. Replace empty `<></>` with `null`
4. Memoize inline style objects
5. Consider removing `will-change` from markers or applying on hover only

