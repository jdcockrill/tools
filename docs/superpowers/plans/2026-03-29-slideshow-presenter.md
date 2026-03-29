# Slideshow Presenter Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build `presenter.html` — a single-file browser tool for queueing images and SVGs, presenting them full-screen with keyboard/swipe navigation, and saving/loading decks as zip files.

**Architecture:** Single self-contained HTML file. Builder mode (queue + preview) and presenter mode (full-screen overlay) coexist on one page. `slides` array is the sole source of truth; all UI re-renders from it. No router, no state library. Plain DOM + vanilla JS. Commits happen after each tested task; push only at the final task.

**Tech Stack:** Vanilla HTML/CSS/JS. JSZip 3.10.1 (jsDelivr CDN) for zip. Google Fonts (Outfit + DM Mono). Pointer Events API for drag-and-drop.

---

## Files

| File | Action | Responsibility |
|------|--------|---------------|
| `presenter.html` | Create | The entire tool — builder + presenter mode |
| `index.html` | Modify | Add link to presenter.html |

---

### Task 1: HTML shell and CSS

**Files:**
- Create: `presenter.html`

Write the complete file with all HTML structure and CSS. The `<script>` block contains only state variables and stubs — no logic yet.

- [ ] **Step 1: Create `presenter.html` with this content**

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Slideshow Presenter</title>
  <link rel="preconnect" href="https://fonts.googleapis.com">
  <link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
  <link href="https://fonts.googleapis.com/css2?family=Outfit:wght@400;500;600&family=DM+Mono&display=swap" rel="stylesheet">
  <style>
    :root {
      --bg: #fafaf9; --fg: #111; --muted: #888;
      --border: #d6d6d4; --surface: #efeeec; --link: #1a56db;
    }
    @media (prefers-color-scheme: dark) {
      :root {
        --bg: #0d0d0d; --fg: #efefef; --muted: #5e5e5e;
        --border: #2b2b2b; --surface: #1a1a1a; --link: #5b9cf6;
      }
    }
    *, *::before, *::after { box-sizing: border-box; }
    body {
      font-family: 'Outfit', system-ui, sans-serif;
      font-size: 1rem; line-height: 1.6;
      margin: 2rem auto; max-width: 1100px;
      padding: 0 1rem;
      background: var(--bg); color: var(--fg);
    }
    h1 { font-size: 1.4rem; font-weight: 600; margin: 0.25rem 0 0; letter-spacing: -0.01em; }
    a.back { color: var(--link); font-size: 0.9rem; text-decoration: none; }
    a.back:hover { text-decoration: underline; }

    /* ── Drop zone ────────────────────────────────────────────────────────── */
    #drop-zone {
      display: block;
      margin-top: 2.5rem;
      border: 1.5px dashed var(--border); border-radius: 10px;
      padding: 5rem 2rem; text-align: center; cursor: pointer;
      transition: border-color 0.2s, background 0.2s;
    }
    #drop-zone:hover, #drop-zone.over { border-color: var(--link); background: var(--surface); }
    #drop-zone .dz-icon {
      display: block; margin: 0 auto 1.1rem; color: var(--fg);
      opacity: 0.25; transition: opacity 0.2s, transform 0.25s;
    }
    #drop-zone:hover .dz-icon, #drop-zone.over .dz-icon { opacity: 0.55; transform: translateY(-4px); }
    #drop-zone .dz-main { font-size: 1rem; font-weight: 500; }
    #drop-zone .dz-sub  { font-size: 0.875rem; color: var(--muted); margin-top: 0.1rem; }
    #drop-zone .dz-sub span { color: var(--link); }

    /* ── Editor ───────────────────────────────────────────────────────────── */
    #editor { display: none; margin-top: 1.5rem; }

    .editor-header {
      display: flex; align-items: center; gap: 0.5rem;
      margin-bottom: 1.25rem; flex-wrap: wrap;
    }
    .editor-header .spacer { flex: 1; }

    .editor-body {
      display: grid; grid-template-columns: 1fr; gap: 1.25rem;
    }
    @media (min-width: 720px) {
      .editor-body { grid-template-columns: 240px 1fr; align-items: start; }
    }

    /* ── Queue panel ──────────────────────────────────────────────────────── */
    .add-zone {
      display: block;
      border: 1.5px dashed var(--border); border-radius: 8px;
      padding: 0.6rem; text-align: center; cursor: pointer;
      font-size: 0.85rem; color: var(--muted); margin-bottom: 0.6rem;
      transition: border-color 0.2s, background 0.2s;
    }
    .add-zone:hover, .add-zone.over { border-color: var(--link); background: var(--surface); color: var(--link); }

    .queue-label {
      font-size: 0.72rem; font-weight: 600;
      text-transform: uppercase; letter-spacing: 0.06em;
      color: var(--muted); margin-bottom: 0.4rem;
    }

    #slide-list { list-style: none; margin: 0; padding: 0; display: flex; flex-direction: column; gap: 0.3rem; }

    .slide-row {
      display: flex; align-items: center; gap: 0.5rem;
      background: var(--bg); border: 1px solid var(--border); border-radius: 5px;
      padding: 0.35rem 0.5rem; cursor: pointer;
      transition: border-color 0.12s; user-select: none;
    }
    .slide-row.selected { border-color: var(--link); border-width: 1.5px; }
    .slide-row.drag-over-above { border-top: 2.5px solid var(--link); }
    .slide-row.drag-over-below { border-bottom: 2.5px solid var(--link); }
    .slide-row.dragging { opacity: 0.4; }

    .slide-handle {
      color: var(--muted); cursor: grab; font-size: 13px;
      flex-shrink: 0; opacity: 0.45; padding: 0 1px; line-height: 1;
      touch-action: none;
    }
    .slide-handle:active { cursor: grabbing; }

    .slide-thumb-wrap {
      position: relative; width: 40px; height: 28px;
      flex-shrink: 0; border-radius: 2px; overflow: hidden; background: var(--surface);
    }
    .slide-thumb-wrap img { width: 100%; height: 100%; object-fit: contain; display: block; }
    .slide-num {
      position: absolute; top: 1px; left: 2px;
      font-size: 8px; color: #fff; background: rgba(0,0,0,0.45);
      border-radius: 2px; padding: 0 3px; line-height: 1.5;
      font-family: 'DM Mono', monospace;
    }

    .slide-name {
      flex: 1; font-size: 0.75rem;
      white-space: nowrap; overflow: hidden; text-overflow: ellipsis;
    }
    .slide-badge {
      font-size: 0.6rem; background: var(--surface); color: var(--muted);
      border-radius: 3px; padding: 1px 4px; flex-shrink: 0;
      font-family: 'DM Mono', monospace; text-transform: uppercase;
    }
    .slide-up, .slide-down, .slide-remove {
      background: none; border: none; color: var(--muted); cursor: pointer;
      font-size: 12px; padding: 0 2px; opacity: 0.4; flex-shrink: 0; line-height: 1;
    }
    .slide-up:hover, .slide-down:hover, .slide-remove:hover { opacity: 1; }
    .slide-remove:hover { color: #c00; }

    /* ── Drag ghost ───────────────────────────────────────────────────────── */
    #drag-ghost {
      position: fixed; pointer-events: none; z-index: 9999;
      opacity: 0.82; transform: rotate(1.5deg) scale(1.02);
      background: var(--bg); border: 1.5px solid var(--link);
      border-radius: 5px; padding: 0.35rem 0.5rem;
      display: none; align-items: center; gap: 0.5rem;
      box-shadow: 0 4px 18px rgba(0,0,0,0.18); max-width: 240px;
    }

    /* ── Preview panel ────────────────────────────────────────────────────── */
    #preview-panel { display: none; }
    @media (min-width: 720px) { #preview-panel { display: block; } }

    .preview-label {
      font-size: 0.72rem; font-weight: 600;
      text-transform: uppercase; letter-spacing: 0.06em;
      color: var(--muted); margin-bottom: 0.4rem;
    }
    #preview-wrap {
      aspect-ratio: 16/9; background: #111;
      border-radius: 6px; border: 1px solid var(--border);
      overflow: hidden; display: flex; align-items: center; justify-content: center;
    }
    #preview-img { max-width: 100%; max-height: 100%; object-fit: contain; display: none; }
    #preview-empty { font-size: 0.8rem; color: #555; }
    .preview-nav { display: flex; justify-content: flex-end; gap: 0.4rem; margin-top: 0.5rem; }

    /* ── Buttons ──────────────────────────────────────────────────────────── */
    button {
      padding: 0.35rem 0.75rem; border: 1px solid var(--border);
      border-radius: 5px; background: var(--surface); color: var(--fg);
      cursor: pointer; font-size: 0.85rem;
      font-family: 'Outfit', system-ui, sans-serif; font-weight: 500;
      transition: opacity 0.15s;
    }
    button:hover { opacity: 0.65; }
    button:disabled { opacity: 0.35; cursor: default; }
    .btn-primary {
      padding: 0.45rem 1.1rem; border: none; border-radius: 5px;
      background: var(--link); color: #fff; font-size: 0.9rem; font-weight: 600;
      font-family: 'Outfit', system-ui, sans-serif; cursor: pointer; transition: opacity 0.15s;
    }
    .btn-primary:hover { opacity: 0.82; }
    .btn-primary:disabled { opacity: 0.35; cursor: default; }

    /* ── Presenter overlay ────────────────────────────────────────────────── */
    #presenter {
      position: fixed; inset: 0; background: #000; z-index: 1000;
      display: none; align-items: center; justify-content: center; overflow: hidden;
    }
    #presenter.active { display: flex; }

    #presenter-img {
      max-width: 100vw; max-height: 100vh; object-fit: contain;
      user-select: none; -webkit-user-select: none; pointer-events: none;
    }

    #presenter-counter {
      position: absolute; top: 12px; right: 14px;
      font-family: 'DM Mono', monospace; font-size: 0.72rem;
      color: rgba(255,255,255,0.28); pointer-events: none;
    }

    #presenter-controls {
      position: absolute; bottom: 0; left: 0; right: 0;
      background: linear-gradient(to top, rgba(0,0,0,0.85) 0%, rgba(0,0,0,0.5) 60%, transparent 100%);
      padding: 2.5rem 1rem 0.8rem;
      display: flex; align-items: center; gap: 0.6rem;
      opacity: 0; transition: opacity 0.2s; pointer-events: none;
    }
    #presenter-controls.visible { opacity: 1; pointer-events: auto; }

    .ctrl-btn {
      background: rgba(255,255,255,0.12); border: 1px solid rgba(255,255,255,0.25);
      color: #fff; border-radius: 5px; padding: 0.32rem 0.7rem;
      font-size: 0.78rem; font-family: 'Outfit', system-ui, sans-serif; font-weight: 500;
      cursor: pointer; transition: background 0.15s; flex-shrink: 0;
    }
    .ctrl-btn:hover { background: rgba(255,255,255,0.22); }
    .ctrl-btn:disabled { opacity: 0.3; cursor: default; }
    .ctrl-btn.muted { color: rgba(255,255,255,0.55); border-color: rgba(255,255,255,0.18); }

    #dots-wrap { display: flex; align-items: center; gap: 5px; flex-wrap: wrap; justify-content: center; }
    .dot {
      width: 6px; height: 6px; border-radius: 50%;
      background: rgba(255,255,255,0.3); cursor: pointer;
      transition: background 0.12s, transform 0.12s; flex-shrink: 0;
    }
    .dot.active { background: #fff; transform: scale(1.25); }
    .dot:hover { background: rgba(255,255,255,0.7); }
  </style>
</head>
<body>
  <a class="back" href="index.html">← Tools</a>
  <h1>Slideshow Presenter</h1>

  <!-- Initial drop zone -->
  <label id="drop-zone" for="file-input">
    <svg class="dz-icon" width="48" height="48" viewBox="0 0 24 24" fill="none"
         stroke="currentColor" stroke-width="1.4" stroke-linecap="round" stroke-linejoin="round">
      <rect x="3" y="3" width="18" height="18" rx="2"/>
      <circle cx="8.5" cy="8.5" r="1.5"/>
      <polyline points="21 15 16 10 5 21"/>
    </svg>
    <div class="dz-main">Drop images or SVGs here</div>
    <div class="dz-sub">or <span>click to browse</span> — JPEG, PNG, WebP, SVG&hellip;</div>
  </label>
  <input type="file" id="file-input" accept="image/*,.svg" multiple style="display:none">

  <!-- Editor -->
  <div id="editor">
    <div class="editor-header">
      <button id="btn-import">Import zip</button>
      <button id="btn-export">Export zip</button>
      <span class="spacer"></span>
      <button class="btn-primary" id="btn-present" disabled>▶ Present</button>
    </div>
    <div class="editor-body">

      <!-- Queue panel (left) -->
      <div id="queue-panel">
        <label class="add-zone" for="add-input" id="add-zone">+ Add slides</label>
        <input type="file" id="add-input" accept="image/*,.svg" multiple style="display:none">
        <div class="queue-label" id="queue-label">Slides (0)</div>
        <ul id="slide-list"></ul>
      </div>

      <!-- Preview panel (right, desktop only) -->
      <div id="preview-panel">
        <div class="preview-label" id="preview-label">Preview</div>
        <div id="preview-wrap">
          <img id="preview-img" alt="">
          <span id="preview-empty">No slide selected</span>
        </div>
        <div class="preview-nav">
          <button id="btn-prev-preview" disabled>‹ Prev</button>
          <button id="btn-next-preview" disabled>Next ›</button>
        </div>
      </div>

    </div>
  </div>

  <!-- Drag ghost (follows pointer during reorder) -->
  <div id="drag-ghost">
    <span style="color:var(--muted);font-size:13px;opacity:.5">⠿</span>
    <div style="width:32px;height:22px;background:var(--surface);border-radius:2px;flex-shrink:0;overflow:hidden">
      <img id="drag-ghost-img" style="width:100%;height:100%;object-fit:contain" alt="">
    </div>
    <span id="drag-ghost-name" style="font-size:0.72rem;max-width:130px;white-space:nowrap;overflow:hidden;text-overflow:ellipsis"></span>
  </div>

  <!-- Import zip picker (hidden) -->
  <input type="file" id="import-input" accept=".zip" style="display:none">

  <!-- Presenter overlay -->
  <div id="presenter">
    <img id="presenter-img" src="" alt="">
    <div id="presenter-counter"></div>
    <div id="presenter-controls">
      <button class="ctrl-btn muted" id="ctrl-exit">✕ Exit</button>
      <div style="flex:1"></div>
      <button class="ctrl-btn" id="ctrl-prev">‹ Prev</button>
      <div id="dots-wrap"></div>
      <button class="ctrl-btn" id="ctrl-next">Next ›</button>
      <div style="flex:1"></div>
      <button class="ctrl-btn muted" id="ctrl-full">⛶ Full</button>
    </div>
  </div>

  <script>
    // ── State ─────────────────────────────────────────────────────────────────
    // slides[i] = { id, name, ext, url }  (url = blob URL, never mutated)
    const slides = [];
    let selectedIdx = -1;   // which slide is selected in builder
    let presenterIdx = 0;   // which slide is showing in presenter
    let nextId = 0;
    let dragState = null;
    let controlBarTimer = null;

    // ── Functions added in Tasks 2-8 ─────────────────────────────────────────

  </script>
</body>
</html>
```

- [ ] **Step 2: Open in browser and verify layout**

Open `presenter.html` in Chrome (via a local server or `file://`). Verify:
- Drop zone renders full-width with icon and text
- Dark mode works if OS is set to dark
- No console errors

- [ ] **Step 3: Commit**

```bash
git add presenter.html
git commit -m "Add slideshow presenter — HTML/CSS shell"
```

---

### Task 2: File loading, slide list, and preview panel

**Files:**
- Modify: `presenter.html` — add to `<script>` block

Add the slide management functions and all event listeners. Replace the `// ── Functions added in Tasks 2-8` comment with the code below.

- [ ] **Step 1: Add slide management functions inside `<script>`**

```javascript
// ── Slide management ──────────────────────────────────────────────────────

function loadFiles(files) {
  let added = 0;
  for (const file of files) {
    if (!file.type.startsWith('image/') && !file.name.toLowerCase().endsWith('.svg')) continue;
    const url = URL.createObjectURL(file);
    const raw = file.name.split('.').pop().toUpperCase();
    const ext = raw === 'JPEG' ? 'JPG' : raw;
    slides.push({ id: nextId++, name: file.name, ext, url });
    added++;
  }
  if (!added) return;
  renderSlideList();
  if (selectedIdx < 0) selectSlide(0);
  document.getElementById('drop-zone').style.display = 'none';
  document.getElementById('editor').style.display = '';
}

function renderSlideList() {
  const list = document.getElementById('slide-list');
  list.innerHTML = '';
  slides.forEach((slide, i) => {
    const li = document.createElement('li');
    li.className = 'slide-row' + (i === selectedIdx ? ' selected' : '');
    li.dataset.idx = i;
    li.innerHTML = `
      <span class="slide-handle" title="Drag to reorder">⠿</span>
      <div class="slide-thumb-wrap">
        <img src="${slide.url}" alt="">
        <span class="slide-num">${i + 1}</span>
      </div>
      <span class="slide-name" title="${slide.name}">${slide.name}</span>
      <span class="slide-badge">${slide.ext}</span>
      <button class="slide-up" data-idx="${i}" title="Move up" ${i === 0 ? 'disabled' : ''}>↑</button>
      <button class="slide-down" data-idx="${i}" title="Move down" ${i === slides.length - 1 ? 'disabled' : ''}>↓</button>
      <button class="slide-remove" data-idx="${i}" title="Remove">×</button>
    `;
    li.addEventListener('click', e => {
      if (e.target.closest('.slide-handle,.slide-remove,.slide-up,.slide-down')) return;
      selectSlide(i);
    });
    list.appendChild(li);
  });

  document.getElementById('queue-label').textContent = `Slides (${slides.length})`;
  document.getElementById('btn-present').disabled = slides.length === 0;

  list.querySelectorAll('.slide-remove').forEach(btn => {
    btn.addEventListener('click', e => { e.stopPropagation(); removeSlide(parseInt(btn.dataset.idx)); });
  });
  list.querySelectorAll('.slide-up').forEach(btn => {
    btn.addEventListener('click', e => { e.stopPropagation(); moveSlide(parseInt(btn.dataset.idx), -1); });
  });
  list.querySelectorAll('.slide-down').forEach(btn => {
    btn.addEventListener('click', e => { e.stopPropagation(); moveSlide(parseInt(btn.dataset.idx), 1); });
  });
}

function selectSlide(idx) {
  if (!slides.length) { selectedIdx = -1; updatePreview(); return; }
  selectedIdx = Math.max(0, Math.min(slides.length - 1, idx));
  renderSlideList();
  updatePreview();
}

function updatePreview() {
  const img   = document.getElementById('preview-img');
  const empty = document.getElementById('preview-empty');
  const label = document.getElementById('preview-label');
  if (selectedIdx >= 0 && selectedIdx < slides.length) {
    img.src = slides[selectedIdx].url;
    img.style.display = '';
    empty.style.display = 'none';
    label.textContent = `Preview — slide ${selectedIdx + 1} of ${slides.length}`;
  } else {
    img.src = '';
    img.style.display = 'none';
    empty.style.display = '';
    label.textContent = 'Preview';
  }
  document.getElementById('btn-prev-preview').disabled = selectedIdx <= 0;
  document.getElementById('btn-next-preview').disabled = selectedIdx >= slides.length - 1;
}

function removeSlide(idx) {
  URL.revokeObjectURL(slides[idx].url);
  slides.splice(idx, 1);
  if (selectedIdx >= slides.length) selectedIdx = slides.length - 1;
  if (!slides.length) {
    selectedIdx = -1;
    document.getElementById('editor').style.display = 'none';
    document.getElementById('drop-zone').style.display = '';
  }
  renderSlideList();
  updatePreview();
}

function moveSlide(idx, dir) {
  const to = idx + dir;
  if (to < 0 || to >= slides.length) return;
  [slides[idx], slides[to]] = [slides[to], slides[idx]];
  if (selectedIdx === idx) selectedIdx = to;
  else if (selectedIdx === to) selectedIdx = idx;
  renderSlideList();
  updatePreview();
}

// ── File input & drop zone events ─────────────────────────────────────────

document.getElementById('file-input').addEventListener('change', e => { loadFiles(e.target.files); e.target.value = ''; });
document.getElementById('add-input').addEventListener('change',  e => { loadFiles(e.target.files); e.target.value = ''; });

// Initial drop zone
const dz = document.getElementById('drop-zone');
dz.addEventListener('dragover',  e => { e.preventDefault(); dz.classList.add('over'); });
dz.addEventListener('dragleave', e => { if (!dz.contains(e.relatedTarget)) dz.classList.remove('over'); });
dz.addEventListener('drop',      e => { e.preventDefault(); dz.classList.remove('over'); loadFiles(e.dataTransfer.files); });

// "Add slides" zone inside editor
const az = document.getElementById('add-zone');
az.addEventListener('dragover',  e => { e.preventDefault(); e.stopPropagation(); az.classList.add('over'); });
az.addEventListener('dragleave', e => { if (!az.contains(e.relatedTarget)) az.classList.remove('over'); });
az.addEventListener('drop',      e => { e.preventDefault(); e.stopPropagation(); az.classList.remove('over'); loadFiles(e.dataTransfer.files); });

// Drop anywhere on the page when editor is open
document.addEventListener('dragover', e => e.preventDefault());
document.addEventListener('drop', e => {
  e.preventDefault();
  if (e.dataTransfer.files.length) loadFiles(e.dataTransfer.files);
});

// Preview nav
document.getElementById('btn-prev-preview').addEventListener('click', () => selectSlide(selectedIdx - 1));
document.getElementById('btn-next-preview').addEventListener('click', () => selectSlide(selectedIdx + 1));

// Present button (enterPresenter defined in Task 4)
document.getElementById('btn-present').addEventListener('click', () => enterPresenter());
```

- [ ] **Step 2: Open in browser and verify**

Drop 2–3 image files onto the drop zone. Verify:
- Drop zone hides, editor appears
- Each file appears as a row with thumbnail, filename, type badge, ↑↓ and × buttons
- Clicking a row highlights it in blue (selected)
- On a ≥720px window: preview panel shows the selected slide
- ↑/↓ buttons reorder the list
- × removes the slide; removing last slide returns to drop zone
- Preview prev/next buttons step through slides

- [ ] **Step 3: Commit**

```bash
git add presenter.html
git commit -m "Slideshow presenter: file loading, slide list, preview panel"
```

---

### Task 3: Drag-and-drop reorder

**Files:**
- Modify: `presenter.html` — add to `<script>` block, after the file events section

- [ ] **Step 1: Add drag-and-drop functions**

```javascript
// ── Drag-and-drop reorder ─────────────────────────────────────────────────

function initDragDrop() {
  document.getElementById('slide-list').addEventListener('pointerdown', e => {
    const handle = e.target.closest('.slide-handle');
    if (!handle) return;
    e.preventDefault();
    const row = handle.closest('.slide-row');
    const fromIdx = parseInt(row.dataset.idx);
    const ghost = document.getElementById('drag-ghost');
    document.getElementById('drag-ghost-img').src = slides[fromIdx].url;
    document.getElementById('drag-ghost-name').textContent = slides[fromIdx].name;
    ghost.style.display = 'flex';
    ghost.style.left = (e.clientX - 16) + 'px';
    ghost.style.top  = (e.clientY - 16) + 'px';
    row.classList.add('dragging');
    dragState = { fromIdx, rowEl: row };
    document.addEventListener('pointermove', onDragMove);
    document.addEventListener('pointerup',   onDragEnd);
    document.addEventListener('pointercancel', onDragEnd);
  });
}

function onDragMove(e) {
  if (!dragState) return;
  const ghost = document.getElementById('drag-ghost');
  ghost.style.left = (e.clientX - 16) + 'px';
  ghost.style.top  = (e.clientY - 16) + 'px';

  const rows = Array.from(document.querySelectorAll('#slide-list .slide-row'));
  rows.forEach(r => r.classList.remove('drag-over-above', 'drag-over-below'));

  // Find the row the pointer is nearest to and whether we're above/below its midpoint
  let target = null, pos = 'below';
  for (const row of rows) {
    const rect = row.getBoundingClientRect();
    if (e.clientY < rect.top + rect.height / 2) { target = row; pos = 'above'; break; }
    target = row; pos = 'below';
  }
  if (target) target.classList.add(pos === 'above' ? 'drag-over-above' : 'drag-over-below');
  dragState.target = target;
  dragState.targetPos = pos;
}

function onDragEnd(e) {
  if (!dragState) return;
  document.getElementById('drag-ghost').style.display = 'none';
  const rows = Array.from(document.querySelectorAll('#slide-list .slide-row'));
  rows.forEach(r => r.classList.remove('drag-over-above', 'drag-over-below', 'dragging'));

  const { fromIdx, target, targetPos } = dragState;
  dragState = null;
  document.removeEventListener('pointermove', onDragMove);
  document.removeEventListener('pointerup',   onDragEnd);
  document.removeEventListener('pointercancel', onDragEnd);

  if (!target) return;
  const targetIdx = parseInt(target.dataset.idx);
  // toIdx = index before which to insert in the current (pre-removal) array
  let toIdx = targetPos === 'above' ? targetIdx : targetIdx + 1;
  if (toIdx === fromIdx || toIdx === fromIdx + 1) return; // no-op

  const item = slides.splice(fromIdx, 1)[0];
  const adjusted = toIdx > fromIdx ? toIdx - 1 : toIdx;
  slides.splice(adjusted, 0, item);

  // Keep selectedIdx tracking the same slide
  const newSelected =
    selectedIdx === fromIdx ? adjusted :
    selectedIdx > fromIdx && selectedIdx <= adjusted ? selectedIdx - 1 :
    selectedIdx < fromIdx && selectedIdx >= adjusted ? selectedIdx + 1 :
    selectedIdx;
  renderSlideList();
  selectSlide(newSelected);
}
```

- [ ] **Step 2: Call `initDragDrop()` at the bottom of the `<script>` block**

After all the function/event definitions, add:

```javascript
// ── Init ──────────────────────────────────────────────────────────────────
initDragDrop();
```

- [ ] **Step 3: Open in browser and verify drag-and-drop**

Load 3+ slides. Grab the ⠿ handle on a row and drag it to a new position. Verify:
- Ghost element follows the pointer
- Blue indicator line shows where drop will land
- Releasing drops the slide into the new position
- Slide numbers in thumbnails update to reflect new order
- Selected slide highlight stays on the same slide content (not position)

- [ ] **Step 4: Commit**

```bash
git add presenter.html
git commit -m "Slideshow presenter: drag-and-drop reorder"
```

---

### Task 4: Presenter mode shell

**Files:**
- Modify: `presenter.html` — add to `<script>` block, before `// ── Init`

- [ ] **Step 1: Add presenter enter/exit/show functions**

```javascript
// ── Presenter mode ────────────────────────────────────────────────────────

function enterPresenter(startIdx) {
  if (!slides.length) return;
  presenterIdx = (startIdx !== undefined) ? startIdx : (selectedIdx >= 0 ? selectedIdx : 0);
  document.getElementById('presenter').classList.add('active');
  document.body.style.overflow = 'hidden';
  showPresenterSlide(presenterIdx);
}

function exitPresenter() {
  document.getElementById('presenter').classList.remove('active');
  document.body.style.overflow = '';
  if (document.fullscreenElement) document.exitFullscreen().catch(() => {});
  document.removeEventListener('keydown', onPresenterKey);
}

function showPresenterSlide(idx) {
  presenterIdx = Math.max(0, Math.min(slides.length - 1, idx));
  document.getElementById('presenter-img').src = slides[presenterIdx].url;
  document.getElementById('presenter-counter').textContent =
    `${presenterIdx + 1} / ${slides.length}`;
  updatePresenterControls();
}

function updatePresenterControls() {
  document.getElementById('ctrl-prev').disabled = presenterIdx <= 0;
  document.getElementById('ctrl-next').disabled = presenterIdx >= slides.length - 1;
  const wrap = document.getElementById('dots-wrap');
  wrap.innerHTML = '';
  if (slides.length <= 20) {
    slides.forEach((_, i) => {
      const dot = document.createElement('div');
      dot.className = 'dot' + (i === presenterIdx ? ' active' : '');
      dot.addEventListener('click', () => showPresenterSlide(i));
      wrap.appendChild(dot);
    });
  }
}
```

- [ ] **Step 2: Open in browser and verify presenter entry/exit**

Load 2+ slides, click **▶ Present**. Verify:
- Black overlay covers the full viewport
- The slide selected in builder is shown first
- Slide counter (e.g. "1 / 3") appears faintly top-right
- Pressing Esc does nothing yet (keyboard wired in Task 5) — that's expected

- [ ] **Step 3: Commit**

```bash
git add presenter.html
git commit -m "Slideshow presenter: presenter mode overlay and slide display"
```

---

### Task 5: Control bar, keyboard navigation, and dot indicators

**Files:**
- Modify: `presenter.html` — add to `<script>` block, before `// ── Init`

- [ ] **Step 1: Add control bar and keyboard functions**

```javascript
// ── Control bar ───────────────────────────────────────────────────────────

function showControls() {
  document.getElementById('presenter-controls').classList.add('visible');
  clearTimeout(controlBarTimer);
  controlBarTimer = setTimeout(hideControls, 3000);
}

function hideControls() {
  clearTimeout(controlBarTimer);
  document.getElementById('presenter-controls').classList.remove('visible');
}

function initControlBar() {
  const presenter = document.getElementById('presenter');
  presenter.addEventListener('pointermove', e => {
    if (e.clientY > window.innerHeight - 80) showControls();
    else { clearTimeout(controlBarTimer); }
  });

  document.getElementById('ctrl-exit').addEventListener('click', exitPresenter);
  document.getElementById('ctrl-prev').addEventListener('click', () => showPresenterSlide(presenterIdx - 1));
  document.getElementById('ctrl-next').addEventListener('click', () => showPresenterSlide(presenterIdx + 1));
}

// ── Keyboard navigation ───────────────────────────────────────────────────

function onPresenterKey(e) {
  switch (e.key) {
    case 'ArrowLeft':
    case 'ArrowUp':
      e.preventDefault(); showPresenterSlide(presenterIdx - 1); break;
    case 'ArrowRight':
    case 'ArrowDown':
      e.preventDefault(); showPresenterSlide(presenterIdx + 1); break;
    case 'Escape':
      exitPresenter(); break;
    case 'f': case 'F':
      if (typeof toggleFullscreen === 'function') toggleFullscreen(); break;
  }
}
```

- [ ] **Step 2: Wire keyboard listener inside `enterPresenter`**

In the `enterPresenter` function (Task 4), add one line at the end:

```javascript
function enterPresenter(startIdx) {
  if (!slides.length) return;
  presenterIdx = (startIdx !== undefined) ? startIdx : (selectedIdx >= 0 ? selectedIdx : 0);
  document.getElementById('presenter').classList.add('active');
  document.body.style.overflow = 'hidden';
  showPresenterSlide(presenterIdx);
  document.addEventListener('keydown', onPresenterKey);  // ← add this line
}
```

- [ ] **Step 3: Add `initControlBar()` to the Init section**

```javascript
// ── Init ──────────────────────────────────────────────────────────────────
initDragDrop();
initControlBar();  // ← add this line
```

- [ ] **Step 4: Open in browser and verify**

Enter presenter mode. Verify:
- Move pointer to within 80px of the bottom → control bar fades in
- Control bar fades out after 3 seconds of inactivity
- ↑ ↓ ← → arrow keys navigate slides
- Esc exits presenter mode and returns to builder
- Prev/Next buttons in the bar navigate slides
- Dot indicators appear (for ≤20 slides), current slide dot is white
- Clicking a dot jumps to that slide
- Prev disabled on slide 1; Next disabled on last slide

- [ ] **Step 5: Commit**

```bash
git add presenter.html
git commit -m "Slideshow presenter: control bar, keyboard nav, dot indicators"
```

---

### Task 6: Touch and swipe navigation

**Files:**
- Modify: `presenter.html` — add to `<script>` block, before `// ── Init`

- [ ] **Step 1: Add touch/swipe handler**

```javascript
// ── Touch / swipe navigation ──────────────────────────────────────────────

function initPresenterTouch() {
  const presenter = document.getElementById('presenter');
  let tx = 0, ty = 0;

  presenter.addEventListener('touchstart', e => {
    tx = e.touches[0].clientX;
    ty = e.touches[0].clientY;
  }, { passive: true });

  presenter.addEventListener('touchend', e => {
    const dx = e.changedTouches[0].clientX - tx;
    const dy = e.changedTouches[0].clientY - ty;
    const absDx = Math.abs(dx), absDy = Math.abs(dy);

    // Tap in bottom third — toggle control bar
    if (absDx < 10 && absDy < 10) {
      if (e.changedTouches[0].clientY > window.innerHeight * 0.67) {
        document.getElementById('presenter-controls').classList.contains('visible')
          ? hideControls() : showControls();
      }
      return;
    }

    // Horizontal swipe — navigate
    if (absDx > 40 && absDx > absDy) {
      if (dx < 0) showPresenterSlide(presenterIdx + 1); // swipe left = next
      else         showPresenterSlide(presenterIdx - 1); // swipe right = prev
    }
  }, { passive: true });
}
```

- [ ] **Step 2: Call `initPresenterTouch()` inside `enterPresenter`**

```javascript
function enterPresenter(startIdx) {
  if (!slides.length) return;
  presenterIdx = (startIdx !== undefined) ? startIdx : (selectedIdx >= 0 ? selectedIdx : 0);
  document.getElementById('presenter').classList.add('active');
  document.body.style.overflow = 'hidden';
  showPresenterSlide(presenterIdx);
  document.addEventListener('keydown', onPresenterKey);
  initPresenterTouch();  // ← add this line
}
```

Note: `initPresenterTouch()` adds event listeners each time presenter is entered. To prevent stacking, also remove them in `exitPresenter`. Update it:

```javascript
function exitPresenter() {
  document.getElementById('presenter').classList.remove('active');
  document.body.style.overflow = '';
  if (document.fullscreenElement) document.exitFullscreen().catch(() => {});
  document.removeEventListener('keydown', onPresenterKey);
  // Touch listeners are on the presenter element itself — removing the element's
  // .active class stops interaction, but we clean up by re-assigning in enterPresenter.
  // No action needed here as the presenter element stays in DOM.
}
```

Actually, to avoid stacking touch listeners, use the `{ once: false }` + a flag approach. Simplest fix: move `initPresenterTouch()` out of `enterPresenter` and call it once at init instead:

- [ ] **Step 2 (revised): Move touch init to the Init section**

Remove the `initPresenterTouch()` call from `enterPresenter`. Add it to the Init block:

```javascript
// ── Init ──────────────────────────────────────────────────────────────────
initDragDrop();
initControlBar();
initPresenterTouch();  // ← add this line
```

The touch listeners on `#presenter` are harmless when presenter is not `.active` — `showPresenterSlide` won't fire usefully since `slides` may be empty or presenter not visible.

- [ ] **Step 3: Test touch/swipe in Chrome DevTools**

Open Chrome DevTools, enable mobile emulation (any device). Enter presenter mode. Verify:
- Swipe left → next slide
- Swipe right → previous slide
- Tap bottom third of screen → control bar appears/disappears

- [ ] **Step 4: Commit**

```bash
git add presenter.html
git commit -m "Slideshow presenter: touch/swipe navigation"
```

---

### Task 7: Fullscreen API

**Files:**
- Modify: `presenter.html` — add to `<script>` block, before `// ── Init`

- [ ] **Step 1: Add fullscreen functions**

```javascript
// ── Fullscreen ────────────────────────────────────────────────────────────

function toggleFullscreen() {
  const presenter = document.getElementById('presenter');
  if (!document.fullscreenElement) {
    presenter.requestFullscreen().catch(() => {});
  } else {
    document.exitFullscreen().catch(() => {});
  }
}

document.addEventListener('fullscreenchange', () => {
  const btn = document.getElementById('ctrl-full');
  btn.textContent = document.fullscreenElement ? '⛶ Exit' : '⛶ Full';
});

document.getElementById('ctrl-full').addEventListener('click', toggleFullscreen);
```

- [ ] **Step 2: Test fullscreen**

Enter presenter mode, move pointer to bottom to reveal controls, click **⛶ Full**. Verify:
- Browser goes fullscreen (no browser chrome visible)
- Button label changes to **⛶ Exit**
- Pressing Esc exits presenter mode (also exits fullscreen)
- Pressing **F** key toggles fullscreen
- Clicking **⛶ Exit** button exits fullscreen without exiting presenter

- [ ] **Step 3: Commit**

```bash
git add presenter.html
git commit -m "Slideshow presenter: fullscreen API support"
```

---

### Task 8: Zip import and export

**Files:**
- Modify: `presenter.html` — add JSZip script tag in `<head>`, add functions to `<script>`

- [ ] **Step 1: Add JSZip script tag to `<head>`, after the Google Fonts link**

```html
<script src="https://cdn.jsdelivr.net/npm/jszip@3.10.1/dist/jszip.min.js"></script>
```

- [ ] **Step 2: Add zip functions to `<script>` block, before `// ── Init`**

```javascript
// ── Zip import / export ───────────────────────────────────────────────────

function getMimeType(filename) {
  const ext = filename.split('.').pop().toLowerCase();
  return { jpg: 'image/jpeg', jpeg: 'image/jpeg', png: 'image/png',
           webp: 'image/webp', gif: 'image/gif', avif: 'image/avif',
           svg: 'image/svg+xml' }[ext] || 'application/octet-stream';
}

async function exportZip() {
  if (!slides.length) return;
  const zip = new JSZip();
  for (const slide of slides) {
    const blob = await fetch(slide.url).then(r => r.blob());
    zip.file(slide.name, blob);
  }
  zip.file('index.txt', slides.map(s => s.name).join('\n'));
  const blob = await zip.generateAsync({ type: 'blob', compression: 'DEFLATE' });
  const a = document.createElement('a');
  a.href = URL.createObjectURL(blob);
  a.download = 'presentation.zip';
  document.body.appendChild(a);
  a.click();
  document.body.removeChild(a);
  URL.revokeObjectURL(a.href);
}

async function importZip(file) {
  const zip = await JSZip.loadAsync(file);
  const supported = /\.(jpe?g|png|webp|gif|avif|svg)$/i;

  let order;
  const indexFile = zip.file('index.txt');
  if (indexFile) {
    const text = await indexFile.async('string');
    order = text.split('\n').map(l => l.trim()).filter(Boolean);
  } else {
    order = Object.keys(zip.files)
      .filter(n => !zip.files[n].dir && supported.test(n))
      .sort();
  }

  // Clear current deck
  slides.forEach(s => URL.revokeObjectURL(s.url));
  slides.length = 0;
  selectedIdx = -1;

  for (const name of order) {
    const entry = zip.file(name);
    if (!entry) { console.warn(`presenter: missing in zip: ${name}`); continue; }
    const blob = await entry.async('blob');
    const url  = URL.createObjectURL(new Blob([blob], { type: getMimeType(name) }));
    const raw  = name.split('.').pop().toUpperCase();
    const ext  = raw === 'JPEG' ? 'JPG' : raw;
    slides.push({ id: nextId++, name, ext, url });
  }

  if (slides.length) {
    renderSlideList();
    selectSlide(0);
    document.getElementById('drop-zone').style.display = 'none';
    document.getElementById('editor').style.display = '';
  }
}

document.getElementById('btn-export').addEventListener('click', exportZip);
document.getElementById('btn-import').addEventListener('click', () =>
  document.getElementById('import-input').click());
document.getElementById('import-input').addEventListener('change', e => {
  if (e.target.files[0]) importZip(e.target.files[0]);
  e.target.value = '';
});
```

- [ ] **Step 3: Test export**

Load 3 slides. Click **Export zip**. Open the downloaded `presentation.zip` in a file manager. Verify:
- Contains `index.txt` with the 3 filenames in order
- Contains all 3 image/SVG files

- [ ] **Step 4: Test import**

Reload the page (clears state). Click **Import zip** and select the zip just exported. Verify:
- All 3 slides appear in the correct order
- Thumbnails render correctly
- Preview panel shows the first slide

- [ ] **Step 5: Test import without `index.txt`**

Create a zip with 2 images and no `index.txt`. Import it. Verify slides load in alphabetical filename order.

- [ ] **Step 6: Commit**

```bash
git add presenter.html
git commit -m "Slideshow presenter: zip import/export with index.txt"
```

---

### Task 9: index.html, full local test, and push

**Files:**
- Modify: `index.html` — add link
- No changes to `presenter.html`

- [ ] **Step 1: Add entry to `index.html`**

In `index.html`, add this item to the `<ul>` (after the Background Remove entry):

```html
<li>
  <a href="presenter.html">Slideshow Presenter</a>
  <span class="desc">— Queue images and SVGs into a presentation, then present full-screen.</span>
</li>
```

- [ ] **Step 2: Full local end-to-end test**

Open `index.html` in a browser served over HTTP (e.g. `python3 -m http.server 8080`). Follow this test sequence:

1. Click **Slideshow Presenter** link from index — verify it opens
2. Drop 4 images onto the drop zone — verify all appear with thumbnails
3. Drag slide 1 to position 3 — verify order updates, numbers update
4. Click slide 2 — verify preview shows it (desktop)
5. Click ↑ on slide 4 — verify it moves to position 3
6. Click **▶ Present** — verify overlay appears on slide 2 (last selected)
7. Press → three times — verify slide advances, counter updates, dots update
8. Press ← — verify slide goes back
9. Move pointer to bottom — verify control bar fades in
10. Click a dot — verify jump to that slide
11. Click **⛶ Full** — verify fullscreen; press Esc — verify exits presenter
12. Re-enter presenter, swipe left (DevTools mobile) — verify next slide
13. Tap bottom third — verify control bar toggles
14. Exit presenter, click **Export zip** — verify download
15. Reload page, **Import zip** — verify deck restored in correct order
16. Verify dark mode renders correctly (toggle OS dark mode)

- [ ] **Step 3: Commit index.html**

```bash
git add index.html
git commit -m "Add Slideshow Presenter to tools index"
```

- [ ] **Step 4: Push**

```bash
git push
```

Verify at `https://jdcockrill.github.io/tools/` that the new tool appears and works.
