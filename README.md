# PlantUML for GitHub

Renders ` ```plantuml ` code blocks directly on GitHub pages, using the
TeaVM-compiled PlantUML engine that runs entirely client-side.

**Now with salt (wireframe) support.** `@startsalt` blocks render inline like
any other diagram — the upstream browser engine omits the salt factory
entirely and paints an error card instead. Everything else PlantUML renders in
the browser is here too; see [Supported diagrams](#supported-diagrams).

**No server. No tokens. No tracking. Zero permissions.**

> Not published to the Chrome Web Store or Firefox Add-ons.
> Install locally from a checkout — see [Install](#install).

---

## Contents

- [Install](#install)
- [Supported diagrams](#supported-diagrams)
- [Live demo](#live-demo)
- [How it works](#how-it-works)
- [Security & permissions](#security--permissions)
- [Development](#development)
- [Roadmap](#roadmap)
- [Why this extension exists](#why-this-extension-exists)
- [License](#license)

---

## Install

Clone the repository, then load the target's folder unpacked. There is nothing
to build — `Chrome/` and `Firefox/` are committed ready to load.

### Chrome

1. Open `chrome://extensions/`
2. Toggle **Developer mode** on (top-right)
3. Click **Load unpacked**
4. Select the **`Chrome/`** subfolder — not the repository root

After a rebuild, hit **↻** on the extension card, then hard-reload the GitHub tab.

### Firefox

1. Open `about:debugging#/runtime/this-firefox`
2. Click **Load Temporary Add-on…**
3. Select **`Firefox/manifest.json`**

Temporary add-ons are removed when Firefox restarts, so this needs redoing each
session.

Recognised fence languages: `plantuml`, `puml`, `wsd`.

---

## Supported diagrams

The bundled engine is a subset of full PlantUML — the browser build registers
its own factory list, so not every `@start…` type is available.

### Rendered

| Type | Opens with | Layout |
| --- | --- | --- |
| **Salt (wireframes)** | **`@startsalt`** | **native** |
| Sequence | `@startuml` | native |
| Class / object | `@startuml` | Graphviz |
| Activity | `@startuml` | native |
| State | `@startuml` | Graphviz |
| Component / deployment / use case | `@startuml` | Graphviz |
| Timing | `@startuml` | native |
| Gantt | `@startgantt` · `@startproject` | native |
| Mindmap | `@startmindmap` | native |
| WBS | `@startwbs` | native |
| Network | `@startnwdiag` | native |
| Packet | `@startpacketdiag` | native |
| JSON | `@startjson` | native |
| YAML | `@startyaml` | native |
| EBNF | `@startebnf` | native |
| Regex | `@startregex` | native |
| Chart | `@startchart` | native |
| Creole | `@startcreole` | native |

### Not available in the browser build

`@startditaa`, `@startjcckit`, `@startmath`, `@startlatex`, `@startwire`,
`@startboard`, `@startflow`, `@startgit`, `@starthcl`, `@startchen`,
`@startbpm`, `@startchronology`, `@startfiles`, `@startdot`.

These render an error card rather than a diagram.

### Salt notes

`@startsalt` must open the block. Two forms **do not** work — both fail the
same way in upstream PlantUML, so this is not a limitation of the extension:

- `@startuml` followed by a `salt` line
- `{{ salt … }}` embedded in a note or label

---

## Live demo

With the extension active, this renders inline as a wireframe:

```plantuml
@startsalt
{+
  {* File | Edit | Help }
  {
    Name     | "                    "
    Password | "                    "
    [X] Remember me
  }
  [Cancel] | [   OK   ]
}
@endsalt
```

To try it yourself, put that block in an issue, discussion, or README in a
repo you own, then reload the page.

---

## How it works

1. A content script scans each GitHub page for `plantuml` code blocks.
2. Each block is replaced with a sandboxed `<iframe>` packaged inside the extension.
3. The iframe loads the TeaVM-compiled `plantuml.js` engine and renders to SVG.
4. The SVG is displayed inline, inside a wrapper with a header bar.

The header bar offers a **source toggle** (`<>`), **Copy as bitmap / SVG**
(also on right-click), and **Edit as draft** — a two-column editor with live
preview. See [HISTORY.md](HISTORY.md) for the full feature log.

This is the same architecture GitHub already uses for Mermaid.

---

## Security & permissions

The extension declares **zero permissions** — no host permissions, no storage,
no tabs API. It ships a content script scoped to `github.com` and a packaged
renderer page. The engine runs inside a sandboxed iframe with an opaque origin,
no network access, and no shared state with the host page.

One deviation from the Manifest V3 default CSP is required:

```json
"content_security_policy": {
  "extension_pages": "script-src 'self' 'wasm-unsafe-eval'; object-src 'self'"
}
```

Salt and sequence diagrams render straight to SVG, but anything needing graph
layout — class, component, deployment, state, use-case — is laid out by
**Graphviz, shipped as a WebAssembly module** (`viz-global.js`). Instantiating
it requires `'wasm-unsafe-eval'`.

Despite the name, that directive **only** permits WebAssembly compilation and
instantiation. It does not re-enable `eval()` or `new Function()`, and
`script-src 'self'` still blocks all remote scripts. Google documents it as the
supported way to ship WASM in MV3.

---

## Development

### Repository layout

| Path | Role |
| --- | --- |
| `template/` | Single source for `content.js`, `renderer.js`, `renderer.html`, `manifest.json` |
| `template.py` | Preprocesses `#if CHROME` / `#if FIREFOX` into `Chrome/` and `Firefox/` |
| `Chrome/`, `Firefox/` | **Generated** — never edit directly |
| `*/vendor/` | Engine bundles, placed per target by hand |

After editing anything in `template/`, run:

```bash
python3 template.py
```

### Engine bundles

The two targets vendor **different builds of the same engine**: Chrome loads
one minified ES module, Firefox loads an unminified build pre-split into
chunks. [FIREFOX.md](FIREFOX.md) explains why and how to regenerate them.

### Packaging (optional)

ZIPs are not needed for local install, but the builders still work:

```bash
python3 build_zip_chrome.py
```

```bash
python3 build_zip_firefox.py
```

---

## Roadmap

- [x] MVP: detect and render `plantuml` blocks
- [x] Firefox support (Manifest V3)
- [x] Copy as SVG / bitmap, source toggle, edit as draft
- [x] Theme matching (light/dark) — follows GitHub's color mode
- [x] `puml` and `wsd` language aliases
- [x] Salt (wireframe) diagrams
- [ ] Options page (toggle, performance settings)

---

## Why this extension exists

PlantUML support on GitHub has been requested for 4+ years:
<https://github.com/orgs/community/discussions/10111>

The blocker was performance and infrastructure cost. With the TeaVM-compiled
engine, **that blocker no longer exists** — PlantUML runs natively on
github.com with zero server-side changes, using the same sandbox pattern
GitHub already uses for Mermaid.

If you'd like to see this integrated natively, please **upvote the
discussion** linked above.

---

## License

MIT
