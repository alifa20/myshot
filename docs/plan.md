# MyShot — implementation plan

A screenshot beautifier (Shots / Xnapper / CleanShot X class) delivered as **one static HTML
file**, opened in Safari and pinned to the Dock via *File > Add to Dock*. No build step, no
frameworks, no server, no native wrapper.

Deliverable: `index.html` at the repo root. That file is the whole product.

---

## 1. The part that decides whether this feels premium

Most "canvas screenshot framer" implementations fail in three specific places. The design below
is organised around not failing there.

1. **Preview vs. export divergence.** Canvas `shadowBlur` / `shadowOffsetX/Y` are *not* affected
   by the current transform — they are applied in the output (device) coordinate space. So
   `ctx.scale(2,2)` doubles the geometry but leaves the shadow the same absolute size. Any code
   that "just scales the context" produces an export whose shadow is half the weight of the
   preview. Handled explicitly in §3.
2. **Transparent screenshot corners.** macOS window captures ship with rounded, *transparent*
   corners. Casting a shadow by filling the rounded rect with an opaque colour puts black
   wedges in those corners. Handled by casting the shadow from the image's own alpha (§3.4).
3. **Flat, banded backgrounds.** A single `createLinearGradient` on a large canvas bands
   visibly in 8-bit sRGB and reads as "generic". Mesh gradients + a dithering grain layer fix
   both (§4).

## 2. Coordinate model

One canonical space — **composition space** — measured in CSS-ish pixels where the source image
is at its natural size. Everything (padding, radius, shadow geometry, layout) is expressed there.

```
render(ctx, compW, compH, scale)
```

`scale` is the only thing that differs between consumers:

| consumer      | scale                              |
| ------------- | ---------------------------------- |
| live preview  | `fitScale × devicePixelRatio`      |
| export 1×     | `1`                                |
| export 2×     | `2`                                |
| bg swatches   | whatever fits a 48×36 chip         |

There is exactly one renderer. Preview and export cannot diverge because they are the same
function called with a different number — *provided* every device-space quantity is multiplied
by `scale` by hand. That is the invariant the verification pass checks.

### Layout

```
S      = sqrt(iw · ih)                  // scale-invariant "size unit" of the image
pad    = round(paddingPct/100 · S)      // padding is proportional, not absolute
box    = (iw + 2·pad) × (ih + 2·pad)
ratio  ≠ auto → grow the deficient axis to hit the ratio (never crop, padding is a minimum)
imgX/Y = round((comp − img)/2)
```

Padding is a **percentage of `sqrt(area)`**, not a pixel count: 64px of padding around a 320px
thumbnail and around a 3200px retina capture are completely different pictures. `sqrt(area)`
behaves better than `min`/`max`/`avg` on extreme aspect ratios (a 2560×400 strip).

All composition-space values are integers, so at export scale 1 and 2 the screenshot lands on
whole device pixels and is blitted **1:1 with no resampling**.

## 3. Render pipeline

```
1. background      (composition space, ctx.scale(scale))
2. grain / dither  (device space, identity transform — grain is a device-pixel effect)
3. shadow layers   (device space, cast from the card's alpha, shape pushed off-canvas)
4. card            (device space, drawn 1:1 from a pre-rendered offscreen)
```

### 3.1 The card

The rounded-corner-clipped screenshot is rendered once into an offscreen canvas sized
`round(iw·scale) × round(ih·scale)` — i.e. at the *target device resolution*, sourced from the
original bitmap. Never render the card at 1× and upscale it for a 2× export; that throws away
the only real resolution in the picture.

Cached on `(imageToken, radius, scale)`. Padding, ratio and background changes don't invalidate
it, so dragging the padding slider costs one gradient fill and two blits.

### 3.2 Manual scaling of device-space quantities

`shadowBlur`, `shadowOffsetX/Y`, hairline `lineWidth`, and the card's draw position are all
multiplied by `scale` explicitly. Corner radius is multiplied by `scale` when building the card.

### 3.3 Layered shadow

One canvas shadow reads as "drop shadow on white". Real product shots use two: a broad ambient
occlusion plus a tight contact shadow. Both scale with `S` so the shadow stays proportional to
the image, and with the shadow slider `t`:

```
ambient:  blur 70·S·t   dy 34·S·t   α 0.30·t
contact:  blur 16·S·t   dy  7·S·t   α 0.16·t
```

### 3.4 Shadow-only drawing (the off-canvas trick)

To cast a shadow that respects the screenshot's alpha without painting the screenshot twice:
draw the card at `x − OFF` with `shadowOffsetX = OFF + dx·scale`. The *shape* lands outside the
canvas and is clipped away; only its shadow lands in frame. `OFF` is bounded by the canvas width
plus the blur radius, not some arbitrary 100000 that risks blur-buffer overflow.

## 4. Backgrounds

Ten presets — 5 mesh, 3 linear, 2 solid — plus a custom colour.

A **mesh** is a base fill plus N radial blobs, each a radial gradient with a smoothstep-ish
alpha falloff (`1, .82, .5, .18, 0`) instead of a linear ramp, composited `screen` on dark bases
and `source-over` on light ones. This is what gives the Xnapper/CleanShot look rather than a
two-stop linear gradient.

**Grain**: a cached 160×160 tile of low-alpha noise, tiled in device space at ~2.5% effective
alpha over the background only (never over the screenshot). It dithers away gradient banding and
adds the slight texture that reads as "designed".

Swatches are drawn with the *same* `paintBackground` into small canvases, so a swatch is never a
CSS approximation of the real thing.

## 5. Safari specifics

- **Clipboard.** `navigator.clipboard.write` needs a secure context; `file://` may block it.
  Copy passes a **Promise** to `new ClipboardItem({'image/png': blobPromise})` — Safari requires
  the write to be issued inside the user-gesture task, and the promise form is the sanctioned
  way to do that while PNG encoding happens async. On any failure: a visible error plus a
  fallback sheet containing the PNG as an `<img>` (natively draggable and right-clickable in
  Safari) and a Download button. Copy never fails silently.
- **`ctx.roundRect`** is not assumed; the path is hand-rolled with `arcTo`.
- **Canvas limits.** Export is clamped to a max side / max area; if the requested multiplier
  doesn't fit, it is reduced and the user is told, rather than getting a blank canvas.
- **`-webkit-backdrop-filter`** for the toolbar vibrancy; `color-scheme` + `prefers-color-scheme`
  so the app follows system appearance like a native app.

## 6. Product decisions

- **Demo composition on first load.** Rather than an empty dashed rectangle, the app boots with
  a procedurally drawn mock app window already beautified, so the controls are live and the
  value proposition is visible in the first second. A badge marks it as a demo; dropping,
  pasting or opening a file replaces it.
- **Export defaults to 1×.** At 1× the screenshot is pixel-exact; 2× upsamples the only real
  pixels in the frame and only sharpens the synthetic parts (corners, shadow, gradient). 1× is
  the honest default; 2× is one click away.
- **Control set stays exactly what the brief asks for** — padding, radius, shadow, background,
  ratio. Shadow is a single 0–100 slider driving a curated two-layer shadow rather than three
  raw sliders; the point is a good default, not a shadow editor.
- Filename `screenshot-YYYYMMDD-HHMMSS.png`.

## 7. Performance

The first working version ran the preview at **4 fps** while dragging a slider — not "live" by
any definition. Profiling a single 2144×1312 preview frame on an M5 Pro (Metal-backed canvas)
located the cost precisely:

| phase | cost |
| --- | --- |
| clear + solid fill | 0.2 ms |
| one radial blob, full canvas | 7.2 ms |
| same with `screen` blending | 7.3 ms (blending is free) |
| **full frame, solid background, no grain** | **64.9 ms** |
| full frame, mesh background | 101.9 ms |

The mesh was never the problem. ~65 ms of every frame was the **two large Gaussian shadow
blurs** — `shadowBlur` scales with the image, so a default frame was blurring with σ ≈ 43 px
over the whole card, twice, every frame.

Two caches fix it:

- **Background layer** (including grain) at device resolution, keyed on preset/colour and
  canvas size. Radius and shadow drags never repaint it.
- **Shadow sprite**: both layers pre-blurred into one sprite, keyed on everything except
  position, so padding drags and ratio changes just re-blit it.

The sprite is rendered at **reduced resolution**, which is lossless rather than a quality
trade: a blurred shadow is band-limited by its own blur, so sampling it at a resolution matched
to that blur discards nothing. Capping the blur at 24 px of sprite space bounds the cost — and
because sprite scale is inversely proportional to blur, the sprite lands at the *same absolute
resolution* at 1× and 2×, so parity is preserved by construction rather than by luck.

Result: **4 fps → 120 fps** (the display's refresh cap), on every slider, for every image size
from the 1440×900 demo up to a 5120×2880 capture. Worst frame ≈ 11 ms.

## 8. Verification

A debug handle (`window.__myshot`) exposes state, the renderer and a `debugOpts.grain` switch —
grain is deliberately device-space and therefore the one part of the image that is *not*
scale-invariant, so it is turned off to compare renders. Run against the real page in Chrome:

1. **Console clean.** (One `file:` origin error appears on a fresh `file://` navigation; a page
   containing nothing but a `<p>` reproduces it, so it is Chrome's favicon fetch, not this app.)
2. **Scale-invariance, instrumented.** The 2D context is wrapped in a Proxy that records every
   property assignment during a render at scale 1 and at scale 2. `shadowBlur`, `shadowOffsetY`
   and `lineWidth` all come out at **exactly 2.0×**. This is the test that would catch a
   device-space quantity left unscaled (§1.1).
3. **Scale-invariance, pixel-wise.** Box-downsample a 2× render to 1× and diff: mean 0.31/255.
   Classified by region, the divergence in the **background and shadow is max 5.8/255** — the
   synthetic image is effectively identical at both scales. The larger interior outliers
   (max 38) are the source bitmap genuinely being interpolated at 2×, which is inherent to
   upscaling and is exactly why 1× is the default.
4. **Alpha-aware shadow.** An image with a cleared corner renders that corner as background
   (96,99,216), not black — no shadow wedges in a macOS window capture's transparent corners.
5. **Geometry.** All five ratios land within rounding of their target, padding is never eaten
   and the image is never cropped. A 2560×300 strip gets 70 px of padding and a 400×2000 column
   72 px — the `sqrt(area)` model holds at both extremes.
6. **Export.** 1622×1082 at 1×, 3244×2164 at 2×, and the encoded PNG decodes to exactly those.
   A 7000×4000 source correctly caps 2× to 1.04× and says so in the UI.
7. **Colour fidelity.** Solid presets and custom colours are exact — Paper `#F2F2F5`, Ink
   `#0B0C10`, custom white `255,255,255`. Grain is skipped on solids: measurement showed it
   otherwise pulled every colour toward mid-grey (Paper 243 → 239, white → 251), and a solid
   fill has no banding to dither in the first place.
8. **Input.** Drop, paste and the picker each replace the image and never stack; the nested
   `dragenter`/`dragleave` counter doesn't strand the overlay; a dropped `.txt` is rejected with
   a visible message and leaves the current image alone.
9. **Persistence.** Every control round-trips a reload; the image does not. Hostile storage
   (wrong types, out-of-range numbers, non-JSON) falls back to defaults with a clean console.
   An idle page performs **zero** writes — settings persist only on real interaction.
10. **Keyboard.** ⌘S emits `screenshot-20260731-162538.png`; ⌘C copies; neither steals a
    real text selection.
11. **Clipboard failure path.** With `clipboard.write` stubbed to reject, Copy surfaces an error
    toast *and* opens the sheet holding the true 1622×1082 PNG as a draggable `<img>`. It never
    fails silently.
