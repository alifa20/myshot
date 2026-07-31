Build me a screenshot beautifier like Shots or Xnapper, as a single static page.

It should feel like a lightweight Mac app: a static HTML file you open in Safari and pin to the
Dock via Safari's "Add to Dock" (File > Add to Dock, or the Share menu), giving it its own window
and icon without a browser tab. Do not build a native wrapper, WKWebView shell, or packaged .app —
the static page is the whole deliverable.

Requirements:

- Drag-and-drop or paste a screenshot onto the page (also support a file picker).
- Render it on a canvas with: adjustable padding, corner radius, and shadow;
  a background picker with ~10 tasteful presets (gradients + solid colors) plus
  a custom color input; optional 16:9 / 4:3 / 1:1 / Twitter-size canvas ratios.
- Live preview of every control, no Apply buttons. Controls in a sidebar.
- Export as PNG at 1x or 2x with one click; also Copy to clipboard.
  - Target Safari as the primary browser. Copy-to-clipboard uses the Clipboard API, which
    requires a secure context — opening the file directly (`file://`, no local server) can
    block `navigator.clipboard.write` in Safari. If the write fails, fall back to a visible
    error plus a way to get the image out manually (e.g. drag the canvas/export out, or a
    "right-click > Save Image" affordance) — do not let Copy silently fail.
  - Export filename pattern: `screenshot-{timestamp}.png`.
- Everything client-side · no uploads, no server, works offline once loaded.
  Vanilla JS + canvas, one HTML file, no build step, no frameworks.
- Make the defaults gorgeous so the first export already looks premium — match the restrained
  padding, soft shadows, and mesh/gradient backgrounds that Xnapper and CleanShot X ship by
  default, not a generic drop-shadow-on-white look.
- Dropping or pasting a new screenshot replaces the current canvas image (no multi-image stacking).
- Keyboard shortcuts: ⌘C copies the current export, ⌘S triggers the PNG export/download.
- Persist the last-used control settings (padding, radius, background, ratio, etc.) in
  localStorage so they survive a reload.
- Handle canvas resolution in two layers: render the live preview crisp on Retina using
  devicePixelRatio, independent of the 1x/2x export multiplier — preview and export should not
  visually diverge.
