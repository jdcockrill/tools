# Slideshow Presenter — Design Spec

**Date:** 2026-03-29
**Status:** Ready for implementation

---

## Overview

A single self-contained `presenter.html` tool that lets you queue up image and SVG slides, reorder them, save/load the deck as a zip, and present full-screen with keyboard and swipe navigation.

Follows the same conventions as the other tools in this repo: plain HTML/CSS/JS, no build step, Google Fonts + CDN dependencies only where unavoidable.

---

## Slide types

- **Images:** JPEG, PNG, WebP, GIF, AVIF
- **SVG:** loaded as an `<img>` tag (not inlined — keeps things simple and sandboxed)
- HTML slides and external URLs are explicitly out of scope for this version

---

## Architecture

Single file: `presenter.html`. Two logical modes rendered in the same page:

1. **Builder mode** — default view; queue management, import/export
2. **Presenter mode** — full-viewport overlay rendered on top of builder mode

No routing, no state libraries. Plain DOM + vanilla JS.

### Dependencies (CDN)

- **JSZip** — zip import/export. No viable pure-JS alternative at a reasonable size.
- **Google Fonts** (Outfit + DM Mono) — matches the other tools.

SortableJS is _not_ used. Drag-and-drop implemented with native pointer events + touch support (gives better control over mobile behaviour and avoids an extra dependency).

---

## Builder mode

### Layout

**Narrow (< 720px — mobile):**
- Full-width vertical slide list
- Drop zone above the list
- Import / Export / Present buttons in a row below

**Wide (≥ 720px — desktop):**
- Left column (~220px): vertical slide list + drop zone
- Right column (flex 1): preview panel showing selected slide at 16:9
- Header bar: title, Import, Export, Present buttons

### Slide list (vertical)

Each row contains:
- **Drag handle** — six-dot grip icon on the left edge; pointer changes to `grab`/`grabbing`
- **Thumbnail** — 40×28px rendered from the loaded file (canvas for images, img for SVG)
- **Filename** — truncated with ellipsis
- **Type badge** — small pill: SVG / PNG / JPG / etc.
- **Remove button** — × on the right

The selected slide (highlighted in blue border) is the one shown in the preview panel and is set by clicking anywhere on the row except the drag handle or remove button.

Slide numbers (1-based) are shown as a faint overlay on each thumbnail.

### Drop zone

Appears above the list (mobile) or at the top of the left column (desktop). Accepts drag-and-drop of files onto the zone _or_ anywhere on the page. Also has a `<input type="file" multiple accept="image/*,.svg">` triggered by clicking the zone.

Files added via drop or picker are appended to the bottom of the queue.

### Drag-and-drop reorder

Implemented with `pointerdown` / `pointermove` / `pointerup`. On drag start, a semi-transparent clone of the row follows the pointer. The list shows a drop-position indicator line as you drag. On release the item is inserted at the indicated position.

On mobile this works via touch (pointer events cover touch). Each row also has small up/down arrow buttons (always visible, secondary to the drag handle) as a fallback for devices where drag proves unreliable.

### Preview panel (wide only)

Shows the currently selected slide rendered at 16:9 aspect ratio. Letterboxed (object-fit: contain) on a dark background. Clicking prev/next buttons below the preview changes the selection.

### Actions

- **Import zip** — file picker accepting `.zip`; reads `index.txt` to determine order, loads all referenced files as blob URLs
- **Export zip** — uses JSZip to bundle all slide files + `index.txt` (one filename per line, in order); triggers download as `presentation.zip`
- **Present** — enters presenter mode; disabled if queue is empty

---

## Presenter mode

A full-viewport `position:fixed` overlay rendered above the builder.

### Slide display

- Black background
- Slide image centred, letterboxed (`object-fit: contain`, max 100vw × 100vh)
- No padding — slide fills as much of the viewport as its aspect ratio allows

### Always-visible element

- Faint slide counter `2 / 7` in the top-right corner (white, low opacity)

### Control bar

Hidden by default. Fades in (opacity transition, ~200ms) when:
- Pointer moves within 80px of the bottom of the viewport
- User taps the bottom third of the viewport (mobile)

Fades out after 3 seconds of no pointer movement near the bottom.

Control bar layout (left → right):
```
[ ✕ Exit ]          [spacer]    [ ‹ Prev ]  [● ● ● ●]  [ Next › ]    [spacer]    [ ⛶ Full ]
```

- **Exit** — returns to builder mode (same as Esc)
- **Prev / Next** — navigate slides
- **Dot indicators** — one dot per slide, current slide filled white; dots are clickable to jump directly to a slide; hidden entirely if > 20 slides (counter top-right is sufficient at that scale)
- **Full** — triggers `element.requestFullscreen()` on the overlay; button changes to ⛶ exit-fullscreen when active

### Keyboard navigation

| Key | Action |
|-----|--------|
| `←` or `↑` | Previous slide |
| `→` or `↓` | Next slide |
| `Esc` | Exit presenter mode (also exits fullscreen if active) |
| `F` | Toggle fullscreen |

### Mobile / touch navigation

- Swipe left → next slide
- Swipe right → previous slide
- Tap bottom third → show/hide control bar
- Minimum swipe distance: 40px to avoid accidental navigation

### Wrap-around

Navigation does **not** wrap. At the first slide, Prev is disabled. At the last slide, Next is disabled. The dot indicator and counter make position clear.

---

## Zip format

```
presentation.zip
├── index.txt           # one filename per line, defines slide order
├── intro-slide.svg
├── chart.png
└── summary.jpg
```

`index.txt` lists only the filenames (no paths). On import, any file in the zip not listed in `index.txt` is ignored. Files listed in `index.txt` but missing from the zip are skipped with a console warning. If `index.txt` is absent, all supported image/SVG files in the zip are loaded in alphabetical order.

---

## State

No persistence beyond the current session. Closing or refreshing the tab clears the queue. The zip export/import is the persistence mechanism.

---

## index.html

Add a new entry to `index.html`:

```html
<li>
  <a href="presenter.html">Slideshow Presenter</a>
  <span class="desc">— Queue images and SVGs into a presentation, then present full-screen.</span>
</li>
```

---

## Out of scope (v1)

- HTML slide type
- External URL slides
- Slide transitions / animations
- Speaker notes
- Laser pointer / drawing overlay
- Multiple presentations / named decks
