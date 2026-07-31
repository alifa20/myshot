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
  `(image, scale, radius, edge)`, so dragging padding or shadow never re-renders the image.

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
