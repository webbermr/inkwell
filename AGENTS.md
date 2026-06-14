# AGENTS.md — Inkwell Markdown Editor

Context for an AI agent working on this app. Read this before editing.

## What it is

Inkwell is a **single-file, browser-based Markdown editor** with a live side-by-side preview. The entire app — HTML, CSS, and JavaScript — lives in one file: `markdown-editor.html` (~1,600 lines). There is **no build step, no bundler, no framework, no package.json**. Open the file in a browser and it runs.

It was built as a prototype that also serves as the behavioral reference for a planned native **SwiftUI macOS port** (see "Native port" below).

## How to run and test

- **Run:** open `markdown-editor.html` in any modern browser.
- **Edit:** change the one file directly. No compile.
- **Syntax-check the JS** without a browser:
  ```bash
  python3 -c "import re;open('/tmp/m.js','w').write(re.findall(r'<script>(.*?)</script>',open('markdown-editor.html').read(),re.S)[-1])"
  node --check /tmp/m.js
  ```
  `node --check` validates syntax only (the code uses browser globals it can't execute).
- **Manual test matrix:** typing + live preview; undo/redo; find/replace (⌘F); outline jump; each export path; theme toggle; a Mermaid diagram; a math expression; a roadmap block; a sortable table; a task checkbox; import a file; refresh to confirm autosave restore.

## Dependencies (all from cdnjs, loaded in `<head>`)

| Library | Version | Purpose |
|---|---|---|
| marked | 12.0.2 | Markdown → HTML |
| DOMPurify | 3.1.6 | Sanitize marked output before it touches the DOM |
| highlight.js | 11.9.0 | Code-block syntax highlighting (+ github / github-dark themes) |
| mermaid | 10.9.1 | Diagram rendering |
| KaTeX | 0.16.9 | Math rendering (core + `auto-render` contrib) |

Every library is **feature-detected** (`if (!window.mermaid) return;`, `if (!window.renderMathInElement) return;`, etc.). If a CDN load fails, that feature silently disables and the editor keeps working. Do not assume a library is present without guarding.

## Architecture

One global IIFE-free script at the end of `<body>`. State is held in module-level `let`/`const` variables; the `<textarea id="editor">` is the source of truth for document content, and `<div id="preview">` holds rendered output. The two are kept in sync by a single `render()` call fired on every change.

### The render pipeline (`render()`) — order matters

1. `extractFrontMatter()` — pull a leading `---` YAML block off the top, return it as a styled HTML card + the remaining source.
2. `protectMath()` — replace `$…$` / `$$…$$` with placeholder tokens (`\uE000n\uE001`) so marked can't mangle LaTeX (e.g. `$x_i$` → `$x<em>i</em>$`).
3. `marked.parse()` → raw HTML.
4. `DOMPurify.sanitize()` → safe HTML.
5. `restoreMath()` — swap tokens back to the original `$…$` text (HTML-escaped).
6. Set `preview.innerHTML = frontMatterCard + sanitizedHtml`.
7. `renderMath()` — KaTeX auto-render over the restored text (ignores `pre`/`code`).
8. `highlightCode()` — highlight.js on `pre code`, **skips `language-mermaid` and `language-roadmap`**; injects a per-block Copy button.
9. `renderMermaid()` — render `language-mermaid` blocks into diagrams.
10. `renderRoadmaps()` — render custom `language-roadmap` blocks into swimlanes.
11. `wireTasks()` — make task-list checkboxes interactive.
12. `wireTables()` — add click-to-sort to table headers.
13. Update word / char / reading-time counts.
14. `buildOutline()` — rebuild the heading sidebar.
15. `scheduleSave()` — debounced autosave.

### Subsystems and their functions

- **Undo/redo history** — custom stack (not the textarea's native undo, which breaks on programmatic edits): `snapshot`, `commitNow`, `commitDebounced` (350 ms typing coalesce), `flush`, `restore`, `undo`, `redo`, `updateHistoryButtons`. Toolbar/structural edits call `flush(); …; commitNow()`. Caret/selection is captured in each snapshot.
- **Toolbar formatting** — `surround` (wrap selection), `linePrefix` (per-line), `insert` (at caret); dispatched from the `actions` map via the toolbar click handler.
- **Smart lists** — `handleEnter` (continue bullets, increment ordered, new task item, exit list on empty item) and `handleTab` (indent / Shift-Tab outdent), in the editor `keydown`.
- **Find & replace** — `openFind`, `runFind`, `gotoMatch`, `replaceCurrent`, `replaceAll`, `paintCount`. Textarea-based: selects + scrolls to the current match (a textarea can only show one selection, so there is no all-matches highlight — a known limitation).
- **Outline / TOC** — `buildOutline` (ATX `#` headings only, skips code fences), `jumpToHeading` (scrolls editor + preview). Toggled by `#outlineToggle`.
- **Mermaid** — `setupMermaid` (theme-aware, `securityLevel:'strict'`, `suppressErrorRendering:true`, `flowchart.htmlLabels:false`), `renderMermaid`, `mermaidSandbox` (off-screen but **real-width** container — width-dependent diagrams like gantt/xy/timeline render blank at zero width), `sweepMermaidOrphans` (removes stray nodes Mermaid leaves on `<body>` so they can't pile up and cover the editor). Validates with `mermaid.parse(..., {suppressErrors:true})` before rendering.
- **Custom roadmap block** — `parseRoadmap` + `buildRoadmap` + `renderRoadmaps`. Not Mermaid; a bespoke ` ```roadmap ` block (see "Custom syntaxes").
- **Tables** — `wireTables`, `sortTable`, `compareVals` (numeric-aware), `paintInd`. Sorting is **preview-only**, resets on re-render; it does not rewrite the Markdown source.
- **Tasks** — `wireTasks` + `toggleTask`: clicking a checkbox flips `[ ]`↔`[x]` in the **source** (so it persists and is undoable), skipping markers inside code fences.
- **Theme** — `data-theme` on `<html>` drives CSS variables; `applyHljsTheme` toggles the two highlight.js stylesheet `<link>`s; `setupMermaid` re-themes diagrams. Toggled by `#theme`.
- **Document name + autosave** — `setDocName`, `scheduleSave`, `persistDoc`, `loadSaved`. Persists `JSON.stringify({name, content})` under key `inkwell:doc` via `window.storage` (the artifact key-value store). The `#docDot` dot indicates an in-flight save.
- **Export** — `openFilenameModal` → `confirmFilename` → `exportMd` / `exportHtml`; plus `copyHtmlOut` and Print (`window.print()`). `cleanPreviewHtml` strips UI-only Copy buttons before export. The standalone HTML export embeds its own stylesheet (including roadmap CSS) and CDN links for KaTeX/highlight.
- **Modals** — `confirmModal` (new-doc), `tableModal` (rows×cols), `diagramModal` (diagram helper), `filenameModal` (export name). All are custom in-page modals; Escape closes whichever is open via the global keydown handler.

## Custom syntaxes (not standard Markdown/Mermaid)

- **` ```roadmap `** — Agile product roadmap. Directives `title`, `scale` (`quarters|halves|years`), `start <year>`, `years <n>`; items `epic <name>`, `capability <name>`, `feature <name> [c1-c2]` where `[c1-c2]` are 1-based timeline column indices. Rendered by `buildRoadmap` as a CSS-grid swimlane.
- **` ```mermaid `** — standard Mermaid; the helper offers templates for all 17 Mermaid diagram types.

## Conventions and constraints (important)

- **No `localStorage` / `sessionStorage`.** This runs as a Claude.ai artifact where browser storage is blocked. Use `window.storage` (async key-value: `get`/`set`/`delete`/`list`). It's feature-detected via `hasStore`; degrade to in-memory if absent.
- **No native `alert` / `confirm` / `prompt`.** The sandbox suppresses them (they return without showing). Every confirmation/prompt is a custom modal. Do not reintroduce native dialogs.
- **No filesystem paths.** The File API exposes only `file.name`, never a directory; downloads don't report their destination. The header shows the file **name** only. Don't fabricate paths (including the browser's `C:\fakepath\` value).
- **Trust boundary:** KaTeX, Mermaid, and roadmap HTML are injected **after** `DOMPurify` (they'd otherwise be mangled). This is safe because each is generated by a library from the user's own input (Mermaid in strict mode; roadmap text is `esc()`-escaped) — not arbitrary pasted HTML. Preserve this: anything injecting raw HTML must come from a trusted generator, not unsanitized user input.
- **Styling is token-based.** All colors come from CSS custom properties defined under `:root` and `[data-theme="dark"]`. Add new UI by reusing these tokens, not hard-coded colors. The accent is ink-teal (`#126e63` light / `#4ec9b6` dark).
- **Formatting:** 2-space indent, semicolons, single quotes, vanilla DOM APIs (no jQuery/framework). Keep functions small and named; group related logic under a `/* ---- section ---- */` comment.

## Common tasks

- **Add a toolbar button:** add `<button class="tool" data-act="x">` in `#toolbar`, add `x: () => …` to the `actions` map. The toolbar handler wraps it with history `flush()`/`commitNow()`.
- **Add a render step:** add a function, call it inside `render()` after `preview.innerHTML` is set, and feature-detect any library it needs.
- **Add a new code-block language renderer:** add `|| block.classList.contains('language-X')` to the `highlightCode` skip check, then handle `pre > code.language-X` in your own render function.
- **Add a modal:** copy an existing `.modal-backdrop` block, wire open/close, and add it to the Escape chain in the global keydown handler.

## Native port

A SwiftUI macOS port is planned; a separate build prompt (`inkwell-macos-swiftui-prompt.md`) specifies it. Several behaviors here are **web-sandbox workarounds** that the native app should replace with platform primitives rather than port literally: the custom undo stack → `NSTextView` `UndoManager`; `window.storage` + filename label → `DocumentGroup`/`FileDocument` with a real file URL and path; custom modals → native sheets/panels. The Mermaid/KaTeX/highlight.js reliance is also the main argument for a `WKWebView` rendering path on macOS over a pure-SwiftUI renderer. If you change feature behavior here, note whether the build prompt needs updating to match.
