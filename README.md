# Myshot

A screenshot beautifier — padding, corner radius, shadow and a background, rendered to a canvas
and exported as a PNG. One static HTML file, vanilla JS, no build step, no dependencies, no server.

```
open index.html
```

That's the whole install. It also works from a `file://` URL with no local server, and offline —
the page makes **zero network requests** after it loads (verified: no webfonts, no CDN, no
analytics; the only URLs in the source are SVG namespace strings inside inline data URIs).

## Using it

| | |
| --- | --- |
| Load | drag a file anywhere on the page, `⌘V` to paste, or **Open…** / **Choose file** |
| Export | **Export PNG** or `⌘S` — saves `screenshot-<timestamp>.png` |
| Copy | **Copy** or `⌘C` — writes a PNG to the clipboard |
| Reset | **Reset** in the footer restores the defaults |

Every control is live — there is nothing to apply. Settings (not the image) persist in
`localStorage`, so the next session opens where you left off.

**Controls** — padding, corner radius, shadow, canvas ratio (Auto · 16:9 · 4:3 · 1:1 · 2:1 for
X/Twitter), 12 background presets (6 mesh, 3 gradient, 3 solid) plus transparent and a custom
colour, edge definition, grain, and 1× / 2× export scale.

### Cropping and annotating

Pick a tool once, then drag on the preview. `V` select · `C` crop · `A` arrow · `R` box · `E` ellipse.

- **Crop** shows the full frame again with a selection over it: drag outside the box to start a new
  one, inside it to move it, or grab any of the 8 handles to resize (dragging a handle past its
  opposite flips the selection). Thirds guides while you work, live dimensions in the bar, `Enter`
  or **Done** to apply, **Reset** to restore the full frame. It is **non-destructive** — the crop is
  a stored rectangle, so you can reopen and readjust it, and nothing is ever thrown away.
- **Arrow / box / ellipse** drag out from the point you press. Six colours, one weight slider that
  scales with the image, and `shift` constrains a box to a square or an ellipse to a circle.
- **Select** then click a shape to select it, `⌫` to delete. `⌘Z` undoes any edit, **Clear all**
  removes everything. Changing the colour or weight while a shape is selected retints that shape.

Annotations are stored in the **source image's own pixels**, not screen coordinates, so they stay
anchored to whatever they point at through cropping, ratio changes and either export scale — and
they are drawn by the same `paint()` as everything else, so a 2× export renders them at full
resolution rather than scaling up a preview. They are clipped to the artwork, so a shape dragged
past the edge stops at the image instead of bleeding onto the background.

## How it renders

One function paints the composition at any scale, and both the live preview and the export call it.
They cannot drift apart, because the same code produces both.

- **Composition units are the source image's own pixels.** `S` is output pixels per unit: `1` or `2`
  for an export, `fit × devicePixelRatio` for the preview. So the preview is Retina-crisp
  independently of the export multiplier.
- **Nothing uses `ctx.scale()`.** Per the HTML spec `shadowBlur` and `shadowOffset*` ignore the
  current transform, so a scaled CTM would desynchronise shadows between preview and export. Every
  dimension is multiplied by `S` explicitly instead.
- **Padding and radius are stored relative** (to the longest and shortest edge respectively) and
  displayed in px — so the defaults look right on a 1280×800 grab and on a 5120×2880 Retina one.
- **The shadow is cast from the artwork's alpha, not from a filled rectangle.** A macOS `⌘⇧4`+space
  capture has transparent corners; a filled rect behind it would show as black nubs. The caster is
  drawn off-canvas with a compensating `shadowOffsetX`, so only its shadow lands and semi-transparent
  pixels are never composited twice. A one-time probe checks the engine actually renders that, and
  falls back if not.
- **Three shadow layers** (ambient, key, contact), bounded by the room the padding gives them — a
  falloff clipped by the canvas edge reads as a mistake.
- **Backgrounds** are a base fill plus soft radial blobs in fractional coordinates, so a preset
  reflows correctly at any aspect ratio. Swatches are real renders of the same function, so a swatch
  can't drift from what the canvas paints.
- **Grain** is a cached 128×128 noise tile at ±2.5/255 — invisible as texture, but it dithers away
  the banding that 8-bit canvas gradients produce.
- Large sources are downscaled to preview size by successive halving; the artwork is cached on
  `(image, crop, scale, radius, edge)`, so dragging padding or shadow never re-renders the image.
  A cropped region is lifted out at native size once and cached too, on whole pixels — a fractional
  source rect would make `drawImage` resample the region instead of copying it 1:1.
- Annotations are drawn in `paint()` rather than into the artwork, so drawing a shape never
  invalidates that cache: a live shape drag stays at ~16 ms even on a 3360×2000 source.
- Crop handles and selection chrome live on a **separate overlay canvas**, so editing UI is
  structurally incapable of reaching the export. Its guides are drawn dark-then-light, because a
  single white line disappears over a white screenshot — which is most screenshots.

## Guardrails

- 2× exports that would exceed the engine's canvas limits fall back to 1× **with a visible notice**,
  rather than handing back a blank PNG.
- `toBlob` returning null, an undecodable file, and a non-image drop each surface as an error.
- Clipboard writes build the `ClipboardItem` from the *promise* of the blob, synchronously inside
  the gesture — Safari drops user activation across an `await`. If the write still fails (e.g. no
  secure context) the toast says why and offers **Download instead**; Copy never fails silently.
- `localStorage` access is wrapped; the app runs normally in private mode.
- `ctx.roundRect` is feature-detected with an `arcTo` fallback.

## Verified

Driven in Chrome 151 over CDP — see `docs/plan.md` for the full log. Exports are pixel-exact against
the stated dimensions (2× → 3254×2174, 1× → 1627×1087 for a 1440×900 source), all five ratios are
numerically exact, transparent exports keep real alpha, drop / paste / picker / rejection paths all
behave, settings survive a reload, the clipboard write succeeds, every muted text colour clears
WCAG AA, and the console stays clean throughout. Live preview holds ~16 ms per frame while dragging
any slider against a 3360×2000 source. Lighthouse scores 100 on accessibility, best practices, SEO
and agentic browsing — 48 audits, 0 failures.

One known browser gap: Chrome ignores `aria-valuetext` on a native `input[type=range]`, so a screen
reader there announces a slider's raw number rather than its px readout. The attribute is set anyway
(Firefox and Safari honour it) and each slider is `aria-describedby` its own readout so the px value
stays reachable in Chrome.

## Layout

```
index.html      the entire application
docs/prompt.md  the brief
docs/plan.md    design direction, architecture, scope calls, verification log
```
