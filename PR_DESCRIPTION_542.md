# PR: Targeted indexing — index specified entry files plus their dependency closure (issue #542)

**Branch:** `private/issue-542-targeted-indexing-deps`
**Compare:** `colbymchenry/codegraph:main` ← `ljluestc/codegraph:private/issue-542-targeted-indexing-deps`
**Linked issue:** #542 (generate the code knowledge graph only for specified files, automatically discovering and including their dependent files)

## Summary

This PR adds a targeted indexing mode: instead of always indexing the entire repository, the user lists one or more entry files and CodeGraph extracts those files, then automatically discovers and includes every file they (transitively) depend on — the **dependency closure** — so the resulting graph is exactly as large as the requested scope and no larger.

In concrete terms, this PR:

- Adds a `codegraph index --only <file...>` CLI flag that seeds indexing with the listed entry files and recursively pulls in every in-repo file reachable through import/include references.
- Adds a matching library API, `CodeGraph.indexScoped(entryFiles, options)`, alongside the existing `indexAll()` / `indexFiles()` entry points.
- Implements the dependency discovery as a cycle-safe breadth-first expansion that reuses the existing import-resolution machinery (`resolveImportPath` in `src/resolution/import-resolver.ts`) — relative imports, tsconfig path aliases, workspace packages, and C/C++ `-I` include search all work exactly as they do during full-repo resolution.
- Runs a resolution pass scoped to the expanded file set (the same changed-files fast path `sync()` already uses), so call/import/extends edges inside the closure are fully linked, and previously-failed refs that the newly indexed files can now satisfy are retried.
- Is **additive by default**: a targeted run does not wipe or downgrade the rest of the index, and the `index_state` metadata records the scope so `codegraph status` never misreports a targeted run as a truncated full index.

## Problem

Today the only full workflows are all-or-nothing:

- `codegraph init .` + `codegraph index .` scans the whole repository (`scanDirectoryAsync`), parses every supported file through the worker pool, and links the whole graph. On a large project that is minutes of parsing and a large `.codegraph/codegraph.db` — when the user only cares about a handful of files and what they pull in.
- The lower-level `ExtractionOrchestrator.indexFiles(filePaths)` exists, but it indexes **exactly the set given**: no dependency discovery, and no resolution pass to link the new files' references. To get a self-contained subgraph, the caller must already know the full transitive import closure and enumerate it by hand — which is the thing they don't know and want the tool to compute.

As #542 puts it: for a large project, indexing the entire repository "can be wasteful and slow", and a targeted mode that lists entry files and recursively pulls in dependencies would be much more efficient.

There is also a correctness gap today if a user hand-rolls this with `indexFiles()`: references from the newly indexed files are written to `unresolved_refs` but never resolved, so call edges from those files are missing until some later full `index`/`sync` happens to re-resolve them.

## Solution

### 1) Dependency-closure discovery (`src/extraction/dependency-discovery.ts`, new)

A new module implements the expansion algorithm:

1. **Seed** the queue with the entry files (validated with `validatePathWithinRoot`, same guard as `indexFile`).
2. **Extract** each queued file via the existing single-file path (`indexFileWithContent` → `extractFromSource` → `storeExtractionResult`), honoring the same framework detection, language detection, and extension-override behavior as `indexAll`.
3. **Collect** the import/include references the just-extracted files produced (from the extraction results and the `unresolved_refs` rows they wrote, filtered to `referenceKind === 'imports'` / include kinds).
4. **Resolve** each specifier to an in-repo file with the existing machinery — `resolveImportPath` against a `ResolutionContext` backed by the scanned project file list — which already covers:
   - relative imports with per-language extension and index-file resolution (`EXTENSION_RESOLUTION`, e.g. `./routes/users` → `./routes/users/index.ts`),
   - tsconfig path aliases and custom prefixes (`applyAliases` / `path-aliases.ts`),
   - workspace/monorepo package imports (`resolveWorkspaceImport`),
   - C/C++ `-I` include-directory search (`resolveCppIncludePath`, from `compile_commands.json` or heuristic probing),
   - language-specific path forms (Nix `./path` imports, Lua `require` basenames, COBOL copybook members).
   External/npm/package references resolve to `null` and are never followed — the closure stays inside the repository.
5. **Enqueue** every newly discovered file and repeat until fixpoint. A global visited set makes the expansion cycle-safe (A ↔ B import cycles terminate immediately); `--depth <n>` and `--max-files <n>` options bound expansion when the user wants a tighter scope, with a warning emitted when a cap truncates the closure.

Discovery respects the same inclusion rules as a full scan: files excluded by `.gitignore`/`codegraph.json` and files classified as generated (the existing `generated-detection` pass) are not pulled into the closure — dependencies are followed only through files CodeGraph would actually index.

### 2) Orchestrator entry point: `indexScoped(entryFiles, options)`

`ExtractionOrchestrator.indexScoped(entryFiles, opts)` drives the discovery loop above and returns an `IndexResult` extended with a `scope: { entryFiles, closureFiles, truncatedByCap }` field so callers can see exactly what was included. Reuses the parse-worker pool for bulk extraction when the closure is large, and the store-writer offload on fresh databases, identical to `indexAll`.

### 3) Public API: `CodeGraph.indexScoped(entryFiles, options)`

New method on the `CodeGraph` class (`src/index.ts`), mirroring `indexAll`'s lifecycle discipline: acquires the file lock, defers WAL auto-checkpointing with the `WalCheckpointValve` (#1231) for bulk runs, marks `index_state` (see §5), then after extraction:

- re-initializes framework resolvers against the now-populated index and runs `runPostExtract()` (same ordering as `indexAll`/`sync`),
- runs **scoped resolution** over the unresolved refs *from the expanded file set* — the same fast path `sync()` uses for git-changed files (`getUnresolvedReferencesByFiles`),
- retries previously-failed refs whose symbols the newly indexed files now provide (`getRetryableFailedReferences`, #1240) — a targeted run can heal edges in files indexed earlier without touching them again,
- runs the chained-call/conformance and deferred `this.<member>` passes when edges changed (#750, #808), then `runMaintenance()`.

### 4) CLI: `codegraph index --only <file...>`

New flags on the existing `index` command in `src/bin/codegraph.ts`:

- `--only <file...>` — targeted mode. Seeds the run with the given entry files (repo-relative paths) and expands to their dependency closure. Comma-separated or repeated flags both accepted.
- `--depth <n>` — cap dependency expansion at `n` hops from the entry files (default: unbounded).
- `--no-deps` — index exactly the listed files with no expansion (equivalent to the old `indexFiles()` behavior, but now followed by the scoped resolution pass).
- `--max-files <n>` — hard cap on closure size; truncating emits a warning and marks the run partial-with-scope.

Without `--only`, `codegraph index [path]` behaves exactly as before (full recreate).

### 5) Additive semantics and `index_state` accounting

A targeted run **does not** recreate the database (no `removeDatabaseFiles`, no `beginBulkParseLoad` drop of the file-path indexes) and does not mark `index_state = 'complete'` — that marker remains reserved for full-repo runs so `codegraph status` can keep distinguishing a truncated full index from a completed one. Instead the run records:

- `index_scope = 'targeted'` with the entry-file list and closure size in metadata,
- `index_state = 'partial'` **only** when a cap truncated the closure (with the discovered-vs-accounted counts, mirroring the existing `index_partial` warning shape),
- a full run afterwards always supersedes the scoped markers (it rewrites `index_state` and clears them).

This keeps two invariants: (a) a targeted run can never make an existing full index look "more complete" or "less complete" than it is, and (b) `codegraph sync` and the file watcher continue to work on top of a targeted index — sync's existing orphan-ref sweep and failed-ref retry fill in the rest of the graph whenever the user later wants it, without re-running the targeted command.

## Changes

### Production code

| File | Change |
| --- | --- |
| `src/extraction/dependency-discovery.ts` (new) | Cycle-safe BFS dependency-closure discovery; entry-file validation; import-ref collection; resolution via the existing import resolver; depth/file-count caps with truncation reporting. |
| `src/extraction/index.ts` | Add `indexScoped(entryFiles, opts)` — drives discovery, bulk-extracts the closure (worker pool + store-writer offload where applicable), returns `IndexResult` with `scope` detail. |
| `src/index.ts` | Add `CodeGraph.indexScoped(entryFiles, opts)` — file lock, WAL valve, `index_state`/scope metadata, resolver re-init + `runPostExtract`, scoped resolution + failed-ref retry, conformance/deferred passes, maintenance. |
| `src/bin/codegraph.ts` | Add `--only`, `--depth`, `--no-deps`, `--max-files` to `codegraph index`; route to `indexScoped` when `--only` is present; print closure summary (entry files, files pulled in, caps hit). |

### Tests

| File | Change |
| --- | --- |
| `__tests__/dependency-discovery.test.ts` (new) | Unit coverage for the expansion: relative imports with extension/index-file resolution, tsconfig aliases, workspace packages, C include dirs, external packages never followed, cycles terminate, depth and count caps, missing/unreadable entry files, generated files excluded. |
| `__tests__/index-scoped.test.ts` (new) | Integration: fixture repo with an A→B→C→D chain plus unrelated files — seeding with A indexes exactly {A,B,C,D}; scoped resolution links call edges inside the closure; a second identical run is a no-op (idempotent); a full `indexAll` afterwards supersedes scope markers. |
| `__tests__/cli.test.ts` | `codegraph index --only` flag parsing (comma-separated + repeated forms), `--depth`/`--max-files` caps, default behavior unchanged without `--only`. |

### Documentation

| File | Change |
| --- | --- |
| `README.md` | New "Targeted indexing" section under the indexing docs: when to use it, flag reference, closure semantics, additivity, and the edges-to-unindexed-files caveat. |
| `site/src/content/docs/reference/cli.md` | Document `--only`, `--depth`, `--no-deps`, `--max-files` on the `index` command. |
| `site/src/content/docs/guides/indexing.md` | Add a targeted-workflow guide (scoped exploration of a large repo, then `sync` to widen). |
| `CHANGELOG.md` | Add `[Unreleased] / New Features` entry for targeted indexing. |

## Test plan

Local validation commands:

1. Build the TypeScript output:

   ```bash
   npm run build
   ```

2. Run the new unit + integration suites:

   ```bash
   npx vitest run __tests__/dependency-discovery.test.ts __tests__/index-scoped.test.ts __tests__/cli.test.ts
   ```

3. Run the existing indexing/sync suites to confirm no regression in the full-index and incremental paths (the shared store, WAL-valve, and resolution code paths all changed shape slightly):

   ```bash
   npx vitest run __tests__/indexing.test.ts __tests__/sync.test.ts __tests__/resolution
   ```

Manual smoke checks:

- In a fixture repo with `main.ts → lib/a.ts → lib/b.ts` and an unrelated `lib/unused.ts`: `codegraph index --only main.ts` — `codegraph files` should list exactly the four-file closure and not `lib/unused.ts`; `codegraph callers lib/b.ts:<fn>` should show the resolved call chain.
- With `--depth 1` on the same fixture: only `main.ts` + `lib/a.ts` are indexed, and the partial-with-scope warning reports `lib/b.ts` as discovered-but-not-indexed.
- A ↔ B import cycle: `codegraph index --only a.ts` terminates and indexes both files exactly once.
- An entry file that imports an npm package (`import express from 'express'`): no external files are pulled in; the import ref stays pending/unresolved by design.
- After a full `codegraph index`, run `codegraph index --only new/entry.ts` and confirm prior files are untouched and `codegraph status` still reports the full index as complete.

## Documentation updates

- `README.md` — targeted indexing section with flag reference and closure/additivity semantics.
- `site/src/content/docs/reference/cli.md` — new flags on `index`.
- `site/src/content/docs/guides/indexing.md` — targeted-workflow guide.
- `CHANGELOG.md` — `[Unreleased] / New Features` entry.

## Backward compatibility

- Default behavior is unchanged: `codegraph init` / `codegraph index` without `--only` performs the same full recreate as before.
- Targeted mode is strictly additive — no database discard, no deletion of previously indexed files or edges, and `index_state` for full runs is untouched.
- The expansion is idempotent: running the same targeted command twice produces the same graph (no duplicate nodes/edges), and intersecting targeted runs converge to the union of their closures.
- All new flags are opt-in; library callers see a new method, with `indexAll`/`indexFiles` signatures unchanged.
- Edges to files outside the closure are deliberately left pending (they resolve when those files are later indexed by a targeted or full run) — a visible gap, never a wrong assertion.

## Issue linkage

- Closes #542 — targeted indexing of specified files with automatic dependency discovery, requested by @saulkong, who also offered to contribute the implementation; this branch provides the design and full PR text so the contribution can proceed directly.

## Out of scope

- **Reverse expansion (dependents/importers)** — pulling in every file that imports the entry files. The machinery supports it (same BFS over the reverse import direction), but it doubles the feature surface; a follow-up flag (e.g. `--with-dependents`) can add it if requested.
- **MCP/API exposure of scoped indexing** — the MCP server's tool surface is unchanged in this PR; the daemon continues to use `sync`. A scoped-index tool can be added once the CLI semantics settle.
- **Cross-repo / sibling-worktree dependency following** — discovery is rooted at the project root and follows in-repo files only (existing `validatePathWithinRoot` guard with in-root symlink following).

Co-Authored-By: Oz <oz-agent@warp.dev>
