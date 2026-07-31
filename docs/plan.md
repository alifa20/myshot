# Myshot — build plan (prompt v1)

Target: `docs/prompt.md`. Deliverable: **one static `index.html`**, vanilla JS + canvas, no build
step, no frameworks, no network requests at runtime.

---

## 1. Scope calls

**In scope (the brief, executed to a high standard)**

| Requirement | Decision |
| --- | --- |
| Input | drag-and-drop, paste (`⌘V`), file picker. A new image *replaces* the current one. |
| Controls | padding, corner radius, shadow, background, canvas ratio — all live, no Apply. |
| Backgrounds | 12 presets (6 mesh, 3 gradient, 3 solid) + transparent + custom colour. |
| Ratios | Auto · 16:9 · 4:3 · 1:1 · 2:1 (X/Twitter). "Twitter-size" is listed separately from 16:9 in the brief, so it must be a *different* ratio → 2:1, the `summary_large_image` shape. |
| Export | PNG at 1× / 2×, one click. Copy to clipboard. |
| Offline | zero external requests: no webfonts, no CDN, no analytics. Fonts come from the platform stack. |

**Deliberately added** (small, natural companions that the brief's "make it premium" implies):

- Retina-crisp preview via `devicePixelRatio`, decoupled from the 1×/2× export multiplier.
- `⌘S` export / `⌘C` copy.
- Settings persisted in `localStorage` (never the image).
- Film-grain overlay (kills gradient banding — a real problem with 8-bit canvas gradients).
- Edge-highlight hairline on the artwork (the thing that makes a screenshot "sit" on a dark background).
- A generated sample screenshot so the page demonstrates itself on first open.

**Deliberately excluded**: 3D/perspective tilt, device & browser frames, watermarking, redaction,
multi-image layouts, image backgrounds. The prompt already narrowed the Shots/Xnapper feature set;
widening it would dilute the part that has to be excellent.

---

## 2. Aesthetic direction — "Darkroom"

The app is a **precision instrument**; the artwork on the stage is the print. Everything in the
chrome recedes so the export is the only thing with colour.

- **Surface** — warm graphite `#0E0E10` → panels `#141416`, hairlines at 6–9% white, a soft
  safelight glow behind the stage. Full-bleed film grain over the chrome at ~3.5% opacity.
- **Accent** — a single warm amber `#E8A05C` (safelight). No purple-on-white SaaS gradient anywhere.
- **Type** — platform stack used deliberately, not lazily: `ui-serif` (New York on macOS) for the
  wordmark and the empty-state line, `ui-sans-serif` (SF) for UI at small sizes with tight tracking,
  `ui-monospace` (SF Mono) with tabular figures for every number. Zero network cost, and on the
  target platform it reads as a native Mac utility — which is the product concept.
- **Detail that makes it memorable** — corner **crop marks** framing the stage, the way a print
  proof is trimmed. Plus an engraved instrument-panel inspector on the right (Mac-native side).
- **Dark only, on purpose.** Image editors are dark (Lightroom, Figma, Photoshop) because a neutral
  dark surround stops the chrome from biasing your read of the artwork's colour. A light mode would
  undercut both the concept and the judgement the tool exists to support.
- **Motion** — one orchestrated entrance (staggered rise), then only functional micro-interactions.
  All of it behind `prefers-reduced-motion`.

---

## 3. Render pipeline

One pure function paints everything, at any scale. Preview and export call the *same* function, so
they cannot diverge.

```
paint(ctx, layout, state, S)      // S = output pixels per composition unit
```

- **Composition units** = the source image's native pixels. `S = 1` for a 1× export, `2` for 2×,
  and `(cssWidth · devicePixelRatio) / compositionWidth` for the preview.
- **Everything is multiplied by `S` explicitly. No `ctx.scale()`.** Reason: per the HTML spec,
  `shadowBlur` / `shadowOffset*` are *not* affected by the CTM, so a scaled transform would silently
  desynchronise shadows between preview and export. Explicit multiplication removes the class of bug.

**Layout**

```
ref  = max(iw, ih)                     // padding is relative → same look at any capture size
pad  = padding% · ref
W,H  = iw + 2·pad, ih + 2·pad
if ratio: grow the short axis to hit the ratio (never crop, never upscale the source at 1×)
radius = radius% · 0.06 · min(iw, ih)  // relative → correct on both 1× and Retina captures
```

Padding and radius are stored as relative values and *displayed* in px, so defaults look right
whether the input is a 1280×800 grab or a 5120×2880 Retina one.

**Draw order**

1. Background — solid, linear gradient, or **mesh** (base fill + N soft radial blobs in fractional
   coordinates, so it reflows with the aspect ratio). Transparent = skip.
2. Grain — one cached 128×128 noise tile, `CanvasPattern.setTransform(scale(S))` so the grain is the
   same physical size in preview and export.
3. Shadow — two layers (broad ambient + tight contact), cast **from the artwork's alpha**, not from a
   filled rectangle. Screenshots taken with macOS `⌘⇧4`+space have transparent corners; a filled
   rect behind them would show as black nubs. Technique: draw the artwork off-canvas at `x − K` with
   `shadowOffsetX = (K + dx)·S`, so only the shadow lands on the canvas — no double-compositing of
   semi-transparent pixels.
4. Artwork — the image, rounded-clipped into a cached offscreen canvas, plus an inner hairline.

**Caches**: the artwork canvas is keyed on `(image, S, radius, edge)` so dragging the padding,
shadow or background sliders never re-renders the image. Large sources are downscaled to preview
size by successive halving (one-shot huge downscales alias badly).

---

## 4. Risk list

| Risk | Handling |
| --- | --- |
| `ctx.roundRect` missing (Safari < 16.4) | feature-detect, arc-based fallback path |
| Clipboard blocked on `file://` (not a secure context) | construct `ClipboardItem` with the **promise** of the blob inside the gesture (Safari loses user activation across `await`); on failure show a visible error + a manual fallback (open in a new tab / drag the canvas out), never fail silently |
| Canvas size limits at 2× on huge captures | guard on max dimension/area, fall back to 1× with a toast rather than exporting a blank PNG |
| `toBlob` returning null | treated as an error path, surfaced in the status line |
| `localStorage` unavailable (private mode) | every access wrapped; the app runs fine without it |
| Gradient banding | grain overlay, on by default |

---

## 5. Verification — results

Driven in a real browser, not eyeballed. The shared MCP Chrome profile was held by another session,
so this ran against a private headless Chrome 151 over a small CDP driver.

| Check | Result |
| --- | --- |
| Console across every scenario | clean, zero messages |
| Network requests after load | **none** — server log shows only `index.html` across 10 loads |
| Export dimensions, 1440×900 source | 2× → 3254×2174, 1× → 1627×1087 — exact match to the readout, confirmed on disk with `sips` |
| Ratios | 16:9 → 1.7778, 4:3 → 1.3335, 1:1 → 1.0000, 2:1 → 2.0000 |
| Large source (3360×2000) | 7594×4874 PNG in ~4.2 s including encode |
| Transparent export | corners `(0,0,0,0)`, artwork opaque, shadow preserved as real alpha |
| Input paths | drop (with veil), paste (replaces), picker, non-image rejected with the image kept |
| Persistence | every setting survives reload; the image deliberately does not |
| `⌘S` / `⌘C` | export saved, clipboard write succeeded |
| `file://` | loads and renders; Chrome treats it as a secure context so copy works there too |
| Live preview under drag (3360×2000 source) | ~16.6 ms median, flat across padding / shadow / radius drags |
| Text contrast | all muted text 4.95:1, primary 15.2:1 |
| Lighthouse (headed Chrome 151, desktop) | Accessibility **100**, Best Practices **100**, SEO **100**, Agentic Browsing **100** — 48 audits passed, 0 failed, in both the empty and loaded states |
| Accessibility tree | every slider, radio and switch carries a name and value; backgrounds announce by preset name; the canvas is a labelled `img`; the toast is the only live region |

**Found and fixed during verification**

1. `[hidden]` was being overridden by class-level `display` rules — an empty chip and a ghost button
   were rendering. Fixed with a global `[hidden]{display:none!important}`.
2. The first shadow model was nearly invisible at 1:1 — too diffuse to read against a saturated
   background. Replaced with the three-layer model plus the padding-bounded falloff.
3. A light screenshot on a light background had no boundary at all. The edge treatment gained an
   outer hairline to go with the inner one, and the control was renamed *Edge definition*.
4. Default corner radius was too tight; raised.
5. Muted text sat at 3.2:1, below AA. Lifted the two muted tokens to 4.95:1 and 7.3:1.
6. Reading the accessibility tree (which Lighthouse scores can't tell you) showed each `<output>` is
   an implicit polite live region, so dragging a slider announced twice — once as the slider value,
   once as the readout. The outputs are now `aria-live="off"`.
7. The sliders' native `valuetext` announced the raw number (`6.5`) rather than the meaningful
   readout (`94 px`). `aria-valuetext` is the correct fix and Firefox/Safari honour it, but a
   sentinel probe proved **Chrome ignores `aria-valuetext` on a native `input[type=range]`**. So each
   slider is additionally `aria-describedby` its own output, keeping the px value reachable in Chrome
   without reintroducing the double announcement.
8. On a short window (806 css px, an ordinary laptop) the sidebar scroll cut off mid-swatch-grid with
   no cue. The footer now casts a shadow up over the scroll area so it reads as continuing beneath.

**Deviations from the plan above**: none in architecture. The off-canvas shadow caster works (the
runtime probe returns true in Chrome), so the fallback path is untaken there but retained.

---

## 6. Follow-up: crop + annotation (requested after the first commit)

Explicitly requested, and scoped by the user to **crop + arrow / box / ellipse** — text, highlighter
and blur/redaction were offered and deliberately not taken, so they stay out.

**Model.** `crop` is a rectangle on whole source pixels; `shapes` are `{type,x1,y1,x2,y2,color,weight}`
in the **source image's own pixels**. Both live outside `state`, so neither is persisted — they are
meaningless against a different image, and loading one resets them. Coordinates in source space is
what makes a shape stay anchored to what it points at through a crop, a ratio change, and either
export scale.

- Crop is **non-destructive**: entering crop mode shows the full frame again with the selection over
  it, so you can readjust or reset rather than losing pixels. `content()` is the single place that
  decides what region is visible.
- Annotations render in `paint()`, not `getArtwork()` — so drawing never invalidates the artwork
  cache. Verified: a live shape drag holds ~16.6 ms on a 3360×2000 source.
- Editing chrome (dim, thirds guides, handles, selection outline) is drawn on a **separate overlay
  canvas**, which makes it structurally impossible for UI to leak into an export.
- One gesture is one undo step; history stores whole `{crop, shapes}` snapshots, capped at 60.

**Verification** — 19 checks driven through real CDP mouse and key events, asserting on canvas pixel
counts rather than DOM state, plus the 10 base checks re-run: **29/29, console clean.** Covers each
tool via its keyboard shortcut, colour selection, shift-constrain, `⌘Z`, click-select, `⌫` delete,
crop new/move/resize/flip, `Enter` apply, Reset, Clear all, that annotations survive a crop, and that
the export matches the cropped readout exactly.

**Found and fixed here**

1. `Clear all` stayed greyed out after drawing, and the crop dimensions did not tick during a drag —
   `syncEditUI()` only ran at mutation sites. Now it runs on the render path.
2. Dragging a crop handle past its opposite collapsed the selection to the 24 px minimum. Now the
   rect normalises, so the handle flips, which is what every real crop tool does.
3. After a Reset, dragging inside the image *moved* an implicit full-frame selection instead of
   starting a fresh one — a full-frame crop is now treated as no crop at all, and only an explicit
   selection can be moved.
4. Crop rects were fractional, which makes `drawImage` resample the region instead of lifting it out
   1:1. They are now rounded to whole source pixels.
5. `setTool` pushed the undo snapshot *after* setting the opening crop, so undoing it was a no-op.
6. Crop guides and frame were white-on-white and vanished over a light screenshot. Now dark-then-light.
7. The app's own shadow-probe canvas triggered Chrome's `getImageData` readback warning on every
   load; it now passes `willReadFrequently`. This was the one console message in the whole app, and
   it is worth recording that it came from the app rather than the test harness.
