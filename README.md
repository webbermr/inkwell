# Inkwell

Inkwell is a single-file, browser-based Markdown editor with a live side-by-side preview. It runs entirely from `markdown-editor.html`: no framework, no build step, no package manager, and no server required.

## Features

- Live Markdown editing and preview
- Formatting toolbar, undo/redo, find and replace, and document outline
- GitHub-flavored Markdown support through Marked
- Sanitized preview output through DOMPurify
- Syntax-highlighted code blocks through highlight.js
- Mermaid diagrams
- KaTeX math rendering
- Interactive task-list checkboxes
- Sortable preview tables
- Custom `roadmap` code blocks for product roadmap swimlanes
- Import, Markdown export, standalone HTML export, copy HTML, and print/PDF support
- Light and dark themes
- Browser autosave when the host environment provides `window.storage`

## Install

Inkwell is a static HTML app. Install it by downloading or cloning this repository:

```bash
git clone https://github.com/webbermr/inkwell.git
cd inkwell
```

Then open `markdown-editor.html` in a modern browser.

On macOS:

```bash
open markdown-editor.html
```

Or serve the directory locally:

```bash
python3 -m http.server 8000
```

Then visit:

```text
http://localhost:8000/markdown-editor.html
```

## Requirements

Inkwell requires a modern browser and internet access for its CDN-hosted libraries:

- marked
- DOMPurify
- highlight.js
- Mermaid
- KaTeX

If a CDN library fails to load, Inkwell keeps running and the related feature is disabled.

## Development

Edit `markdown-editor.html` directly. There is no compile step.

To syntax-check the JavaScript embedded in the HTML:

```bash
python3 -c "import re;open('/tmp/m.js','w').write(re.findall(r'<script>(.*?)</script>',open('markdown-editor.html').read(),re.S)[-1])"
node --check /tmp/m.js
```

## License

Add a license before publishing this repository if you want others to use or redistribute Inkwell.
