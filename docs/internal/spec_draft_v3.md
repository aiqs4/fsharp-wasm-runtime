**fsharp-wasm-runtime — Complete Project Specification**  
**Version 2.0 — Full Scope, Production-Ready Runtime for F# → WasmGC**

### Project Goal

`fsharp-wasm-runtime` is the official, standalone browser runtime for the FSharp-WASM compiler (WasmGC target).  

It is **not** a UI framework.  
It is **the complete JavaScript glue + typed F# Browser BCL** that makes the browser feel like a first-class F# platform.

The runtime turns every major browser capability into clean, typed, handle-based F# modules so that application code stays 100 % F# while the thin JS layer stays invisible and minimal.

**Final vision**: Any developer can write a full-featured browser application (or library) in F#, compile it once with `fsharp-wasm`, and get native performance + full browser access with zero JavaScript application code.

### Fundamental Architecture (Unchanging)

```
F# source
   ↓ (fsharp-wasm compiler → WasmGC)
.wasm module
   ↓ (instantiated in browser)
JS glue layer (thin, modular, < 400 lines total when complete)
   ↓
Browser APIs (DOM, Canvas, Web Components, Fetch, WebSocket, Storage, IndexedDB,
              Workers, Pub/Sub, WebGPU, Timing, Events, etc.)
```

The JS glue is **required by the WASM standard** and will never be eliminated. Our job is to make it perfectly structured, extensible, and invisible to F# developers.

### Repository Name & Namespace (Final Decision)

- **Repository**: `fsharp-wasm-runtime` (kept for clarity and discoverability)
- **Top-level F# namespace**: `Browser`

All modules live under `Browser.*` exactly like a real BCL:

```fsharp
open Browser.Dom
open Browser.Canvas
open Browser.Net
open Browser.Storage
open Browser.Components
open Browser.PubSub
// etc.
```

This matches the “Browser.BCL Design” philosophy discussed in the glue conversations and gives the cleanest possible developer experience.

### Repository Structure (Final)

```
fsharp-wasm-runtime/
├── runtime/                          # All JS glue – pure ES modules
│   ├── core.js                       # Shared handle store + pre-registered IDs
│   ├── dom.js
│   ├── events.js
│   ├── canvas.js
│   ├── components.js
│   ├── net.js                        # fetch + WebSocket
│   ├── pubsub.js                     # in-process publish/subscribe
│   ├── storage.js                    # localStorage / sessionStorage
│   ├── indexeddb.js
│   ├── workers.js
│   ├── timing.js                     # setTimeout / setInterval / requestAnimationFrame
│   └── gpu.js                        # WebGPU (low-level)
│
├── src/                              # F# source – pure F#
│   ├── Browser/
│   │   ├── Core.fs                   # Handle, nil, body, runtime helpers
│   │   ├── Dom.fs
│   │   ├── Events.fs
│   │   ├── Canvas.fs
│   │   ├── Components.fs
│   │   ├── Net.fs
│   │   ├── PubSub.fs
│   │   ├── Storage.fs
│   │   ├── IndexedDb.fs
│   │   ├── Workers.fs
│   │   ├── Timing.fs
│   │   └── Gpu.fs
│   ├── Html/                         # Declarative DSL (immediate-mode, extensible to vdom later)
│   │   ├── Elements.fs
│   │   ├── Attrs.fs
│   │   └── Events.fs
│   └── Elmish/                       # Minimal MVU engine (web-targeted, written from scratch)
│       └── Core.fs
│
├── tests/                            # Testing
│   ├── Unit/                         # F# unit tests (xUnit or Expecto)
│   └── Integration/                  # Browser integration tests (Playwright)
│
├── demos/                            # Complete showcase applications
│   ├── counter/
│   ├── todo/
│   ├── canvas-mandelbrot/
│   ├── web-component/
│   ├── realtime-pubsub/
│   ├── indexeddb-crud/
│   ├── worker-compute/
│   ├── webgpu-triangle/              # Phase 4
│   └── full-app/                     # End-to-end example
│
├── build/                            # Build & release pipeline
│   ├── build.js                      # Node script for JS + WASM bundling
│   └── release.sh
├── index.html                        # Main demo / documentation page (PicoCSS)
├── package.json                      # Only for dev tools (serve, playwright, etc.)
└── README.md
```

### Full Roadmap (All Browser Features)

Every capability mentioned in the glue discussions is explicitly included. No shortcuts.

#### Phase Core – Foundation (always first)
- Handle-based runtime (`core.js`)
- DOM (create, append, text, attributes, classes, styles, query, remove, clear)
- Events (click, input, change, submit, custom, with proper cleanup)
- Minimal Elmish MVU engine (pure F#, compiles through the pipeline)
- HTML DSL (declarative `div`, `button`, `input`, etc. with immediate rendering)
- Basic demos (Counter, Todo)

#### Phase DOM & Rendering
- Full Canvas 2D (contexts, fillRect, paths, images, text, pixel manipulation)
- Web Components (native `customElements.define` with F# factory support)
- Declarative rendering improvements (clearAndRender → optional diffing once `seq` lands)
- PicoCSS integration (zero-config, classless semantic HTML)

#### Phase Communication
- Fetch (text, JSON, binary, headers, POST, etc.)
- WebSocket (open, send, onmessage, onerror, close, binary support)
- Pub/Sub (in-process, topic-based, for internal graph/events/AI messaging)

#### Phase Storage
- localStorage / sessionStorage (sync)
- IndexedDB (open, put, get, delete, transactions, object stores – Promise-wrapped cleanly in F#)

#### Phase Concurrency & Scheduling
- Web Workers (create, postMessage, onmessage, terminate)
- Message passing between main thread ↔ workers (typed via handles)
- Timing (timeout, interval, requestAnimationFrame, cancel)
- Combined Worker + PubSub pattern (mini actor system)

#### Phase Advanced Graphics & Compute
- WebGPU (adapter, device, shaders, buffers, pipelines – low-level but fully exposed)
- Canvas + WebGPU interop examples

#### Phase Polish & Extensibility
- Unified Event System (DOM + WS + Workers + PubSub in one model)
- Error handling & logging bridge (JS exceptions → F#)
- Resource cleanup (drop handles, remove listeners)
- Performance batching utilities
- Full async support (once compiler has Tasks / async {})
- Optional Lit adapter layer (as plugin, never core)

### Elmish in This Runtime

`Browser.Elmish` is **deliberately minimal and web-targeted**.  
It is **not** a port of Fable.Elmish — it is the exact ~150-line core needed for MVU in WasmGC.  
It lives inside the runtime because it is the natural way to structure browser apps in this environment.  
Later it can be extracted into its own tiny package if desired.

### Build / Release Pipeline

- `dotnet fable` (or the fsharp-wasm CLI) compiles F# → WASM
- Simple Node.js build script (`build.js`) that:
  - Copies all `runtime/*.js` to `dist/`
  - Validates the WASM module (`wasm-tools`)
  - Generates a tiny loader (`runtime.js`) that exports `instantiate` + all entry points
- Release process (`release.sh`):
  - Builds WASM + JS
  - Runs all tests
  - Publishes GitHub release with `dist/` bundle
  - Deploys demo to GitHub Pages
- No heavy bundlers required (ES modules only)

### Testing Strategy

- **Unit tests** (`tests/Unit/`): Pure F# tests for Elmish, HTML DSL, PubSub, storage wrappers, etc. (runs in dotnet test via the compiler)
- **Integration tests** (`tests/Integration/`): Playwright tests that:
  - Start a static server
  - Load `index.html`
  - Verify every demo works end-to-end
  - Test Web Component embedding
  - Test Worker ↔ main thread messaging
  - Test IndexedDB persistence across reloads
- CI: GitHub Actions runs full build + all tests on every PR

### Demo Page (`index.html`)

Single beautiful page (PicoCSS) that showcases **every** capability:
- All Phase Core demos
- Canvas compute demo
- Web Component live embedding
- Real-time Pub/Sub + WebSocket demo
- IndexedDB CRUD demo
- Worker offloading demo
- WebGPU triangle (when ready)
- Clear “View Source” links to both F# and JS glue

### Success Definition (Full Goal Achieved)

When this runtime is complete:

- A developer can write **any** browser application using only F# modules under the `Browser.*` namespace.
- The JS glue remains < 400 lines total and is perfectly modular.
- Every browser API mentioned in the original glue discussions is supported with clean, typed F# bindings.
- The project serves as the reference “Browser BCL” for the entire FSharp-WASM ecosystem.
- Demos prove that F# + WasmGC is a viable, modern frontend platform.

This specification is the complete, final blueprint.  
It incorporates every idea from the glue conversations (WASM-JS-GLUE-0x.md files), the spec.md draft, and the fsharp-wasm-runtime-spec.md while expanding them into a production-grade, fully featured runtime.

Implement this structure and the F# → Browser story is solved.
