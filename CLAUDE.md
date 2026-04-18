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
Three modes in Options → "Card Suit Style":

| Mode     | Diamonds | Hearts  | Spades  | Clubs     | Default | Sub-variants? |
|----------|----------|---------|---------|-----------|---------|----------------|
| Animals  | Dolphins | Hares   | Spiders | Cubs      | **Yes** | No (single look per animal) |
| Laser    | Diodes   | Prisms  | Blades  | Combiners | No      | Yes (scheme per suit + blade style) |
| Classic  | ♦ Diamonds | ♥ Hearts | ♠ Spades | ♣ Clubs | No    | No |

### Animal Pip Colors (single fixed scheme)
- Dolphins (diamonds): blue `#1565C0` with lighter/darker gradient, belly counter-shading, eye glint
- Hares (hearts): saddle-brown `#8B4513`, upright ears with pink inner ear, cotton-puff tail, front paw
- Spiders (spades): black `#1a1a1a`, 8 bent legs radiating, cephalothorax with yellow + white eye dots
- Cubs (clubs): dark brown `#3E2723`, round body with separate head + two round ears, lighter muzzle and inner ears

All animal pips:
- Normalize to `s = size / 20`, fit within roughly `14*s` visual footprint (same pip cell as Laser)
- Use the CUSTOM_PIP_LAYOUTS wide spread (2-3-3-2 for rank 10, etc.) — same as Laser
- Skip the corner Unicode symbol (the rank alone is shown in corners)
- Drop shadow beneath each pip via `drawAnimalShadow()`
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
- Adds `btn-mode-animals` handler; `setAllSuitMode` now supports `'animals'`
- `resetSuitDefaults()` defaults all four suits to `animals` mode; laser variant options hidden on load
- `renderSuitPreview()` labels:
  - Animals: `Dolphins (Diamonds)` etc.
  - Laser: `Diodes (Diamonds)` etc. (unchanged from SoloTerra)
  - **Classic: plain `Diamonds` / `Hearts` / `Spades` / `Clubs` — no parentheticals**
- `SUIT_NAMES_BY_MODE` table + `suitNameFor()` / `currentSuitNames()` helpers used by both results-screen score formula and How-to-Play text
- Score-formula on results screen now reads `Hares × 1 + Spiders × 2 + Cubs × 3` in Animals mode

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

## Cache Busting
Reset to `?v=1` on all CSS/JS script tags in `Solitairra.html`. Bump on each deploy.

## Deployment
- **Not yet deployed.** Currently GitHub-only per user request ("publish to GitHub for now, not Railway")
- If/when deployed alongside SoloTerra in `wbcgamez/public/`, the server would need a matching `/api/solitairra/leaderboard` endpoint and a separate data file at `data/solitairra-leaderboard.json` — not yet created
