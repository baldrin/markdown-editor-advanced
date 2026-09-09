# Plan

Markdown Editor Advanced is a fork of [markdown-editor](https://github.com/baldrin/markdown-editor). The original is finished and will not change. This project starts from the same file and grows.

## Decisions already made

- **Separate repository.** Nothing here touches the original.
- **Single file, no build step, for now.** Everything ships in `index.html` (plus `sw.js` once the PWA lands, since a service worker cannot be inlined). A bundler only comes in if the editing core is replaced by CodeMirror, and that decision is deferred until the overlay approach has been tried.
- **Storage is IndexedDB, with no library.** IndexedDB is built into the browser like localStorage, so the page stays self-contained. It holds objects and binary blobs and is not capped near 5 MB. A small wrapper (about 40 lines) is enough; no dependency.
- **Images are stored as blobs, not data URIs in the text.** A pasted image goes into IndexedDB and the markdown references it by id. The preview and the export inline it. This keeps the editor text small; a data URI would put a 300 KB line in the textarea.
- **Out of scope.** Mermaid diagrams (too large). Share by URL (not needed).
- **Math is wanted but not urgent.** KaTeX, loaded on demand from a CDN only when a document contains math, so the editor itself stays offline-capable.
- **Storage keys use the `mdadv.` prefix** so this editor never shares drafts or settings with the original.
- **License is MIT** with no personal name in the copyright line.

## Feature order

Each step should leave the editor working and committed before the next starts.

### 1. Markdown highlighting in the editor pane (overlay)

This goes first because it decides whether the textarea stays. If the overlay cannot be aligned, the editor core moves to CodeMirror and find and replace would use CodeMirror's own search, so building find on the textarea first would be wasted.

- Keep the textarea. Render a `<pre>` copy of the text underneath it with spans for: headings, emphasis markers, inline code, code fences and their bodies, links and URLs, blockquote markers, list markers, task boxes, table pipes, horizontal rules.
- The textarea gets `color: transparent` and a visible `caret-color`. Selection highlight still comes from the textarea. `--accent-soft` must stay translucent or the selection will hide the overlay text beneath it.
- **Color only. Never change font weight, style, size, or family** in the overlay. Any of those changes glyph widths and the two layers drift.
- Overlay and textarea must share every layout-affecting property: font, size, line-height, letter-spacing, padding, `white-space: pre-wrap`, `overflow-wrap: break-word`, `tab-size`.
- The overlay width must equal the textarea's `clientWidth`, not `100%`, so a visible scrollbar does not change where long lines wrap. The existing caret mirror already does this.
- Mirror scroll from the textarea to the overlay inside `requestAnimationFrame`, not in the scroll handler directly, to avoid one-frame jitter.
- Re-render the overlay on input, debounced only if typing lag appears above a few thousand lines.
- Test with: the welcome document (it has an emoji on line 1), long wrapped lines, tabs, CJK text, a 5,000-line file, and macOS with "Always show scrollbars" or a mouse plugged in.
- If the layers cannot be kept aligned on the test cases, stop and revisit CodeMirror rather than piling on fixes.
- Line numbers are **not** part of this step. With wrapped lines they need per-line height measurement and are where the overlay approach gets ugly.

### 2. Find and replace

- Cmd+F opens a small bar above the editor: search field, match count, previous/next, replace field, replace one, replace all, close (Esc). Enter and Shift+Enter in the search field step through matches. Cmd+F while the bar is open refocuses the search field.
- Case-insensitive by default with a toggle. Optional regex toggle.
- The overlay highlights every match and the current one, because the textarea's own selection is invisible in Safari and gray in Chrome while focus is in the search field.
- Matches are also selected in the textarea and scrolled into view. `ensureCaretVisible` returns early when the editor is not focused, so factor the scroll part out and call that.
- Replace goes through `insertText` so it stays on the native undo stack. Replace all replaces the span from the first match to the last match only, not the whole document, so caret and scroll position survive.

### 3. Print to PDF

- A print stylesheet that hides toolbar, editor, sidebar, and status bar and prints the preview at full width with sensible page margins.
- The page is `height: 100%` with `overflow: hidden` and the preview pane scrolls internally, so hiding chrome alone prints one screenful. The print stylesheet must also reset html and body height and overflow, make the preview pane static, and drop the 60vh bottom padding on the preview.
- Force the light palette in print so the dark theme does not print black pages.
- A Print button in the toolbar and Cmd+P.
- Code blocks should not split awkwardly (`break-inside: avoid` on pre and table rows).

### 4. Multiple documents

- Move draft storage from localStorage to IndexedDB. Each document is one record: id, name, text, cursor, scroll position, last modified.
- A documents panel (can share the sidebar with the TOC as tabs): list, new, rename, delete with confirmation, last-modified time, current document highlighted.
- Every document autosaves. Switching documents is instant and preserves cursor and scroll position per document.
- Keep the one-time migration: on first load, if an `mdadv.content` draft exists in localStorage, import it as the first document.
- Files opened from disk: opening a file creates a document in the list and remembers the file name. The file handle is stored with the document (handles are storable in IndexedDB) and permission is re-requested when the document is next opened. The browser copy is the working copy; Save writes it back to disk. The old backup-before-replace behaviour in `loadContent` goes away, since opening no longer replaces anything.
- "Export all" as a zip is the escape hatch since browser storage is not a backup. Use fflate (about 10 KB) or defer this until it is needed.

### 5. Image paste

- Paste or drop an image into the editor. Store the blob in IndexedDB under an id and insert `![](img:ID)` at the caret.
- Downscale large images on a canvas before storing (cap the long edge around 1600 px). Keep PNG when the image has transparency, otherwise JPEG at quality around 0.85, so one screenshot does not become 4 MB.
- The preview resolves `img:` references to object URLs after render. Export HTML and the zip export inline them as data URIs so the output is still self-contained.
- The existing window drop handler reads every dropped file as text and would load an image's bytes as a document. Branch on file type there.
- Deleting a document deletes its blobs. Blobs are per document, so the same image pasted twice is stored twice; that is fine.

### 6. PWA

- `manifest.webmanifest` as a real file. A data URI manifest has no base URL, so its paths would have to be absolute, and Safari ignores it. Icons must be real files, so a small `icons/` folder is acceptable here.
- `sw.js` at the repository root caching `index.html` and icons.
- Network-first for `index.html` with the cache as fallback, so users get the current build whenever they are online and the cache name never strands anyone. Still version the cache name and delete old caches on activate. Show a "new version available, reload" bar when a new worker takes control while offline.
- Runtime-cache KaTeX responses after the first fetch so math works offline afterwards.
- GitHub Pages serves from the branch root, which is the path the service worker needs.

### 7. Math (later)

- Detect `$...$` and `$$...$$` in the source. Inline math requires no whitespace just inside the delimiters, so "$5 and $10" is not math. Only then inject KaTeX CSS and JS from a CDN.
- Math must be captured before `marked` runs inline parsing, with a `marked` tokenizer extension. Rendering after `marked` does not work because underscores and asterisks inside the math have already become emphasis.
- `throwOnError: false`, leave the `trust` option off.
- Note in the README that math needs a network connection the first time.

## Not planned

- Mermaid or other diagram rendering.
- Share by URL.
- Collaboration or any server component.

## Working notes

- The editor keeps the native textarea undo stack by inserting text with `document.execCommand('insertText')` and falling back to `setRangeText`. New editing features should go through `insertText` in the same way.
- Scroll sync between the panes uses a `scrollLock` owner and short timers. Typing sets the lock to the editor so preview re-renders cannot scroll the editor away from the caret. Anything that scrolls a pane programmatically should claim the lock first.
- `ensureCaretVisible` measures the caret with a hidden mirror div. The overlay in step 1 can replace this mirror, since the overlay is a live mirror already.
- The table of contents scroll spy treats a heading as current once it is in the top third of the preview pane and always picks the last heading when the pane is at the bottom.

## Verification

Check each feature in a real browser, not only by reading code: open the page, use the feature, watch the console for errors. Keep the original editor's behaviour intact throughout: file open and save, export, themes, list continuation, Tab handling, shortcuts.
