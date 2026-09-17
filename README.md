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
| Two-up | **Layout → Two-up** in the Frame group puts two screenshots side by side, before and after |
| Export | **Export PNG** or `⌘S` — saves `screenshot-<timestamp>.png` |
| Copy | **Copy** or `⌘C` — writes a PNG to the clipboard |
| Reset | **Reset** in the footer restores the defaults |

Every control is live — there is nothing to apply. Settings (not the image) persist in
`localStorage`, so the next session opens where you left off.

**Controls** — padding, corner radius, shadow, canvas ratio (Auto · 16:9 · 4:3 · 1:1 · 2:1 for
X/Twitter), layout (single · two-up), 12 background presets (6 mesh, 3 gradient, 3 solid) plus
transparent and a custom colour, edge definition, grain, and 1× / 2× export scale (1× — actual
size — by default; every export is also capped at 1MB regardless of which scale is picked, see
Guardrails below).

### Two-up, for before and after

Switch **Layout** to **Two-up** and the stage shows two equal-width slots on one shared background,
before on the left, after on the right. One slot is always *active*, marked with a thin amber line:
`⌘V`, **Open…** and **Sample** load into it, and after each load the active slot moves on to the
next empty one, so two pastes in a row fill the pair. Dragging a file highlights the slot under the
cursor and drops it there. Click a slot, or press `1` / `2`, to make it active; **Clear** empties the
active slot. Each slot keeps its own crop and annotations, and `⌘Z` is one shared history.

The two images are scaled to the same width — the wider one is 1:1 and the narrower one is scaled up
to match — and padding, corner radius and shadow are measured against the shared slot, so both
halves get identical treatment with a gutter equal to the padding. Switching back to **Single**
keeps the left image and parks the right one in memory; switch back and it returns with its crop and
shapes (if only the right slot was filled, it moves into the single view instead). Export and Copy
work with one slot empty — that half is bare background and the toast says so. In two-up, 1× means
the wider image at its native size *before* the 1MB cap; two full-screen captures side by side will
usually be scaled down to fit under it, and the toast names that too.

### Cropping and annotating

Pick a tool once, then drag on the preview. `V` select · `C` crop · `A` arrow · `R` box · `E` ellipse.

- **Crop** shows the full frame again with a selection over it: drag outside the box to start a new
  one, inside it to move it, or grab any of the 8 handles to resize (dragging a handle past its
  opposite flips the selection). Thirds guides while you work, live dimensions in the bar, `Enter`
  or **Done** to apply, **Reset** to restore the full frame. It is **non-destructive** — the crop is
  a stored rectangle, so you can reopen and readjust it, and nothing is ever thrown away.
- **Arrow / box / ellipse** drag out from the point you press. Six colours, one weight slider that
  scales with the image, and `shift` constrains a box to a square or an ellipse to a circle.
  The arrow is a single filled path — a shaft that tapers from tail to head, then a broad head about
  6× the shaft across with a slight sweep. A uniform line with a small triangle on the end reads as
  timid at screenshot scale; the head has to carry the arrow.
- **Select** then click a shape to select it, `⌫` to delete. `⌘Z` undoes any edit, **Clear all**
  removes everything. Changing the colour or weight while a shape is selected retints that shape.

Annotations are stored in the **source image's own pixels** (each slot's own, in two-up), not screen
coordinates, so they stay anchored to whatever they point at through cropping, ratio changes, layout
changes and either export scale — and
they are drawn by the same `paint()` as everything else, so a 2× export renders them at full
resolution rather than scaling up a preview. They are clipped to the artwork, so a shape dragged
past the edge stops at the image instead of bleeding onto the background.

## How it renders

One function paints the composition at any scale, and both the live preview and the export call it.
They cannot drift apart, because the same code produces both.

- **Composition units are the widest image's own pixels.** In single layout that is simply the
  source image; in two-up each slot is scaled to that width by `k = slotWidth / imageWidth`, so the
  wider image is 1:1 and the other is scaled up to match. `S` is output pixels per unit: `1` or `2`
  for an export, `fit × devicePixelRatio` for the preview. So the preview is Retina-crisp
  independently of the export multiplier, and single layout renders byte-for-byte what it did before
  slots existed.
- **In two-up every shadow is painted before any artwork.** A shadow's blur can spread across the
  whole gutter, and painting it after the neighbour's image would darken that image.
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
- **Grain** is a cached 8×8 ordered (Bayer) dither tile at a fixed ~2.5/255 alpha — invisible as
  texture, but it dithers away the banding that 8-bit canvas gradients produce. It used to be a
  128×128 tile of independent random noise; that looked the same but produced near-incompressible
  PNGs (a 12MB export was routine). An ordered pattern gives the same effect at a fraction of the
  exported file size, since it's periodic instead of random.
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
- Every export/copy is capped at 1MB. It's measured synchronously (`toDataURL`, not `toBlob`) right
  after painting; if it lands over budget, grain is dropped and re-measured, and if it's *still* over
  budget the resolution is shrunk in a loop (each step re-measured, not estimated-and-hoped) until it
  fits or hits a floor — an output small enough that even literal random-noise pixels would compress
  under 1MB, so the loop can't stop with the cap still blown. Whatever combination of grain-drop and
  shrink actually happened is named in the toast, e.g. "grain dropped and scaled down to stay under
  1MB". An undecodable file and a non-image drop each surface as their own error.
- Clipboard writes build the `ClipboardItem` from the real `Blob`, synchronously inside the gesture —
  Safari drops user activation across an `await`, and the size-budget loop above is itself sync
  (`toDataURL`) precisely so it can run before `clipboard.write()` without losing that activation. If
  the write still fails (e.g. no secure context) the toast says why and offers **Download instead**;
  Copy never fails silently.
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

Two-up was verified the same way in Chrome 153 headless over CDP, 119 assertions plus a byte-for-byte
diff of the single-layout exports against the pre-two-up build (1× and 2×, cropped and uncropped —
identical). A 1440×900 beside a 1280×800 composes to exactly 3161×1087; a shape drawn in the scaled
slot lands on the same feature in the preview, the 1× export and the 2× export; the crop dim, handles,
selection, `⌫`, **Clear all** and `⌘Z` all act on the right slot; the parked slot round-trips through
single and back with its crop; the layout survives a reload and **Reset**; and an export with an
empty slot keeps real alpha there on a transparent background. See `docs/plan.md` §7.

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
