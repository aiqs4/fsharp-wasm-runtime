Here's the **extension** to the existing `fsharp-wasm-runtime` specification (Version 2.0). It adds two new dedicated sections that address your requests:

- A clean **F# CSS specification** (under the `Browser` namespace, lightweight and aligned with PicoCSS philosophy).
- **Build & Release Pipeline** instructions, including how to consume artifacts from the `fsharp-wasm` compiler (soon available as downloadable modified Fable + fsharp-wasm builds).

These sections integrate directly into the full spec without shortening any browser features.

### Added Section: F# CSS Specification (Browser.Css)

**Goal**  
Provide a minimal, type-safe, declarative way to work with CSS in pure F# while staying lightweight. No full CSS-in-JS engine, no runtime parsing overhead. Focus on common needs: classes, inline styles, themes, and PicoCSS integration.

**Design Principles**
- Classless-first (PicoCSS default) → semantic HTML gets nice defaults automatically.
- Optional typed helpers for classes and styles.
- No build step for CSS (pure `<link>` or inline).
- Extensible for future themes, scoped styles, or CSS modules.
- Everything lives under `Browser.Css`.

**F# API (src/Browser/Css.fs)**

```fsharp
module Browser.Css

open Browser.Dom  // for Element

/// Simple class list builder (space-separated)
type ClassList = string list

module Class =
    let inline add (cls: string) = [cls]
    let inline many (classes: string list) = classes
    let inline combine (a: ClassList) (b: ClassList) = a @ b
    let inline toString (classes: ClassList) = String.concat " " classes

/// Inline style as key-value pairs (for setStyle)
type StyleProp = string * string

module Style =
    let inline prop key value : StyleProp = (key, value)
    let inline background color = prop "background" color
    let inline color c = prop "color" c
    let inline fontSize size = prop "font-size" size
    let inline width w = prop "width" w
    let inline height h = prop "height" h
    let inline padding p = prop "padding" p
    // ... add more as needed

    let inline toString (styles: StyleProp list) =
        styles |> List.map (fun (k,v) -> $"{k}:{v}") |> String.concat "; "

/// PicoCSS theme helpers (light/dark)
module Theme =
    let light = "data-theme=\"light\""
    let dark  = "data-theme=\"dark\""

    /// Apply theme to body or root element
    let applyTo (el: Element) theme =
        setAttr el "data-theme" (if theme = "dark" then "dark" else "light")

/// Convenience for common PicoCSS utility patterns (still semantic-first)
module Pico =
    let inline card children = el "article" [class_ "card"] children  // example wrapper
    let inline buttonPrimary = class_ "primary"
    let inline buttonSecondary = class_ "secondary"
    let inline container = class_ "container"
    let inline grid = class_ "grid"

// Usage example in Html DSL (Elements.fs)
let myView =
    div [class_ "grid"; Style.background "#f8f9fa" |> Style.prop ... ] [
        article [Pico.card] [ ... ]
        button [Pico.buttonPrimary; onClick_ handler] [text "Click"]
    ]
```

**Integration with HTML DSL**  
Extend `Attr` type in `Html/Attrs.fs`:

```fsharp
type Attr =
    // ... existing
    | Classes of ClassList
    | Styles of StyleProp list
    | DataTheme of string
```

Update `applyAttr` to handle them via `setClass`, `setStyle`, etc.

**CSS File Strategy (recommended)**
- Include PicoCSS via CDN in `index.html` (no F# change needed):
  ```html
  <link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/@picocss/pico@2/css/pico.min.css">
  ```
- For custom CSS: optional `static/css/app.css` with manual classes. Reference via `<link>` or `setAttr` on `<style>` element.
- Advanced (future): Support for CSS custom properties via `setStyle` on `:root`.

This keeps CSS **declarative in F#** but **zero-overhead** and compatible with the classless PicoCSS approach.

### Added Section: Build & Release Pipeline

**Overview**  
The pipeline consumes the **fsharp-wasm** compiler (and its modified Fable fork) to compile the runtime's F# sources into a single `.wasm` module, then bundles it with the thin JS glue for browser consumption.

**Current & Upcoming Compiler Artifacts (from https://github.com/fsharp-wasm/fsharp-wasm)**  
- The compiler is built from source today (`dotnet build src/Fable.WasmGc.fsproj`).
- Soon: downloadable artifacts will be published (modified Fable + fsharp-wasm CLI/binary).
- Compilation command (stable):
  ```bash
  dotnet fable <path-to-fsproj> --lang wasmgc --output dist/
  ```
  This produces:
  - `dist/app.wasm` (main binary)
  - `dist/app.wat` (text format for debugging)
  - Optional WIT file for Component Model.

**Project Build Script (build/build.js or package.json scripts)**

Add to `package.json` (dev-only):

```json
{
  "scripts": {
    "build:wasm": "dotnet fable src/FSharpWasmRuntime.fsproj --lang wasmgc --output dist/wasm/",
    "build:glue": "cp -r runtime/*.js dist/",
    "build": "npm run build:wasm && npm run build:glue",
    "validate": "wasm-tools validate dist/wasm/app.wasm",
    "serve": "npx serve .",
    "test": "playwright test"
  }
}
```

**Full Release Pipeline (build/release.sh)**

```bash
#!/bin/bash
set -e

echo "=== Building fsharp-wasm-runtime ==="

# 1. Ensure compiler artifact is available
# Option A (soon): Download pre-built artifact
# curl -L -o compiler.zip https://github.com/fsharp-wasm/fsharp-wasm/releases/download/vX.Y.Z/fsharp-wasm-artifact.zip
# unzip compiler.zip -d /tmp/compiler

# Option B (current / fallback): Build from source if needed
# git clone https://github.com/fsharp-wasm/fsharp-wasm /tmp/fsharp-wasm || true
# cd /tmp/fsharp-wasm && dotnet build src/Fable.WasmGc.fsproj

# 2. Compile runtime F# sources
dotnet fable src/FSharpWasmRuntime.fsproj \
  --lang wasmgc \
  --output dist/wasm/ \
  --define RELEASE

# 3. Copy & prepare JS glue (thin layer)
mkdir -p dist/js
cp runtime/*.js dist/js/
cat > dist/js/runtime.js << EOF
// Tiny loader
export async function instantiate(wasmUrl) {
  const response = await fetch(wasmUrl);
  const bytes = await response.arrayBuffer();
  const { instance } = await WebAssembly.instantiate(bytes, {
    // imports for Browser.* modules go here if needed
  });
  return instance;
}
EOF

# 4. Validate
wasm-tools validate dist/wasm/app.wasm

# 5. Build demo page
cp index.html dist/

# 6. Run tests
npm test

# 7. Package release
mkdir -p release
cp -r dist/* release/
zip -r fsharp-wasm-runtime-v$(date +%Y%m%d).zip release/

echo "Release ready in release/ folder."
```

**How to Consume Upcoming fsharp-wasm Artifact**

1. Download the artifact (ZIP containing modified `dotnet fable` + WasmGC backend).
2. Extract and add the binary to your PATH, or use it via `dotnet tool` if packaged that way.
3. In CI / local build, replace the `dotnet fable` call with the artifact version for reproducibility.
4. The runtime's `src/` remains a standard F# project (multi-file `.fsproj` referencing `Browser.*` modules).
5. FFI declarations (`[<Import("functionName", "./relative/path/to/js/module.js")>]`) stay exactly as defined — the compiler emits the correct import section in the `.wasm`.

**GitHub Actions Example (for CI/release)**

- Trigger on push to main / tags.
- Checkout both `fsharp-wasm-runtime` and (if needed) `fsharp-wasm`.
- Run the build script above.
- Upload `dist/` as artifact + create GitHub Release with the ZIP.
- Deploy `dist/` to GitHub Pages for the live demo.

This pipeline keeps everything **simple**, **reproducible**, and ready for when official fsharp-wasm artifacts become available. No heavy bundlers — pure ES modules + static files.

---

These two extensions complete the specification:

- **Browser.Css** gives you a natural F# way to handle styling without fighting the runtime.
- The **build/release** section makes the project production-ready and future-proof with the upcoming compiler artifacts.

The full runtime still covers **every** browser capability mentioned earlier (DOM, Canvas, Web Components, Net, Storage, IndexedDB, Workers, PubSub, Timing, WebGPU, unified events, etc.) under the clean `Browser.*` namespace.

You can now copy-paste this extension directly into the previous Version 2.0 document. Let me know if you want the entire merged spec in one file or further refinements (e.g., more CSS helpers or CI YAML).
