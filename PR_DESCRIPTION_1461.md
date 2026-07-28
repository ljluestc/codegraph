# Fix Spring route indexing: index every declared mapping path, resolve constant paths

**Fixes #1461**

## Summary

Spring mapping annotations (`@RequestMapping`, `@GetMapping`, `@PostMapping`, etc.) declare a **list** of paths, but `parseMappingPath` in `src/resolution/frameworks/java.ts` returned a single string. That one type choice produced two distinct, silent wrong results:

1. **Multi-valued arrays truncated** — `@RequestMapping({"/a", "/b"})` indexed only `/a`. Every other path the mapping serves was missing from the graph, because `.match` (not `.matchAll`) captured the first string literal only.
2. **Constant paths collapsed to `/`** — `@RequestMapping(Foo.PATH)` contains no string literal, so the parser returned `""`, which `joinPath` turned into `/`. A route was indexed at the site root instead of at its real path.

Both were silent: the route node existed and looked plausible, so nothing signaled the graph was incomplete or wrong. In the project that prompted the issue, 14 of 42 endpoints were missing while the graph read as complete.

This PR makes the parser return **every declared path** and emit the **cross product** of class-level prefixes × method-level paths, and resolves same-file `static final String` constant references to their literal values.

## Root cause

```js
function parseMappingPath(args: string): string {
  const m = args.match(/["']([^"']*)["']/);   // first literal only
  return m ? m[1]! : '';                       // no literal -> ""
}

function joinPath(prefix: string, sub: string): string {
  const parts = [prefix, sub].map((p) => p.replace(/^\/+|\/+$/g, '')).filter(Boolean);
  return '/' + parts.join('/');                // "" + "" -> "/"
}
```

- `.match` instead of `.matchAll` → issue 1.
- The `''` fallback flowing through `joinPath` → issue 2.

## What changed

### 1. `parseMappingPaths` returns `string[]`, scoped to `value`/`path`

- Only the `value=` / `path=` attribute is treated as a route path. Other attributes hold string literals too — `produces = "application/json"` must never be read as a route.
- Bare annotations (`@GetMapping` with no args) return `['']`, which joins to the class-level base, preserving existing behavior.

### 2. Constant resolution

A single pass per compilation unit collects same-file constants:

```
/\bstatic\s+final\s+String\s+(\w+)\s*=\s*"([^"]*)"/g
```

`@RequestMapping(Foo.PATH)` and `@RequestMapping(PATH)` resolve to the literal value (`/error`). Constants imported from another class are **left unindexed** rather than emitted as `/` — a visible gap is better than a wrong assertion at the site root.

### 3. Cross-product emission at the call site

For each class-level prefix × each method-level path, `joinPath` is applied and a route node is emitted. Distinct paths produce distinct nodes without collision because the existing node id already includes `routePath`:

```
route:${filePath}:${line}:${method}:${routePath}
```

### 4. Comment handling preserves line numbers

Comments are **blanked** (replaced with spaces, newlines kept) instead of stripped, so every `start_line` reported for a mapping annotation matches its position in the original file even after multi-line Javadoc.

### 5. Class-level mapping treated as a prefix, not a route

When iterating mapping annotations for methods, the class-level annotation is skipped, preventing the base from being joined onto itself (`/api` → `/api/api`).

## Behavior comparison

| Input | Before | After |
|---|---|---|
| `@RequestMapping({"/a","/b"})` + `@GetMapping({"/x","/y"})` | 1 route: `/a/x` | 4 routes: `/a/x`, `/a/y`, `/b/x`, `/b/y` |
| `@RequestMapping(Foo.PATH)` where `PATH = "/error"` | `ANY /` | `GET /error` |
| `@GetMapping(value = "/ok", produces = "application/json")` | `/ok` (or worse if literal order changed) | `/ok` only |
| `@RequestMapping(method = {RequestMethod.GET})` under `@RequestMapping("/base")` | `/` | `/base` |
| `@GetMapping` (bare) under `@RequestMapping("/base")` | `/base` | `/base` (unchanged) |
| Mapping preceded by multi-line Javadoc | shifted `start_line` | correct `start_line` |

## Testing

Regression cases added covering the matrix from the issue:

- Multi-valued arrays at both class and method level emit the full cross product.
- Same-file `static final String` path constants resolve to their literal.
- Cross-file constants are left unindexed (no false `/` route).
- `produces`/`consumes`/`method` attributes are never read as paths.
- Bare method annotations join to the class-level base.
- Annotations following multi-line Javadoc keep their original `start_line`.

Verified against the two reproductions in the issue:

- `ItemController` (`{"/api", "/{tenant}/api"}` × `{"/items", "/items/{id}"}`) — all 4 routes indexed (was 1).
- `ErrorHandler` (`@RequestMapping(ErrorHandler.PATH)`) — `GET /error` indexed (was `ANY /`).

## Impact

Anything built on route nodes — `codegraph impact`, route-to-handler tracing, and every consumer of route nodes — previously inherited both defects. The constant case was the more dangerous: it did not merely omit a route, it asserted a route at `/` that does not exist, and for shared-library controllers that wrong root route was attributed to every embedding application. After this change, the indexed HTTP surface matches the actually served surface for multi-valued mappings, and no phantom `/` routes are produced.

## Out of scope

- Resolution of constants imported from other files/classes (cross-compilation-unit constant folding). These are deliberately left unindexed in this PR; a follow-up can add cross-file constant resolution if desired.
- Ant-style pattern expansion (`/items/**`, `/items/{id:[0-9]+}`) — patterns are indexed as declared, as before.
