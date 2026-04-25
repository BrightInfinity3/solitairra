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
- `card-viewer.html` - Reference page rendering all 14 ranks (1, 2-10, A, J, Q, K) × 4 suits across every mode (Animals default, Classic, all Laser variants). A is grouped with the face cards at the end of each row. Useful when iterating on pip design.

## Suit Skin System
Options → "Card Suit Style" exposes two modes (Animals & Classic). Laser mode is kept in code but the button is commented out in `Solitairra.html`.

| Mode     | Diamonds | Hearts  | Spades  | Clubs     | Default | Visible in UI | Sub-variants? |
|----------|----------|---------|---------|-----------|---------|---------------|----------------|
| Animals  | Dolphins | Hares   | Spiders | Cubs      | **Yes** | Yes           | No (single look per animal) |
| Classic  | ♦ Diamonds | ♥ Hearts | ♠ Spades | ♣ Clubs | No    | Yes           | No |
| Laser    | Diodes   | Prisms  | Blades  | Combiners | No      | **Hidden** (code preserved) | Yes (scheme per suit + blade style) |

### Animal Pip Colors
- Dolphins (diamonds): **V11 "Iconic leaping (emoji style)"** is the chosen design — emoji-inspired leaping bottlenose with domed melon, falcate dorsal, pectoral fin, horizontal tail flukes, counter-shaded light-blue body (medium blue back → white belly), upturned smile just under the eye. Defined in `DOLPHIN_VARIANTS[10]`; `DEFAULT_DOLPHIN_VARIANT = 10`. The array still contains the full set of alternate concepts (baby face, swoosh, infinity, Monogram D, dotwork, postage stamp, wave jumper, pixel 8-bit, Celtic knot, Art Nouveau, heraldic, minimalist, splashing-leap) and the card-viewer cycles through them for reference. API: `Renderer.setDolphinVariant(idx|null)` — passing `null` reverts to V11.
- Hares (hearts): illustration uses saddle-brown `#8B4513`; **rank text font is pink `#E91E63`**. Face only, smiling. Tall ears sit above the head (bottoms tangent with the face top). Head rx 5.082*s, ry 5.566*s. Ear half-width 1.15*s, same centre-to-centre spacing. Muzzle patch 25% larger (rx 2.31*s, ry 1.625*s); smile stroke 15% thicker (0.368*s).
- Spiders (spades): black `#1a1a1a`, 8 bent legs radiating, cephalothorax with yellow + white eye dots.
- Cubs (clubs): dark brown `#3E2723`. Face only, smiling. Two round ears sit at the upper-side head corners and are drawn before the head; the fully-opaque head then covers their inner halves so only the outer crescent shows — classic teddy-bear silhouette.

All animal pips:
- Normalize to `s = size / 20`, fit within roughly `14*s` visual footprint (same pip cell as Laser).
- **Per-suit size boosts for counts 2-10**: Animals get a baseline +10%, then per-suit: Hares 25% · Dolphins 10% · Spiders 10% · Cubs 25% × 1.10 × 1.10 = 51.25%. (Effective totals: Hares 37.5%, Dolphins/Spiders 21%, Cubs 66.375%.) **The same +10% baseline applies to animal face-card (A/J/Q/K) centre pips**, so face-card pips track 2-10 pip sizing for their suit.
- **Rank-1 pip sizes diverge by mode**: Laser 48.875 (57.5 × 0.85), Animals 72.7375 (57.5 × 1.15 × 1.10), Classic 61.09375 (71.875 × 0.85). Underlying progression was 32 → 40 → 46 → 57.5 (animals/laser) and 40 → 50 → 57.5 → 71.875 (classic) before the latest mode-specific tweaks.
- **Empty-stack (foundation placeholder) pip size**: Animals use 69.12 (48 × 1.20 × 1.20, two compounded +20% boosts); Laser and Classic stay at 48.
- **Face-card centre pip matches the 2-10 pip size for its suit** (diamonds/spades 16, hearts 20, clubs 22; classic 20).
- Each animal pip function **translates the drawing so the visual centre (top-of-pip ↔ bottom-of-pip midpoint) sits on the pip's origin**, so rows of pips have equal top and bottom margins on the card. Examples: hare translates by `(0, 2.1*s)` to compensate for the ears extending far above the face; dolphin translates by `(0, -1.8*s)` pre-rotation to compensate for the body's rotated centre-of-mass landing below the origin (without it, the rank-1 dolphin and the empty-foundation dolphin sit visibly low). Centering correctness is empirically measurable: render the pip on a transparent canvas at any size, and `topGap` should equal `bottomGap` to within a pixel.
- Use the CUSTOM_PIP_LAYOUTS wide spread (2-3-3-2 for rank 10, etc.) — same as Laser.
- Skip the corner Unicode symbol (the rank alone is shown in corners).
- No ground shadow — silhouettes render cleanly on the card surface.
- Support the `flip` parameter (180° rotation).

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
- `.title-stack` is a CSS-grid container with two children placed in the same grid cell: `.title-card-bg` (the card-back shape) and `.game-title` (the text). `place-items: center` handles the centering automatically — the text is always perfectly centered on the card regardless of viewport.
- The text sits **in front of** the card: `.game-title` has `z-index: 1` and is emitted after the card in document order.
- On landscape phones the card is hidden (`.title-card-bg { display: none }`).

## Card Back
- Big centered gold "S" monogram (46–48px Cinzel 900, gold-foil fill with subtle outer glow). Replaces the former two-line "Soli / Tairra" text.

## Foundation Placeholder — Translucency Handling
An empty foundation stack draws its suit pip at ~55% opacity to look "ghosted". Animal pips are composed of multiple overlapping sub-shapes (ears behind head, body behind arm), so naïvely setting `globalAlpha = 0.55` causes the under-layers to bleed through the upper ones (the head looks translucent, the pectoral fin looks translucent).

Fix: the animal placeholder branch renders the pip **opaquely to an offscreen canvas first**, then blits that single image at `globalAlpha = 0.55`. Inner shapes cover correctly, then the whole composite is made semi-transparent.

```js
var off = document.createElement('canvas');
off.width = CARD_W; off.height = CARD_H;
drawAnimalPip(off.getContext('2d'), CARD_W/2, CARD_H/2, phPipSize, suit, false);
c.save();
c.globalAlpha = 0.55;
c.drawImage(off, 0, 0);
c.restore();
```

## Cache Busting
`?v=18` on all CSS/JS tags in `Solitairra.html` and `card-viewer.html`. Bump on each deploy.

## Deployment
- **Live on Ladybug Gamez** (Railway). Hosted alongside Laserman / Lango / 30 — NOT on `wbcgamez` where SoloTerra lives.
- Source repo: `BrightInfinity3/solitairra` at `C:\Users\MK\MKCC\Solitairra\`
- Deploy repo: `BrightInfinity3/ladybug-gamez` at `C:\Users\MK\MKCC\ladybug-gamez\` — Railway auto-deploys on push to `main`
- **Deploy workflow**: copy `Solitairra.html → ladybug-gamez/public/solitairra/index.html`, plus `css/`, `js/`, `card-viewer.html`. Bump `?v=N` cache-bust in both source and deploy HTMLs. Commit + push both repos.
- Leaderboard API: `/api/solitairra/leaderboard` (GET/POST/DELETE) defined in `ladybug-gamez/server.js`. Persists to `$RAILWAY_VOLUME_MOUNT_PATH/solitairra/solitairra-leaderboard.json` — **the volume is mounted and confirmed persistent** (verified: leaderboard survived a redeploy). Falls back to `./data/` locally without the volume.
- `data/` is gitignored on the deploy repo (matches wbcgamez convention).
