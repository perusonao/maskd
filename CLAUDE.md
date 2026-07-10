# CLAUDE.md

Guidance for AI assistants (Claude Code and others) working in this repository.

## What this project is

**MASKD** is a two-player, psychological bluff / mind-game card game themed around a
masquerade ball. In this repository it ships as a single self-contained web page where
a human plays against a CPU opponent. It is a prototype: rules and balance are still
being tuned via simulation.

The UI, README, and virtually all code comments are written in **Japanese**. Preserve
this. When adding or editing comments, UI strings, or docs, write them in Japanese to
match the surrounding style unless the user explicitly asks otherwise.

## Repository layout

The entire project is intentionally tiny and has **no build step**:

```
index.html    # The whole game — HTML + CSS + JS, all inline (~1900 lines)
README.md     # Short project overview (Japanese)
CLAUDE.md     # This file
```

Everything lives in `index.html`. There is no `package.json`, no bundler, no
dependencies to install, and no test suite. The one external dependency is Google Fonts
(`Cormorant Garamond`, `Zen Kaku Gothic New`), pulled in via a CSS `@import` — so a live
network connection is needed for the intended typography, but the game works without it.

## Running / developing

- **Run it:** open `index.html` directly in a browser, or serve the folder statically
  (e.g. `python3 -m http.server`) and visit it. No build or install required.
- **Deploy:** the game is meant to be published via **GitHub Pages** from this repo. On
  mobile, "Add to Home Screen" launches it full-screen like an app (PWA-style meta tags
  and an inline SVG icon are set in `<head>`).
- **Verify changes:** since there are no automated tests, verify by playing through a
  match in the browser. A full match is 12 rounds; check the round flow, the fight/retreat
  resolution, the end-game scoring screen, and the history/stats modals.

## Structure inside `index.html`

Read the file as three stacked sections:

1. **`<head>` / `<style>`** — PWA meta tags, inline SVG favicon/app icon, and all CSS.
   Colors are defined as CSS custom properties on `:root` (`--gold`, `--ink`, and one
   per attribute: `--sun --dark --moon --kaos --air`). Reuse these variables instead of
   hardcoding colors.
2. **`<body>`** — the single-page app. It is one `.app` container holding every screen and
   modal as absolutely-positioned overlays (instructions, dealer draw, game-over overlay,
   and the history / stats / tracker / hand / played-detail modals). Screens are shown and
   hidden by toggling the `show` class, not by navigation. Key element ids: `instructions`,
   `dealerDrawOverlay`, `overlay` (game over), `historyModal`, `statsModal`, `trackerModal`,
   `handModal`, `playedDetailModal`.
3. **`<script>`** (starts ~line 681) — all game logic in plain vanilla JS (no framework,
   no modules). Order matters: constants first, then mutable state (`let` globals), then
   functions. `initCpuSettingUI()` is called at the very bottom, after everything is
   defined, on purpose — don't move initialization above the declarations it depends on.

### Code conventions

- **Vanilla JS only.** No frameworks, no libraries, no imports. Keep it that way unless
  asked. DOM access is direct (`document.getElementById(...)`, `.innerHTML`, class
  toggling).
- **Global mutable state.** Game state lives in top-level `let` variables (hands, LP,
  trust, `round`, `youIsDealer`, `phase`, reasoning pools, etc.). Functions read and
  mutate these globals directly. When you add state, follow the same pattern and initialize
  it in the relevant `start*`/`reset` path.
- **Rendering pattern.** State changes are followed by `render*()` calls
  (`renderScore`, `renderHand`, `renderOppHand`, `renderYouPlayed`, ...) that rebuild
  DOM from state. There is no reactive binding — if you change state that's shown on
  screen, call the matching render function.
- **`busy` flag** guards against input during animations/timeouts. Respect it: many flows
  set `busy=true`, run `setTimeout`-driven animation sequences, then clear it.
- Comments are dense and in Japanese, often explaining *why* a balance number was chosen
  (e.g. tuning a cost from 3→2). When you change a tunable, update its comment to match.

## Game rules & key constants (so you don't break balance)

All tunables are `const`s near the top of the `<script>` block. The important ones:

- `TOTAL_ROUNDS = 12` — a match is 12 rounds. Each round has a **dealer (親)** and a
  **child (子)**; roles alternate every round.
- **Deck:** 5 attributes × numbers 0–5. Attributes: `sun / dark / moon / kaos / air`.
  `Air` (0–5) is a **shared pool** shuffled and split 3/3 between players in `dealHands()`;
  the other four attributes are dealt per number.
- **Attribute matchups** (`beats`): `moon > dark`, `dark > sun`, `sun > moon` (a
  rock-paper-scissors triangle). Same attribute → higher number wins.
- **`kaos`** is the wildcard: number-based against non-kaos, wins ties by beating the
  opponent's attribute, plus special cases (`kaos 0/1` beat any `5`; `kaos 5` loses to any
  `1`; `kaos` beats `air` regardless of number).
- **`air`** always forces a **draw** (except vs `kaos`). Playing `air` grants **+1 strategy
  counter** regardless of outcome, and unused `air` cards count **double** at end-game
  (`AIR_LEFTOVER_MULTIPLIER = 2`).
- **Strategy counter (戦略カウンター, "trust"):** a resource separate from score, starting
  at `START_TRUST = 5`. Spent on retreat (`PASS_TRUST_COST = 1`, dealer), ambush/force
  (`FORCE_TRUST_COST = 2`, child), and swap (`SWAP_COST = 2`, dealer, once per round).
- **Scoring:** the fight winner gains points equal to the loser's card number. Final score
  = `LP + remaining strategy counter + leftover-hand bonus`.

The authoritative win/lose/draw resolution is `judge(dealer, child)` and its human-readable
explanation is `whyText(...)`. These two must stay in sync — if you change a rule in
`judge`, update `whyText` (and the instructions markup in `<body>`) to match.

### CPU opponent

The CPU has a **type** (`selectedCpuType`: random/aggressive/cautious/tricky) and a
**difficulty** (`selectedCpuDifficulty`: weak/normal/strong), chosen on the settings
screen. Decision logic lives in `cpuDecide`, `cpuChooseCard`, `cpuDecide`-adjacent
estimators (`cpuEstimate`, `cpuPossibleAttrsFor`, `cpuWantsForce`), and card play in
`cpuActAsDealer` / `cpuActAsChild`. These use the same hidden-information reasoning
(`possibleAttrsByNum`, `allUsedPoolCards`, `revealedCards`) that the player-facing tracker
uses, so keep those pools accurate when you touch card flow.

## Persistence

Cross-match history and aggregate stats are stored in `localStorage` under the key
**`maskd_game_history_v1`** (`HISTORY_KEY`). Reads/writes are wrapped in try/catch so the
game degrades gracefully where storage is unavailable — keep that resilience. If you change
the stored shape incompatibly, bump the key version suffix rather than silently breaking
existing saved data.

## Working conventions for this repo

- **Keep it single-file and dependency-free.** Don't introduce a build system, framework,
  npm packages, or split the code into modules unless the user explicitly requests it.
- **Match the existing style:** Japanese comments/UI, CSS custom properties for color,
  vanilla DOM manipulation, `render*()` after state changes.
- **No test/lint tooling exists** — validate by playing the game in a browser and reasoning
  through the affected flow. State clearly that you verified by manual play, not tests.
- When adjusting balance constants, update the explanatory Japanese comment next to the
  constant and any rule text shown to the player in the instructions overlay.
