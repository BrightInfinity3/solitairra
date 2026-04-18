# Face Card Rendering Reference (A / J / Q / K)

This document describes how **Solitairra** renders face-card ranks — the letters A, J, Q, K — so the same look can be reproduced in another Canvas 2D card game (e.g. the `30` game).

Source of truth: `js/renderer.js`, function `renderFaceCard(c, area, rank, suit)` (near line 3237). Gold-foil helper lives in `js/textures.js`, `Textures.goldFoilGradient(ctx, x, y, w, h)`.

The rendering is a **5-layer sandwich**: frame → chess watermark → drop-shadow of the letter → gold-foil overlay of the letter → solid suit-coloured letter → suit pip below → four corner flourishes. Drawing in this exact order is what gives the letter its embossed, foil-stamped look.

---

## Card dimensions & context

Same as the 2-10 pip cards — draw to an offscreen canvas at `TEX_SCALE = 2` for crisp display:

```
CARD_W = 70          // logical width (px)
CARD_H = 100         // logical height (px)
CARD_R = 7           // corner radius
TEX_SCALE = 2        // offscreen render scale
```

All drawing uses logical (1×) coordinates after `ctx.scale(TEX_SCALE, TEX_SCALE)`.

The `area` rectangle passed to `renderFaceCard` is the interior pip area — the same rect renderPips uses. In Solitairra it's:

```js
var pipMarginX = 14;
var pipMarginY = 18;
var area = {
  x: pipMarginX,
  y: pipMarginY,
  w: CARD_W - pipMarginX * 2,    // 42
  h: CARD_H - pipMarginY * 2     // 64
};
```

Centre of the pip area (used for the letter):
```js
var cx = area.x + area.w / 2;    // 35
var cy = area.y + area.h / 2;    // 50
```

---

## Step 1 — Inner decorative frame

A thin rounded-rect stroke inset from the pip area, in very-faint gold. Purely decorative — it frames the face-card artwork:

```js
c.save();
var frameInset = 2;
var frameR = 3;
roundRect(c, area.x + frameInset, area.y + frameInset,
  area.w - frameInset * 2, area.h - frameInset * 2, frameR);
c.strokeStyle = '#c9952a';
c.globalAlpha = 0.2;
c.lineWidth = 0.6;
c.stroke();
c.restore();
```

`roundRect(c, x, y, w, h, r)` is the standard rounded-rectangle path helper (see existing renderer for implementation).

---

## Step 2 — Vertical alignment constants

All four face-card ranks must sit on the **same visual baseline** so they line up across the deck. The tricky one is **Q** — the descender tail on its bottom-right pushes the visual centre up when you use `textBaseline: 'middle'`, so we nudge Q down by 2 px to realign its O-body with A/J/K:

```js
var rankCenterY = cy - 4;                               // cy from above, minus 4
var qDescenderOffset = (rank === 'Q') ? 2 : 0;
var rankY = rankCenterY + qDescenderOffset;
```

`rankY` is the y-coordinate of the letter's visual middle — used in all subsequent letter draws.

---

## Step 3 — Chess-piece watermark (behind the letter)

Each face-card has a faint chess-piece symbol drawn behind the letter as a watermark. Unicode code points:

| Rank | Chess symbol | Unicode | Notes |
|------|--------------|---------|-------|
| K | `♚` | `\u265A` | Black king |
| Q | `♛` | `\u265B` | Black queen |
| J | `♘` | `\u2658` | **White** knight (outline reads as a knight at small size — the solid-black knight is too heavy) |
| A | `✦` | `\u2726` | Black four-pointed star — stands in for an ornament |

```js
c.save();
c.textAlign = 'center';
c.textBaseline = 'middle';
var chessSym = rank === 'K' ? '\u265A'
             : rank === 'Q' ? '\u265B'
             : rank === 'J' ? '\u2658'
             :                 '\u2726';   // A
c.font = 'bold 30px serif';
c.fillStyle = color;                       // suit colour
c.globalAlpha = 0.08;                      // very faint — almost subliminal
c.fillText(chessSym, cx, rankY);
c.globalAlpha = 1;
c.restore();
```

`color` is `getSuitColor(suit)` — the same colour used for 2-10 pips.

---

## Step 4 — The letter itself (three-pass gilded stamp)

The letter is drawn in **three passes**:

1. **Shadow** offset 1 px down-right, opaque black at 15% alpha.
2. **Gold-foil overlay** at 30% alpha — gives the letter a sheen.
3. **Solid suit colour** at 100% alpha — the main readable fill.

Because of the alpha values, all three passes blend to produce a letter that has a dark drop-shadow, a metallic gold highlight, and a crisp suit-coloured body.

### Font choice

- **A** and **Q** — `Cinzel` (a decorative Roman-style display font with strong serifs). This makes the A and Q look ornate. Fallback `Georgia, serif`.
- **J** and **K** — `Georgia, serif`. A classic book face with clean letterforms, keeping J/K from feeling too showy next to A/Q.

```js
var faceFont = (rank === 'A' || rank === 'Q')
  ? '900 28px Cinzel, Georgia, serif'
  : '900 28px Georgia, serif';
```

`900` weight + `28px` size.

### Full three-pass draw

```js
c.save();
c.font = faceFont;
c.textAlign = 'center';
c.textBaseline = 'middle';

// Pass 1: drop shadow (1px down-right, dark, 15% alpha)
c.fillStyle = 'rgba(0, 0, 0, 0.15)';
c.fillText(rank, cx + 1, rankY + 1);

// Pass 2: gold-foil overlay (30% alpha)
var goldGrad = Textures.goldFoilGradient(c, cx - 14, rankY - 14, 28, 28);
c.fillStyle = goldGrad;
c.globalAlpha = 0.3;
c.fillText(rank, cx, rankY);

// Pass 3: solid suit colour (full alpha)
c.globalAlpha = 1;
c.fillStyle = color;
c.fillText(rank, cx, rankY);

c.restore();
```

### Gold-foil gradient

Defined once in `Textures` and reused everywhere gold appears. It's a diagonal linear gradient with five stops that read as "warm gold foil":

```js
function goldFoilGradient(ctx, x, y, w, h) {
  var g = ctx.createLinearGradient(x, y, x + w, y + h);
  g.addColorStop(0,    '#d4a849');   // light warm gold
  g.addColorStop(0.25, '#f5d78e');   // bright cream-gold
  g.addColorStop(0.5,  '#c9952a');   // darker warm gold
  g.addColorStop(0.75, '#f0d070');   // light gold highlight
  g.addColorStop(1,    '#b8860b');   // deep old gold
  return g;
}
```

The call `Textures.goldFoilGradient(c, cx - 14, rankY - 14, 28, 28)` means the gradient spans a 28×28 box centred on the letter — roughly matching the letter's bounding box so the sheen falls across the glyph nicely.

---

## Step 5 — Suit pip below the letter

A small suit pip is drawn centred horizontally, 24 px below `rankCenterY`:

```js
var suitPipY = rankCenterY + 24;
```

Pip size **should match the 2-10 pip size for the same suit** so cards feel consistent. For standard classic mode the call is:

```js
c.save();
c.font = '20px serif';                    // matches 2-10 classic pip font size
c.textAlign = 'center';
c.textBaseline = 'middle';
c.fillStyle = 'rgba(0, 0, 0, 0.1)';       // subtle drop shadow
c.fillText(sym, cx + 0.5, suitPipY + 0.5);
c.fillStyle = color;
c.fillText(sym, cx, suitPipY);
c.restore();
```

Where `sym = SUIT_SYM[suit]` is the Unicode suit glyph (`♥ ♦ ♠ ♣`) and `color = getSuitColor(suit)`.

(In Solitairra, when a "laser" or "animals" skin is active, the code calls the suit-specific pip-drawing function instead of `fillText`. For a classic-only card game like `30`, the above is all you need.)

---

## Step 6 — Four corner flourishes

Short quarter-circle gold curls in each inner corner, inside the frame. Purely decorative:

```js
c.save();
c.strokeStyle = '#c9952a';
c.globalAlpha = 0.18;
c.lineWidth = 0.8;

// top-left
c.beginPath();
c.moveTo(area.x + 2, area.y + 12);
c.quadraticCurveTo(area.x + 2, area.y + 2, area.x + 12, area.y + 2);
c.stroke();

// top-right
c.beginPath();
c.moveTo(area.x + area.w - 2, area.y + 12);
c.quadraticCurveTo(area.x + area.w - 2, area.y + 2, area.x + area.w - 12, area.y + 2);
c.stroke();

// bottom-left
c.beginPath();
c.moveTo(area.x + 2, area.y + area.h - 12);
c.quadraticCurveTo(area.x + 2, area.y + area.h - 2, area.x + 12, area.y + area.h - 2);
c.stroke();

// bottom-right
c.beginPath();
c.moveTo(area.x + area.w - 2, area.y + area.h - 12);
c.quadraticCurveTo(area.x + area.w - 2, area.y + area.h - 2, area.x + area.w - 12, area.y + area.h - 2);
c.stroke();

c.restore();
```

Each flourish is a 10-px quarter-circle, offset 2 px inside the frame.

---

## Full drawable reference — complete function

Minimal self-contained version (drop-in for a classic-suits game like `30`):

```js
function renderFaceCard(c, area, rank, suit) {
  var SUIT_SYM    = { hearts:'\u2665', diamonds:'\u2666', clubs:'\u2663', spades:'\u2660' };
  var SUIT_COLORS = { hearts:'#b71c1c', diamonds:'#b71c1c', clubs:'#1a1a1a', spades:'#1a1a1a' };
  var sym   = SUIT_SYM[suit];
  var color = SUIT_COLORS[suit];
  var cx    = area.x + area.w / 2;
  var cy    = area.y + area.h / 2;

  // 1. Inner frame
  c.save();
  roundRect(c, area.x + 2, area.y + 2, area.w - 4, area.h - 4, 3);
  c.strokeStyle = '#c9952a';
  c.globalAlpha = 0.2;
  c.lineWidth = 0.6;
  c.stroke();
  c.restore();

  // 2. Vertical position
  var rankCenterY = cy - 4;
  var rankY = rankCenterY + ((rank === 'Q') ? 2 : 0);

  // 3. Chess-piece watermark
  c.save();
  c.textAlign = 'center';
  c.textBaseline = 'middle';
  var chessSym = rank === 'K' ? '\u265A'
               : rank === 'Q' ? '\u265B'
               : rank === 'J' ? '\u2658'
               :                 '\u2726';
  c.font = 'bold 30px serif';
  c.fillStyle = color;
  c.globalAlpha = 0.08;
  c.fillText(chessSym, cx, rankY);
  c.globalAlpha = 1;
  c.restore();

  // 4. Letter: shadow → gold foil → suit colour
  c.save();
  var faceFont = (rank === 'A' || rank === 'Q')
    ? '900 28px Cinzel, Georgia, serif'
    : '900 28px Georgia, serif';
  c.font = faceFont;
  c.textAlign = 'center';
  c.textBaseline = 'middle';
  c.fillStyle = 'rgba(0,0,0,0.15)';
  c.fillText(rank, cx + 1, rankY + 1);
  var g = c.createLinearGradient(cx - 14, rankY - 14, cx + 14, rankY + 14);
  g.addColorStop(0,    '#d4a849');
  g.addColorStop(0.25, '#f5d78e');
  g.addColorStop(0.5,  '#c9952a');
  g.addColorStop(0.75, '#f0d070');
  g.addColorStop(1,    '#b8860b');
  c.fillStyle = g;
  c.globalAlpha = 0.3;
  c.fillText(rank, cx, rankY);
  c.globalAlpha = 1;
  c.fillStyle = color;
  c.fillText(rank, cx, rankY);
  c.restore();

  // 5. Suit pip below
  var suitPipY = rankCenterY + 24;
  c.save();
  c.font = '20px serif';
  c.textAlign = 'center';
  c.textBaseline = 'middle';
  c.fillStyle = 'rgba(0,0,0,0.1)';
  c.fillText(sym, cx + 0.5, suitPipY + 0.5);
  c.fillStyle = color;
  c.fillText(sym, cx, suitPipY);
  c.restore();

  // 6. Four corner flourishes
  c.save();
  c.strokeStyle = '#c9952a';
  c.globalAlpha = 0.18;
  c.lineWidth = 0.8;
  c.beginPath(); c.moveTo(area.x + 2,          area.y + 12);
                 c.quadraticCurveTo(area.x + 2, area.y + 2, area.x + 12, area.y + 2); c.stroke();
  c.beginPath(); c.moveTo(area.x + area.w - 2, area.y + 12);
                 c.quadraticCurveTo(area.x + area.w - 2, area.y + 2, area.x + area.w - 12, area.y + 2); c.stroke();
  c.beginPath(); c.moveTo(area.x + 2,          area.y + area.h - 12);
                 c.quadraticCurveTo(area.x + 2, area.y + area.h - 2, area.x + 12, area.y + area.h - 2); c.stroke();
  c.beginPath(); c.moveTo(area.x + area.w - 2, area.y + area.h - 12);
                 c.quadraticCurveTo(area.x + area.w - 2, area.y + area.h - 2, area.x + area.w - 12, area.y + area.h - 2); c.stroke();
  c.restore();
}
```

---

## Font loading

The decorative Cinzel face needs to be loaded before A/Q cards are drawn. Solitairra does this in the HTML `<head>`:

```html
<link rel="preconnect" href="https://fonts.googleapis.com">
<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
<link href="https://fonts.googleapis.com/css2?family=Cinzel:wght@400;700;900&family=Crimson+Text:ital,wght@0,400;0,600;0,700;1,400&display=swap" rel="stylesheet">
```

If cards are rendered before the font loads, the browser will substitute Georgia. To guarantee Cinzel is available when cards render, the card-viewer uses `document.fonts.load()` before drawing:

```js
Promise.all([
  document.fonts.load('bold 28px Cinzel'),
  document.fonts.load('900 28px Cinzel'),
  document.fonts.load('bold 11px Cinzel')
]).then(function () {
  // draw cards here
});
```

In the main game, cards are rendered inside `buildCardTextures()` after PixiJS initialises, which usually happens well after fonts finish loading. The game uses Georgia as a fallback for J/K anyway, so the only ranks that visibly depend on Cinzel are A and Q.

---

## Corner rank display (top-left / bottom-right of card)

Separate from `renderFaceCard` — the rank letter in the two small corners of the card is drawn by `renderCardToImage` itself (not inside `renderFaceCard`). Font rules:

```js
// Use Georgia for numeric ranks ("1" in Cinzel looks like capital I) AND for J, K.
// A and Q use Cinzel with Georgia fallback.
var isNumeric = !isNaN(parseInt(rank));
var useGeorgia = isNumeric || rank === 'J' || rank === 'K';
var rankFontSize = (numericRank === 1) ? 22 : 11;   // 1-cards get a doubled corner
var rankFont = useGeorgia
  ? 'bold ' + rankFontSize + 'px Georgia, serif'
  : 'bold ' + rankFontSize + 'px Cinzel, Georgia, serif';
```

The corner rank is drawn in plain suit colour, with a 0.5-px drop shadow — no gold-foil treatment (that's reserved for the big centre letter on face cards). This keeps the corners legible when cards overlap in a tableau.

---

## Visual summary

- **J**: bold Georgia letter, white knight watermark, gold-foil sheen, suit-coloured body, small diamond/heart/spade/club pip 24 px below, faint gold frame + corner flourishes.
- **K**: bold Georgia letter, black king watermark, otherwise identical to J.
- **Q**: 900-weight Cinzel letter (more decorative), black queen watermark, nudged down 2 px so the O-body aligns with A/J/K.
- **A**: 900-weight Cinzel letter, four-pointed star watermark, otherwise identical to Q.

All four ranks share the **five-layer sandwich**: frame → chess watermark → letter shadow → gold foil → solid colour → suit pip → corner flourishes.
