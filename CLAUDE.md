# Vibetools Development Guide

This file describes the patterns and conventions used in this project, so Claude (or a human) can easily create more tools.

## Project Structure

```
vibetools/
├── CLAUDE.md               # Development guide (this file)
├── README.md               # Project readme
├── LICENSE                 # License file
└── public/                 # Static assets directory
    ├── index.html          # Main index, sections map to directories
    ├── converters/         # Bidirectional conversion tools
    ├── decoders/           # One-way decoding/inspection tools
    ├── tools/              # Interactive utilities (regex, lookups, etc.)
    └── browser/            # Browser introspection tools
```

**Rule**: Each directory inside `public/` becomes an `<h2>` section in `index.html`. One tool = one line/link.

## File Conventions

- **One HTML file per tool** - no build step, no external files (except CDN libs)
- **Naming**: `kebab-case.html` (e.g., `json-yaml-converter.html`)
- **CDN libraries**: OK when needed (js-yaml, etc.), loaded from jsdelivr/cdnjs

## Dark Theme Color Palette

```css
/* Backgrounds */
--bg-body: #1a1a2e;
--bg-panel: #16213e;
--bg-panel-hover: #1d2a4d;
--bg-input: #0f1629;

/* Borders */
--border: #333;
--border-focus: #00d4ff;

/* Text */
--text: #eee;
--text-muted: #888;
--text-placeholder: #555;

/* Accent */
--accent: #00d4ff;
--accent-hover: #00a8cc;

/* Status */
--success: #2ecc71;
--error: #ff6b6b;
--warning: #f39c12;
```

## Standard CSS Base

Every tool starts with this base:

```css
* { box-sizing: border-box; margin: 0; padding: 0; }
body {
    font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
    background: #1a1a2e;
    color: #eee;
    min-height: 100vh;
    padding: 20px;
}
h1 { text-align: center; margin-bottom: 20px; color: #00d4ff; font-weight: 300; }
.container { max-width: 1000px; margin: 0 auto; }
```

## Common UI Components

### Panel Header
```html
<div class="panel-header">
    <h2>Panel Title</h2>
    <div class="buttons">
        <button onclick="copy()">Copy</button>
        <button class="btn-secondary" onclick="clear()">Clear</button>
    </div>
</div>
```

```css
.panel-header {
    display: flex;
    justify-content: space-between;
    align-items: center;
    margin-bottom: 10px;
}
.panel-header h2 {
    font-size: 14px;
    text-transform: uppercase;
    letter-spacing: 1px;
    color: #888;
}
```

### Buttons
```css
button {
    background: #00d4ff;
    color: #1a1a2e;
    border: none;
    padding: 8px 16px;
    border-radius: 4px;
    cursor: pointer;
    font-size: 13px;
    font-weight: 600;
    transition: background 0.2s;
}
button:hover { background: #00a8cc; }
button:active { transform: scale(0.98); }
.btn-secondary { background: #333; color: #eee; }
.btn-secondary:hover { background: #444; }
```

### Textarea
```css
textarea {
    width: 100%;
    background: #16213e;
    border: 1px solid #333;
    border-radius: 8px;
    padding: 16px;
    color: #eee;
    font-family: 'Monaco', 'Menlo', 'Ubuntu Mono', monospace;
    font-size: 14px;
    line-height: 1.5;
    resize: vertical;
    outline: none;
}
textarea:focus { border-color: #00d4ff; }
textarea::placeholder { color: #555; }
```

### Input Field
```css
input[type="text"] {
    width: 100%;
    background: #16213e;
    border: 1px solid #333;
    border-radius: 8px;
    padding: 14px 16px;
    color: #eee;
    font-size: 16px;
    outline: none;
}
input[type="text"]:focus { border-color: #00d4ff; }
input[type="text"]::placeholder { color: #555; }
```

### Error Display
```css
.error { color: #ff6b6b; font-size: 13px; margin-top: 8px; min-height: 20px; }
```

## Tool Patterns

### Bidirectional Converter (public/converters/)
Two panels side by side, typing in either updates the other.

```
┌─────────────────┐    ┌─────────────────┐
│ Format A        │    │ Format B        │
│ [Copy] [Clear]  │    │ [Copy] [Clear]  │
├─────────────────┤    ├─────────────────┤
│                 │ ↔  │                 │
│   textarea      │    │   textarea      │
│                 │    │                 │
└─────────────────┘    └─────────────────┘
```

**Key behavior**: Debounced input (100-500ms), convert on every keystroke.

```javascript
let timeout;
inputA.addEventListener('input', () => {
    clearTimeout(timeout);
    timeout = setTimeout(() => convertAtoB(), 100);
});
```

### Decoder/Inspector (public/decoders/)
Input at top, structured output below. Often with expandable sections.

```
┌─────────────────────────────────────┐
│ Input                    [Paste]    │
├─────────────────────────────────────┤
│ textarea                            │
└─────────────────────────────────────┘

┌─────────────────┐  ┌─────────────────┐
│ Section 1       │  │ Section 2       │
│ decoded data    │  │ decoded data    │
└─────────────────┘  └─────────────────┘
```

### Multi-Output Tool
One input, multiple outputs displayed simultaneously (e.g., text case converter).

```
┌─────────────────────────────────────┐
│ Input                               │
└─────────────────────────────────────┘

┌─────────┐ ┌─────────┐ ┌─────────┐
│Output 1 │ │Output 2 │ │Output 3 │
└─────────┘ └─────────┘ └─────────┘
```

### File Upload Tool
For tools that need external data (like Azure IP lookup).

```javascript
const fileInput = document.getElementById('fileInput');
fileInput.addEventListener('change', (e) => {
    const file = e.target.files[0];
    if (!file) return;
    const reader = new FileReader();
    reader.onload = (event) => {
        const data = JSON.parse(event.target.result);
        // process data
    };
    reader.readAsText(file);
});
```

## Common JavaScript Utilities

### Copy to Clipboard
```javascript
function copyText(id) {
    const el = document.getElementById(id);
    if (el.value) {
        navigator.clipboard.writeText(el.value).then(() => {
            const btn = event.target;
            const orig = btn.textContent;
            btn.textContent = 'Copied!';
            setTimeout(() => btn.textContent = orig, 1500);
        });
    }
}
```

### Escape HTML
```javascript
function escapeHtml(str) {
    const div = document.createElement('div');
    div.textContent = str;
    return div.innerHTML;
}
```

### Debounce Pattern
```javascript
let timeout;
input.addEventListener('input', () => {
    clearTimeout(timeout);
    timeout = setTimeout(process, 150);
});
```

## Updating index.html

When adding a new tool:

1. Add the HTML file to the appropriate directory inside `public/`
2. Add a `<li><a href="...">Tool Name</a></li>` to the matching section in `public/index.html`
3. If it's a new directory, add a new `<h2>` section

Section order in index.html:
1. Converters
2. Decoders
3. Tools
4. Browser Info

## Example: Creating a New Converter

1. Create `public/converters/foo-bar-converter.html`
2. Use the bidirectional converter pattern
3. Add to `public/index.html` under Converters section
4. Include example data on page load

## Git Workflow

- One feature = one branch
- Branch naming: `claude/<feature-name>-<session-suffix>`
- Commit message format: Brief title, bullet points for details, session link at end
