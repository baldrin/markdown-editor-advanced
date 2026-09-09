# Plan

Markdown Editor Advanced is a fork of [markdown-editor](https://github.com/baldrin/markdown-editor). The original is finished and will not change. This project starts from the same file and grows.

## Decisions already made

- **Separate repository.** Nothing here touches the original.
- **Single file, no build step, for now.** Everything ships in `index.html` (plus `sw.js` once the PWA lands, since a service worker cannot be inlined). A bundler only comes in if the editing core is replaced by CodeMirror, and that decision is deferred until the overlay approach has been tried.
- **Out of scope.** Mermaid diagrams (too large). Share by URL (not needed).
- **Math is wanted but not urgent.** KaTeX, loaded on demand from a CDN only when a document contains math, so the editor itself stays offline-capable.
- **Storage keys use the `mdadv.` prefix** so this editor never shares drafts or settings with the original.
- **License is MIT** with no personal name in the copyright line.

## Feature order

Each step should leave the editor working and committed before the next starts.

### 1. Find and replace

- Cmd+F opens a small bar above the editor: search field, match count, previous/next, replace field, replace one, replace all, close (Esc).
- Case-insensitive by default with a toggle. Optional regex toggle.
- Matches are selected in the textarea and scrolled into view (reuse `ensureCaretVisible`).
- Replace goes through `insertText` so it stays on the native undo stack.

### 2. Markdown highlighting in the editor pane (overlay)

- Keep the textarea. Render a `<pre>` copy of the text underneath it with spans for: headings, emphasis markers, inline code, code fences and their bodies, links and URLs, blockquote markers, list markers, task boxes, table pipes, horizontal rules.
- The textarea gets `color: transparent` and a visible `caret-color`. Selection highlight still comes from the textarea.
- **Color only. Never change font weight, style, size, or family** in the overlay. Any of those changes glyph widths and the two layers drift.
- Overlay and textarea must share every layout-affecting property: font, size, line-height, letter-spacing, padding, width, `white-space: pre-wrap`, `overflow-wrap: break-word`, `tab-size`.
- Mirror scroll from the textarea to the overlay inside `requestAnimationFrame`, not in the scroll handler directly, to avoid one-frame jitter.
- Re-render the overlay on input, debounced only if typing lag appears above a few thousand lines.
- Test with: the welcome document (it has an emoji on line 1), long wrapped lines, tabs, CJK text, a 5,000-line file.
- If the layers cannot be kept aligned on the test cases, stop and revisit CodeMirror rather than piling on fixes.
- Line numbers are **not** part of this step. With wrapped lines they need per-line height measurement and are where the overlay approach gets ugly.

### 3. Print to PDF

- A print stylesheet that hides toolbar, editor, sidebar, and status bar and prints the preview at full width with sensible page margins.
- A Print button in the toolbar and Cmd+P.
- Code blocks should not split awkwardly (`break-inside: avoid` on pre and table rows).

### 4. Multiple documents

- Move draft storage from localStorage to IndexedDB. localStorage caps near 5 MB per origin, which pasted images will exhaust quickly.
- A documents panel (can share the sidebar with the TOC as tabs): list, new, rename, delete with confirmation, last-modified time, current document highlighted.
- Every document autosaves. Switching documents is instant and preserves cursor and scroll position per document.
- Keep the one-time migration: on first load, if an `mdadv.content` draft exists in localStorage, import it as the first document.
- "Export all" as a zip is the escape hatch since browser storage is not a backup. Use fflate (about 10 KB) or defer this until it is needed.

### 5. Image paste

- Paste or drop an image into the editor and insert `![](data:...)` at the caret.
- Downscale large images on a canvas before embedding (cap the long edge around 1600 px, JPEG quality around 0.85) so one screenshot does not become a 4 MB document.
- Rendered preview and exported HTML need no changes; data URIs already work.

### 6. PWA

- `manifest.webmanifest` inline as a data URI in the HTML, or as a file. Icons must be real files, so a small `icons/` folder is acceptable here.
- `sw.js` at the repository root caching `index.html` and icons.
- Version the cache name on every release and delete old caches on activate. Show a "new version available, reload" bar when a new worker takes control. Getting this wrong strands users on an old build.
- GitHub Pages serves from the branch root, which is the path the service worker needs.

### 7. Math (later)

- Detect `$...$` and `$$...$$` in the source. Only then inject KaTeX CSS and JS from a CDN and render after `marked`.
- `throwOnError: false`, leave the `trust` option off.
- Note in the README that math needs a network connection the first time.

## Not planned

- Mermaid or other diagram rendering.
- Share by URL.
- Collaboration or any server component.

## Working notes

- The editor keeps the native textarea undo stack by inserting text with `document.execCommand('insertText')` and falling back to `setRangeText`. New editing features should go through `insertText` in the same way.
- Scroll sync between the panes uses a `scrollLock` owner and short timers. Typing sets the lock to the editor so preview re-renders cannot scroll the editor away from the caret. Anything that scrolls a pane programmatically should claim the lock first.
- `ensureCaretVisible` measures the caret with a hidden mirror div. The overlay in step 2 can replace this mirror, since the overlay is a live mirror already.
- The table of contents scroll spy treats a heading as current once it is in the top third of the preview pane and always picks the last heading when the pane is at the bottom.

## Verification

Check each feature in a real browser, not only by reading code: open the page, use the feature, watch the console for errors. Keep the original editor's behaviour intact throughout: file open and save, export, themes, list continuation, Tab handling, shortcuts.
