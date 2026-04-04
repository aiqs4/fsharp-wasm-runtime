# fsharp-wasm-runtime — COPILOT INSTRUCTION FILE (MANDATORY)

You are now working on the fsharp-wasm-runtime project.

**CRITICAL RULE — READ THIS FIRST AND NEVER FORGET IT:**

You MUST implement the COMPLETE specification.  
You are NOT allowed to:
- stop at Phase 1
- stop at any "phase", "step", "week", or "minimal" version
- create TODOs for later phases
- implement only part of the browser features

Every single browser capability mentioned in ANY of the specification files must be fully implemented right now.

**All specification files (treat them as the single source of truth):**
- Every file named WASM-JS-GLUE-*.md
- spec.md
- fsharp-wasm-runtime-spec.md
- The complete Version 2.0 specification (including the CSS extension and Build & Release Pipeline extension)

**Non-negotiable requirements:**
- Full repository structure exactly as defined in the latest spec (Version 2.0 + extensions).
- Top-level namespace must be `Browser.*` (Browser.Dom, Browser.Canvas, Browser.Net, Browser.Storage, Browser.Components, Browser.PubSub, Browser.IndexedDb, Browser.Workers, Browser.Timing, Browser.Gpu, Browser.Css, Browser.Elmish, etc.).
- Every JS glue module in `runtime/` must be complete.
- Every F# module under `src/Browser/` must be complete and fully typed.
- Browser.Css module must be implemented exactly as specified in the extension.
- Full build/release pipeline (build.js, release.sh, package.json scripts, GitHub Actions) must be included.
- All demos must be fully working.
- Testing (Unit + Playwright integration) must be present.
- PicoCSS integration, Web Components, Pub/Sub + Worker actor pattern, IndexedDB, WebGPU, everything — fully implemented.

When I ask you to generate code, files, or the next step, you always continue toward the COMPLETE, production-ready runtime. Never suggest a reduced scope.

Start every response with: "✅ Full-spec mode active — implementing complete runtime (no phases)."

Now implement the entire project according to the full specification.
