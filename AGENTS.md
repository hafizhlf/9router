# AGENTS.md — 9Router Fork Customization Guide

> This document governs how customizations are made on the `hafizhlf/9router` fork while staying in sync with upstream (`decolua/9router`).

## 1. Project Overview

**9Router** is a local AI routing gateway and Next.js dashboard. It provides a single OpenAI-compatible endpoint (`/v1/*`) and routes traffic across 40+ upstream providers with format translation, fallback, token refresh, and usage tracking.

| What | Value |
|---|---|
| Upstream | `https://github.com/decolua/9router` |
| Fork | `git@github.com:hafizhlf/9router.git` |
| Stack | Next.js 16, React 19, Zustand, Tailwind CSS 4, Express 5 |
| Runtime core | `open-sse/` (SSE, translation, execution, RTK) + `src/sse/` |
| Dashboard | `src/app/` (App Router) + `src/store/` (Zustand) + `src/lib/` |

### Remotes

```
origin   → git@github.com:hafizhlf/9router.git  (your fork)
upstream → https://github.com/decolua/9router.git (original)
```

## 2. Branching Strategy

```
upstream/master ──●──●──●──●──●──●──●──●──●──  (upstream moves forward)
                  │           │        │
master ───────────●───────────●────────●───────  (clean mirror of upstream)
                  └───┬───┘   └─ rebase ─┘
                      │
custom ───────────C───C───C───C───C─────────────  (your customizations)
                  ^                       ^
                  base commit             HEAD
```

### Branch roles

| Branch | Purpose | Rule |
|---|---|---|
| `master` | Clean mirror of `upstream/master` | **Never commit here.** Only fast-forward merges from upstream. |
| `custom` | Long-lived branch for all fork customizations | Rebased onto `master` after each upstream sync. Force-push with `--force-with-lease`. |
| `feat/*`, `fix/*` | Short-lived feature/fix branches | Branched from `custom`, merged back via squash-merge or regular merge. Delete after merge. |

### Why rebase, not merge?

Rebase keeps a linear history — your custom commits always sit on top of upstream's latest. This makes it trivial to see what's yours vs. what's upstream, and makes conflict resolution scoped per-commit rather than in a single massive merge commit.

## 3. Syncing with Upstream

Run this whenever you want to pull in upstream changes:

```bash
# 1. Update master to match upstream
git checkout master
git fetch upstream
git merge upstream/master          # fast-forward only (should never conflict)
git push origin master

# 2. Rebase custom onto the new master
git checkout custom
git rebase master

# 3. Force-push the rebased custom branch
git push origin custom --force-with-lease
```

### If rebase has conflicts

```bash
# Git pauses at the conflicting commit. Resolve, then:
git add <resolved-files>
git rebase --continue

# To skip a commit that's no longer needed (e.g. upstream fixed the same thing):
git rebase --skip

# Nuclear option — abort and try later:
git rebase --abort
```

**Tip:** Smaller, focused commits = easier conflict resolution. If a conflict is hard, it usually means the commit touches a high-churn area (see §5).

### Sync cheat sheet (one-liner)

```bash
git checkout master && git fetch upstream && git merge upstream/master && git push origin master && git checkout custom && git rebase master && git push origin custom --force-with-lease
```

## 4. Customization Guidelines

### 4.1 Prefer adding files over modifying upstream files

Adding new files creates zero conflict surface. Modifying upstream files creates conflict surface proportional to how often upstream changes that file.

| Approach | Conflict risk | Example |
|---|---|---|
| **New file** | None | Add `open-sse/translator/request/openai-to-newprov.js` |
| **New file + 1-line import** | Very low | Register the translator by adding one `register()` call |
| **Modify upstream file** | High | Change logic inside `open-sse/config/providers.js` |

### 4.2 When you must modify an upstream file

- Keep changes **small and localized**
- Place custom code **at the end of the file** or **at the end of a function** when possible — upstream rarely appends to the same spots
- Mark custom sections with a comment: `// [CUSTOM]` or `/* [CUSTOM] */` so they're searchable and identifiable during rebase

```js
// upstream code...
// upstream code...

// [CUSTOM] hafizhlf: added GLM OpenAI-compatible output format
export function formatGLMResponse(data) {
  // ...
}
```

### 4.3 Organize commits for easy rebasing

- **One logical change per commit.** Don't bundle unrelated customizations.
- **Atomic commits** can be reordered, squashed, or dropped during interactive rebase if they conflict.
- **Descriptive commit messages** with a prefix like `feat(custom):` or `fix(custom):` make it easy to identify your commits in `git log`.

```
feat(custom): add GLM OpenAI-compatible output format
fix(custom): handle edge case in custom provider fallback
chore(custom): add custom ESLint rules for fork
```

### 4.4 Standalone features → dedicated directory

If a customization is a self-contained feature (e.g., a new dashboard page, a utility script), place it in its own directory or file rather than weaving it into upstream structure:

```
src/app/custom/my-feature/       ← new dashboard page
src/lib/custom/                  ← shared custom utilities
open-sse/translator/request/     ← new translator (follows upstream convention)
open-sse/translator/response/    ← new translator (follows upstream convention)
```

This makes the feature easy to find, easy to delete, and unlikely to conflict.

## 5. High-Conflict Zones

These files change frequently upstream. Avoid modifying them directly — extend via separate files or wrapper patterns instead.

| File | Why it churns | Safer alternative |
|---|---|---|
| `open-sse/config/providerModels.js` | New models added regularly | Add models in a separate file, import and merge at runtime |
| `open-sse/config/providers.js` | Provider config updates | Same — overlay pattern |
| `package.json` | Dependency bumps | Minimize custom deps; if needed, accept the conflict (inevitable) |
| `src/app/layout.js` | Dashboard restructuring | Keep custom UI in separate page routes under `src/app/custom/` |
| `src/app/page.js` | Landing page changes | Override via route, don't edit directly |
| `open-sse/translator/index.js` | New translators registered | Add registration in a separate init file, import once |

### Low-conflict extension points

These are safe to add to with minimal conflict risk:

- `open-sse/translator/request/` — new request translator files
- `open-sse/translator/response/` — new response translator files
- `open-sse/executors/` — new provider executors
- `src/app/api/` — new API routes
- `src/store/` — new Zustand stores
- `src/lib/` — new utility modules
- `public/` — static assets

## 6. Coding Conventions

### Translator pattern

New provider → two files + one registration:

```
open-sse/translator/request/openai-to-myprov.js   ← translate OpenAI → myprov
open-sse/translator/response/myprov-to-openai.js  ← translate myprov SSE → OpenAI
```

Register in a custom init file (not by editing `translator/index.js` directly):

```js
// src/lib/custom/registerTranslators.js
import { register } from "@/open-sse/translator";
import translateRequest from "@/open-sse/translator/request/openai-to-myprov";
import translateResponse from "@/open-sse/translator/response/myprov-to-openai";
import { FORMATS } from "@/open-sse/translator/formats";

register(FORMATS.openai, FORMATS.myprov, translateRequest, translateResponse);
```

### New provider config

Add to `PROVIDER_MODELS` in `providerModels.js` (high-conflict — expect to resolve on rebase) + provider entry in `providers.js`.

### Dashboard

- Next.js App Router: `src/app/**/page.js` for pages, `layout.js` for layouts
- State: Zustand stores in `src/store/`
- Styling: Tailwind CSS 4 — utility classes, no custom CSS files unless necessary
- Components: co-locate with pages or place in `src/lib/components/`

### Running & testing

```bash
npm run dev                    # dev server on :20128
npm run build                  # production build

# Translator tests (no credentials needed)
cd app && npx vitest run --config tests/vitest.config.js "tests/translator/"

# Translator tests with real providers
cd app && RUN_REAL=1 npx vitest run --config tests/vitest.config.js "tests/translator/real/"

# Lint
npx eslint .
```

## 7. Current Custom Branches

| Branch | Status | Description |
|---|---|---|
| `custom` | Active | Long-lived fork customization branch |
| `feat/glm-openai-output-format` | Active | Configurable OpenAI-compatible output format for GLM Coding |

## 8. Checklist: Adding a New Customization

1. [ ] Branch from `custom`: `git checkout custom && git checkout -b feat/my-feature`
2. [ ] Prefer new files over modifying upstream files
3. [ ] If modifying upstream file: use `// [CUSTOM]` comment, keep changes at end of file/function
4. [ ] One logical change per commit, prefixed `feat(custom):` / `fix(custom):`
5. [ ] Test locally: `npm run dev`, `npm run build`, relevant `vitest` suite
6. [ ] Merge back: `git checkout custom && git merge --squash feat/my-feature` (or regular merge)
7. [ ] Push: `git push origin custom`
8. [ ] Delete feature branch: `git branch -d feat/my-feature`
9. [ ] When upstream updates: follow sync procedure in §3
