# Solitairra - Klondike Solitaire (Animal Edition)

## Overview
Browser-based Klondike solitaire variant, forked from **SoloTerra** (`../soloterra`). Same 40-card deck, suit hierarchy, scoring, and tableau/foundation rules — but with a new default suit style called **Animals** (Dolphins / Hares / Spiders / Cubs) drawn as silhouettes on the Canvas 2D pips. Laser and Classic suit styles are preserved and selectable in Options.

Published to GitHub only (no Railway deployment yet). All leaderboard data is local-storage only.

## Relationship to SoloTerra
- Fork of `../soloterra` — all game logic, scoring, layout, rendering pipeline, and UI is inherited
- Distinct localStorage keys (`solitairra_*`) so the leaderboard, save, and suit prefs are independent
- Adds a third suit-style mode (`animals`) alongside the existing `laser` and `classic` modes
- Renames watermark (`Solitairra` on the felt) and card-back text (`Soli` / `Tairra`)
- How-to-Play section now generated dynamically from the currently selected suit style

## File Structure
- `Solitairra.html` - All screens (title, how-to-play, options, game, leaderboard, results)
- `css/style.css` - All styling (unchanged from SoloTerra)
- `js/cards.js` - Card data model (unchanged)
- `js/game.js` - Game state, move validation, scoring, win detection (unchanged)
- `js/renderer.js` - PixiJS rendering + custom Canvas 2D suit drawing; adds ANIMAL pip functions
- `js/animations.js` - Deal animations, card flights (unchanged)
- `js/textures.js` - Texture generation helpers (unchanged)
- `js/ui.js` - Screen management, drag/drop, options UI, preview rendering, **dynamic How-to-Play**
- `js/save.js` - Leaderboard API client with localStorage fallback; Solitairra-specific keys

## Suit Skin System
Options → "Card Suit Style" exposes two modes (Animals & Classic). Laser mode is kept in code but the button is commented out in `Solitairra.html`.

| Mode     | Diamonds | Hearts  | Spades  | Clubs     | Default | Visible in UI | Sub-variants? |
|----------|----------|---------|---------|-----------|---------|---------------|----------------|
| Animals  | Dolphins | Hares   | Spiders | Cubs      | **Yes** | Yes           | No (single look per animal) |
| Classic  | ♦ Diamonds | ♥ Hearts | ♠ Spades | ♣ Clubs | No    | Yes           | No |
| Laser    | Diodes   | Prisms  | Blades  | Combiners | No      | **Hidden** (code preserved) | Yes (scheme per suit + blade style) |

### Animal Pip Colors (single fixed scheme)
- Dolphins (diamonds): blue `#1565C0`. Realistic bottlenose silhouette leaping right — bulbous melon, short rostrum, mouth "smile", blowhole, eye, pectoral fin, dorsal fin, curved tail fluke. Counter-shaded (dark back, pale belly).
- Hares (hearts): saddle-brown `#8B4513`. **Face only** — tall ears with pink inner ear, round head, big eyes with glints, muzzle patch, nose, mouth, whiskers.
- Spiders (spades): black `#1a1a1a`, 8 bent legs radiating, cephalothorax with yellow + white eye dots.
- Cubs (clubs): dark brown `#3E2723`. **Face only** — round head with two round ears and lighter inner ears, muzzle patch, oval nose with highlight, eyes, mouth.

All animal pips:
- Normalize to `s = size / 20`, fit within roughly `14*s` visual footprint (same pip cell as Laser)
- Use the CUSTOM_PIP_LAYOUTS wide spread (2-3-3-2 for rank 10, etc.) — same as Laser
- Skip the corner Unicode symbol (the rank alone is shown in corners)
- No ground shadow — silhouettes render cleanly on the card surface
- Support the `flip` parameter (180° rotation)

### Renderer API additions (js/renderer.js)
- `ANIMAL_COLORS` — per-suit color map
- `isAnimalsSuit(suit)` — returns true when `suitSkins[suit] === 'animals'`
- `drawDolphinPip / drawHarePip / drawSpiderPip / drawCubPip(c, x, y, size, flip)` — individual pip drawers
- `drawAnimalPip(c, x, y, size, suit, flip)` — dispatcher
- `drawAnimalShadow(c, s)` — shared soft ellipse shadow under each animal
- Dispatch sites updated: `drawCornerPip`, `renderPips` loop, `renderFaceCard` center pip, `renderPlaceholder` foundation placeholder, `_renderSuitPip` (results screen)
- `getSuitColor(suit)` now returns `ANIMAL_COLORS[suit]` when suit is in animals mode

### UI changes (js/ui.js)
- Adds `btn-mode-animals` handler; `setAllSuitMode` supports `'animals'` / `'laser'` / `'classic'`.
- `btn-mode-laser` lookups are null-safe via `toggleMode()` helper so the deactivated button doesn't throw.
- `resetSuitDefaults()` defaults all four suits to `animals` mode; laser variant options hidden on load.
- `renderSuitPreview()` labels:
  - **Animals: plain `Dolphins` / `Hares` / `Spiders` / `Cubs` — no parentheticals.**
  - **Classic: plain `Diamonds` / `Hearts` / `Spades` / `Clubs` — no parentheticals.**
  - Laser: `Diodes (Diamonds)` etc. (two-line label, unchanged from SoloTerra).
  - Cards are pulled tight under the label for single-line modes (labelGap reduced).
- `SUIT_NAMES_BY_MODE` table + `suitNameFor()` / `currentSuitNames()` helpers used by both results-screen score formula and How-to-Play text.
- Score-formula on results screen now reads `Hares × 1 + Spiders × 2 + Cubs × 3` in Animals mode.

### How to Play (Dynamic)
- HTML holds an empty `<div id="rules-content">` — no static rule text
- `renderRules()` in `ui.js` populates it each time How-to-Play is opened
- Suit names in the rules (Goal card name, suit hierarchy line, scoring formula) swap based on the currently selected suit-style mode

## Save / Prefs / Leaderboard Keys
All three were renamed to avoid colliding with SoloTerra's data when both games are opened in the same browser:

| Purpose          | SoloTerra key              | Solitairra key              |
|------------------|----------------------------|------------------------------|
| Game state       | `soloterra_game_save`      | `solitairra_game_save`       |
| Suit preferences | `soloterra_suit_prefs`     | `solitairra_suit_prefs`      |
| Leaderboard      | `soloterra_leaderboard`    | `solitairra_leaderboard`     |
| Server API       | `/api/soloterra/leaderboard` | `/api/solitairra/leaderboard` (no server deployed yet; falls back to localStorage) |

## Game Rules (unchanged from SoloTerra)
- 40-card deck, 4 suits, ranks 1–10, 6 tableau columns
- Foundations build up by suit; tableau builds down by rank with suit hierarchy: diamonds > hearts > spades > clubs (i.e. Dolphins > Hares > Spiders > Cubs in the active mode's naming)
- Win: 10 of diamonds on its foundation
- Score: `saved_hearts × 1 + saved_spades × 2 + saved_clubs × 3`; 60 is a perfect game

## Title Screen
- Title text "Solitairra" is rendered over a stylized card-back shape. The shape is a `.game-title::before` pseudo-element centered at `top:50%; left:50%; transform: translate(-50%, -50%)` so the card always sits exactly centered on the heading regardless of viewport size. The old `.title-card-fan` / `.title-card` divs were removed.
- On landscape phones the pseudo-element is hidden (`display:none`).

## Card Back
- Big centered gold "S" monogram (46–48px Cinzel 900, gold-foil fill with subtle outer glow). Replaces the former two-line "Soli / Tairra" text.

## Cache Busting
`?v=3` on all CSS/JS tags in `Solitairra.html` (bumped after the pip redesign). Bump on each deploy.

## Deployment
- **Not yet deployed.** Currently GitHub-only per user request ("publish to GitHub for now, not Railway")
- If/when deployed alongside SoloTerra in `wbcgamez/public/`, the server would need a matching `/api/solitairra/leaderboard` endpoint and a separate data file at `data/solitairra-leaderboard.json` — not yet created
