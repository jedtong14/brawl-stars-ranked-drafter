# AGENTS.md

## What this repo is
- Draft Coach: a single-file, no-build Brawl Stars ranked-draft assistant at `public/index.html` (portal-style counter picker upgraded to Bobby's draft-guide rules: 7 custom draft classes, trait flags, pick-slot logic, mode strategies, and coin-flip ban recommendations).
- Legacy prototype at root `brawl_stars_fast_draft_counter (1).html` is superseded — don't edit it; `public/index.html` is the only source of truth. Quote the legacy filename (spaces + parentheses) if ever needed.
- Not a git repo yet — no history, branches, or CI.

## Verifying changes
- Direct open works, but CDN assets matter; test like this: serve `public/` over HTTP (any static server) and load `http://127.0.0.1/...` in a browser, then read console errors (`tools.browser.console`).
- Tooling note: Node is NOT on PATH; a portable Node lives at `C:\Users\michael.tong\AppData\Local\Programs\nodejs\node-v22.12.0-win-x64\` (node/npm/npx there). Prefix PATH or call binaries by full path.
- Tailwind (Play CDN), FontAwesome, Google Fonts, and Brawlify portraits all load from CDNs — offline, styling/icons vanish but logic still runs.

## Deployment (Cloudflare free tier)
- `wrangler.jsonc` at root: Workers **assets-only** (no worker script, no build): `{ name: 'brawl-stars-ranked-drafter', assets: { directory: './public' } }`.
- `package.json` scripts call `npx wrangler` (wrangler not in devDependencies; npx cache filled on first run — first call is slow).
- Auth: `npx wrangler login` opens a browser OAuth consent; `npx wrangler whoami` to check state. Then `npx wrangler deploy` (first deploy creates the workers.dev URL).
- Keep `wrangler.jsonc`'s asset dir pointing at `public/`; don't deploy repo root (config files would be published).

## Architecture (all inline in public/index.html)
- Data script: `CLASS_META` (7 draft classes: ANTI_TANK, SPACE_MAKER, CONTROL, SNIPER, TANK, SUPPORT, THROWER with hex colors), `CLASS_MODES` (class → default mode affinity, overridable per brawler via `modes`), `TRAIT_META` (HYPER/KB/WALLBREAK/PIERCE/SPECIAL), `GAMEMODES` (each has `meta: 'AGGRO'|'PASSIVE'`), `BRAWLERS` (108 entries: Bobby's 101 Role & Ability Map + 2026 new-era additions bolt/damian/wendy/nori/nova/cosmo/vince; verify counts: 11/12/21/26/8/11/19), `BUFFIES` (17 brawlers with Supercell Buffie upgrades → 🧪 chip + +5 rec bonus), `META5` (Part 2 current-meta top picks), `ICONS` (our id → Brawlify borderless portrait CDN, Supercell numeric ids).
- Logic script: one global `state` (`mode`, `map`, `slot` '1'/'23'/'45', `coin` 'win'/'lose', `target` you/enemy, both 3-slot pick arrays), re-rendered wholesale via innerHTML template literals; after any change call `recompute()`.
- `recommend()` (pick recs): class-need rules per meta × slot band + META5 bonus + mode affinity ± + counter pairs +18/counter − vulnerability warnings (throwers vs mobility, pure snipers vs space makers, supports needing a damage partner).
- `banPicks()` (Part 2 bans): opponent-first → ban meta anti-tanks (aggro) or passive mainstays (KO/Bounty); we-first → NEVER ban meta anti-tanks (hands them the best pick), ban 2/3 pivots + last-pick cheese (throwers, Shade) instead.

## Data-edit gotchas (the most likely task)
- `counters` keys must exactly match a brawler `id` OR a GAMEMODES id (mode-keyed counter, e.g. Chuck's `'heist'`). Both are valid — but typos fail silently in the UI (fixed case: Jessie's `'poko'` → `poco`).
- Since colors/traits come from `CLASS_META`/`TRAIT_META`, adding a brawler = one entry with valid `cls` (exact enum) and an `ICONS` id (numeric Supercell id from api.brawlapi.com/v1/brawlers, name-matched).
- Sniper `sub: 'PURE'|'UTILITY'` drives rules (pure snipers penalized before last pick) — set it on every SNIPER.
- `state.coin` values are 'win' (we pick first) / 'lose' (opponent picks first) — inverted naming trips people up.
- Dynamically composed classes only work because Tailwind Play CDN JIT compiles at runtime; a static Tailwind build would drop arbitrary classes like `border-[#fbbf24]/60` unless safelisted.
