# ⚡ Draft Coach — Brawl Stars Ranked Draft Assistant

A single-file, no-build ranked-draft assistant for Brawl Stars, inspired by PL Prodigy's draft UI but powered by a **self-contained rule engine** built on Bobby's draft-guide rules (7 draft classes, ban strategy, pick-slot logic, mode strategies, coin-flip bans).

> Online: `public/index.html` is the entire app. No build step, no framework, no backend.
> Companion pages: `public/matrix.html` (108-brawler reference matrix), `public/guide.html` (role & ability cheat sheet), `public/strategy-diff.html` (13-row rule diff report from strategy videos).

---

## 1. How to use

| Step | Action |
|---|---|
| 1 | Pick the **game mode**, **map**, **draft slot band** (`PICK 1`, `PICKS 2-3`, `PICKS 4-5`) and the **coin flip** (🪙 we pick first / opponent picks first) |
| 2 | In the **DRAFT BOARD** card, tap **Pick Blue** / **Pick Red** / **Ban**, then tap brawlers in the selection grid to fill the slots |
| 3 | Tap a filled portrait to select that slot (next pick replaces it); tap its `×` badge to clear |
| 4 | Watch the **SCORE** card (left column) — top-6 recommendations update live after every pick, ban, map or mode change. Tap a row to lock that brawler |
| 5 | The **DRAFT BOARD** lanes show per-lane matchup % + a ★ suggested counter when your lane slot is empty against a locked enemy; once all 6 are locked, lanes **auto-rearrange to your best possible matchups** (strongest on MID, ⚡ shown); the amber **Odds of winning** pill appears when all 6 picks are locked |
| 6 | Recommended bans (grey) live inside the DRAFT BOARD under "Recommended bans ▾" |
| 7 | `RESET` clears the draft |

Layout: laptop = draft board + score + intel on the left half, full-height brawler selection on the right half (50/50). Mobile stacks everything vertically.

**Color identity:** blue = your team · red = enemy team · grey = bans (grayscale portraits).

---

## 2. Scoring system (`recommend()`)

Every not-yet-taken brawler gets a raw score from additive rules. The SCORE card shows the top 6 in **descending order**, displayed as Prodigy-style "absolute" numbers — anchored to a fixed 140-point scale (`99 × score / max(140, top+6)`, clamped to 99), so the numbers themselves move when picks/enemy/map change instead of pinning the best at 99.

### Score components (universal)

| Component | Effect |
|---|---|
| Mode affinity | ±20 / −12 if the brawler's mode affinity doesn't include the current mode |
| Meta5 | +25 for current-meta top picks |
| Buffie upgrades 🧪 | +5 per-brawler Gadget/Star-Power/Hypercharge upgrade bonus |
| Map geometry | OPEN: snipers +14, throwers/tanks −12 · CLOSED/BUSHY: space makers +10, throwers +8, pure snipers −10 |
| **Per-map class bias** | every map carries its own class swings (`MAP_META.bias`, ±4 to ±16) — so the **same mode recommends differently on each specific map** (e.g. Super Beach pushes range, Sneaky Fields pushes dive/ambush) |
| Mode objectives | mode-specific rules beyond AGGRO/PASSIVE: Gem Grab mid-control +8 / knockback +6, Hot Zone area denial +8 & bodies +6, Heist tank HP +8, Brawl Ball wall-break +6, Knockout range +6 / sustain −8, Bounty thrower zones +6 |
| Map edge list | +20 for classic per-map strengths |
| Counter pairs | +18 for each locked enemy the brawler counters (mode-keyed counters count too, e.g. Chuck on Heist) |
| Being countered | −22 per locked enemy that counters the brawler |
| Class vulnerability | throwers vs enemy mobility −40 · pure snipers vs enemy space makers −45 |
| Support without damage partner | −25 |
| Role stacking | 2nd duplicate of a class −18 · 1st duplicate of anti-tank/control/support −8 · 3rd sniper in late band −25 |
| Thrower vs double-support heal comp | −15 |

### Draft-band rules (the drafting system)

- **PICK 1 (first lock)** — safest-all-rounder logic: aggressive modes favor anti-tanks (+82 on open maps, +100 otherwise); passive modes favor utility snipers (+60) and control (+50) while punishing first-pick throwers (−80) and space makers (−30). Ban-aware: counts remaining counters to refine risk (−10 risky, +11 ban-freed).
- **PICKS 2-3** — react to the reveal: if the enemy has no anti-tank yet, space makers +80; anti-tank +55 (+25 to also answer a locked tank); passive modes: utility sniper +45, control +40, anti-tank +35, hyper-charge +20.
- **PICKS 4-5** — counter/fill: enemy comp fully passive → space makers +85 / tanks +65; anti-tank vs their space maker or tank +85; vs their sniper/thrower → space maker +80 (+20 only if their anti-tank screen is 2+); passive modes: throwers +85 into a counter-less comp, tanks +45 vs double sustain, space makers +100 over 2+ pure snipers, anti-tank +60 vs their space maker.

### Mode strategies
`GAMEMODES` entries carry `meta: 'AGGRO'` (Brawl Ball, Gem Grab, Hot Zone, Heist) or `'PASSIVE'` (Knockout, Bounty) — that flips the whole rule table per mode. Heist adds extra hyper-charge and lobbed-range rules.

---

## 3. Drafting system (engine internals)

- **7 draft classes** (`CLASS_META`): ANTI_TANK, SPACE_MAKER, CONTROL, SNIPER, TANK, SUPPORT, THROWER — each with a color, label and default mode affinity (`CLASS_MODES`). Snipers carry `sub: 'PURE' | 'UTILITY'`, which drives several rules.
- **Trait flags** (`TRAIT_META`): ⭐ good hypercharge, 👊 knockback/stun, 🧱 wall break, ➶ pierce damage, ⚙ support utility.
- **Roster**: 108 brawlers — Bobby's 101-role map plus the 2026 new-era additions; `BUFFIES` marks the 17 Supercell Buffie-upgraded brawlers (🧪 chip + score bonus).
- **Ban engine** (`banPicks()`), coin-flip based:
  - Opponent picks first → ban their best first-pick brawlers: meta anti-tanks/divers in aggressive modes, meta control/utility mainstays in passive modes.
  - We pick first → **never** ban meta anti-tanks (it hands them the best pick); ban 2/3-pivot threats and last-pick cheese (throwers, Shade) instead.
- **Lane matchups** (`laneMatchup`): per-lane % from counter pairs, class geometry, mode affinity and map geo; badge green ≥56%, amber 45–55%, red ≤44%. The ★ suggestion picks the best available counter for an empty lane facing a locked enemy.
- **Lane optimizer** (`bestLaneAssignment`): lanes are **not locked to pick order** — once all 6 brawlers are locked, the engine tests every pairing (36 permutations), maximizes your total matchup %, and tilts the strongest lane onto MID (powered-by ⚡ hint + odds recalculated on the optimal assignment).
- **Odds of winning** = average of the three lane matchups, shown in the amber pill once all 6 picks lock.
- Banned brawlers are excluded from every pool (recommendations, ban guide, selection grid, lanes).

---

## 4. Data-edit notes (gotchas)

- `counters` keys must exactly match a brawler `id` or a `GAMEMODES` id (e.g. Chuck's `'heist'`). Typos fail silently in the UI.
- Adding a brawler = one `BRAWLERS` entry with a valid `cls` enum + an `ICONS` id (Supercell numeric id, borderless portrait CDN).
- Every SNIPER needs `sub`.
- `state.coin`: `'win'` = **we** pick first, `'lose'` = opponent picks first (naming is inverted from intuition).
- Keep `public/matrix.html` in sync when `BRAWLERS` changes.
- Tailwind Play CDN JIT compiles dynamically composed classes at runtime — a static Tailwind build would drop arbitrary classes (e.g. `border-[#f9a8d4]/60`) unless safelisted.

---

## 5. Run & verify

- Serve `public/` over HTTP (any static server) and open `http://127.0.0.1:<port>/index.html` — CDN assets (Tailwind, FontAwesome, fonts, portraits) need network.
- Node is not required for the app; a portable Node lives at `C:\Users\michael.tong\AppData\Local\Programs\nodejs\node-v22.12.0-win-x64\` for maintenance scripts (npm/wrangler).

## 6. Deploy

### Option A — GitHub Pages (no build, auto on every push)
- `.github/workflows/deploy-pages.yml` publishes the `public/` folder on every push to `main`.
- One-time setup: repo **Settings → Pages → Source = "GitHub Actions"**.
- Live URL: `https://jedtong14.github.io/brawl-stars-ranked-drafter/`.

### Option B — Cloudflare Workers (free tier)

- `wrangler.jsonc` is a Workers **assets-only** config (no worker script, no build): `{ name: 'brawl-stars-ranked-drafter', assets: { directory: './public' } }`.
- Auth: `npx wrangler login` (browser OAuth) → check with `npx wrangler whoami` → deploy with `npx wrangler deploy` (first deploy creates the workers.dev URL).
- Keep the asset dir pointing at `public/` — don't deploy the repo root.
- Either host works independently, or both (same `public/` content).

## 7. Repo layout

```
public/index.html        # the app (single file: data + engine + UI)
public/matrix.html       # mirrored 108-brawler reference matrix
public/guide.html        # role & ability cheat sheet
public/strategy-diff.html# rule-diff report from strategy videos
wrangler.jsonc           # Cloudflare Workers assets config
brawl_stars_fast_draft_counter (1).html   # legacy prototype (do not edit)
```
