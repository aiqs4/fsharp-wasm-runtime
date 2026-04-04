**fsharp-wasm-runtime — Complete Project Specification**  
**Version 1.0** — Focused, honest, and scoped for real delivery

### Core Purpose

`fsharp-wasm-runtime` is a **thin, typed browser runtime** for the FSharp-WASM compiler (WasmGC target).  

It provides the minimal JavaScript glue + clean F# bindings needed to write real browser applications entirely in F#, compiled to WebAssembly with garbage collection, **without** depending on JavaScript application code, React, Fable, Feliz, or any existing JS-first framework.

The project proves that F# → WasmGC can be a practical frontend platform for demos and small-to-medium applications.

### Fundamental Constraints (Non-Negotiable)

- WASM cannot access the DOM directly. The JS glue layer is **required** and will remain required.
- The glue must stay **thin** (ideally < 200 lines of JS total in Phase 1).
- No virtual DOM in Phase 1 (no `seq`, no sophisticated diffing yet).
- No async/await or Tasks in Phase 1 (those come later when the compiler supports them).
- No external UI libraries (no Feliz, no Lit as core dependency, no React).
- CSS is handled purely via PicoCSS (one `<link>` tag, classless by default). No CSS-in-F# bindings needed.

### What This Project Is

- A separate repository from the compiler.
- JS glue modules (plain ES modules, no build step).
- Typed F# bindings using `[<Import>]` that work with the FSharp-WASM compiler.
- A minimal Elmish-style MVU engine written purely in F#.
- A small, declarative HTML DSL that feels natural in F#.
- 3–4 compelling demos that run 100% in WASM (logic + rendering loop).
- Clear documentation showing “F# does the work, JS only provides the bridge”.

### What This Project Is NOT

- Not a full UI framework.
- Not a port of Fable.Browser or Fable.Elmish.
- Not a React/Feliz clone.
- Not an attempt to eliminate the JS glue layer.
- Not dependent on any npm packages except for local development serving.

### Repository Structure

```
fsharp-wasm-runtime/
├── runtime/                  # JavaScript glue (thin, modular, ES modules)
│   ├── core.js               # Shared handle store + pre-registered objects
│   ├── dom.js                # DOM creation, manipulation, attributes
│   ├── events.js             # Event listeners (click, input, etc.)
│   ├── canvas.js             # Canvas 2D (for compute demos)
│   └── components.js         # Native Web Components support
│
├── src/                      # F# code (compiled by fsharp-wasm)
│   ├── Runtime/
│   │   ├── Core.fs           # Handle type, nil, body
│   │   ├── Dom.fs            # Element wrapper + operations
│   │   └── Events.fs         # Event helpers
│   ├── Html/
│   │   ├── Elements.fs       # DSL: div, button, text, etc.
│   │   └── Attrs.fs          # class_, id_, onClick_, etc.
│   └── Elmish/
│       └── Core.fs           # Minimal MVU (Program, Cmd, run)
│
├── demos/
│   ├── counter/              # Classic counter (MVU)
│   ├── todo/                 # Todo list with input + list ops
│   ├── canvas/               # Mandelbrot or simple drawing (compute showcase)
│   └── web-component/        # Defines native <fs-counter> custom element
│
├── index.html                # Single-page demo showcase (PicoCSS)
├── dist/                     # Build output (generated)
└── README.md
```

### Phase 1 Scope (Deliverable in 3–4 Weeks)

**Goal:** Have a polished, public demo page where visitors see real F# code running in the browser via WasmGC.

#### 1. JS Glue Layer (runtime/)

**core.js** – Handle management (the heart of the glue)

```javascript
const store = new Map();
let nextId = 2; // 0 = nil, 1 = document.body

export const save = (obj) => { const id = nextId++; store.set(id, obj); return id; };
export const get  = (id)  => store.get(id);
export const drop = (id)  => store.delete(id);

store.set(0, null);
store.set(1, document.body);
```

**dom.js** – Core DOM operations

```javascript
import { save, get, drop } from "./core.js";

export const createElement = (tag) => save(document.createElement(tag));
export const createTextNode = (text) => save(document.createTextNode(text));
export const setTextContent = (id, text) => { const el = get(id); if (el) el.textContent = text; };
export const appendChild = (parentId, childId) => {
  const p = get(parentId), c = get(childId);
  if (p && c) p.appendChild(c);
};
export const setAttribute = (id, key, value) => { const el = get(id); if (el) el.setAttribute(key, value); };
export const setClassName = (id, cls) => { const el = get(id); if (el) el.className = cls; };
export const clearInnerHTML = (id) => { const el = get(id); if (el) el.innerHTML = ""; };
```

**events.js**

```javascript
import { get } from "./core.js";

export const addClickListener = (id, callback) => {
  const el = get(id);
  if (el) el.addEventListener("click", () => callback(id));
};

export const addInputListener = (id, callback) => {
  const el = get(id);
  if (el) el.addEventListener("input", (e) => callback(e.target.value));
};
```

**components.js** (simple)

```javascript
import { save } from "./core.js";

export const defineComponent = (name, factory) => {
  customElements.define(name, class extends HTMLElement {
    connectedCallback() {
      factory(save(this));
    }
  });
};
```

**canvas.js** (minimal for Phase 1)

```javascript
import { save, get } from "./core.js";

export const getContext2D = (canvasId) => {
  const canvas = get(canvasId);
  return canvas ? save(canvas.getContext("2d")) : 0;
};

export const fillRect = (ctxId, x, y, w, h) => {
  const ctx = get(ctxId);
  if (ctx) ctx.fillRect(x, y, w, h);
};

export const setFillStyle = (ctxId, style) => {
  const ctx = get(ctxId);
  if (ctx) ctx.fillStyle = style;
};
```

#### 2. F# Bindings (src/Runtime/)

**Core.fs**

```fsharp
module FSharpWasm.Runtime.Core

type Handle = int

let nil : Handle = 0
let body : Handle = 1
```

**Dom.fs** (typed, clean wrapper)

```fsharp
module FSharpWasm.Runtime.Dom

open Core

[<Import("createElement", "./dom.js")>] extern Handle _createElement(string tag)
[<Import("createTextNode", "./dom.js")>] extern Handle _createText(string text)
[<Import("setTextContent", "./dom.js")>] extern unit _setText(Handle id, string text)
[<Import("appendChild", "./dom.js")>] extern unit _append(Handle p, Handle c)
[<Import("setAttribute", "./dom.js")>] extern unit _setAttr(Handle id, string k, string v)
[<Import("setClassName", "./dom.js")>] extern unit _setClass(Handle id, string c)
[<Import("clearInnerHTML", "./dom.js")>] extern unit _clear(Handle id)

type Element = private El of Handle

module Element =
    let create tag = El (_createElement tag)
    let text t     = El (_createText t)
    let ofBody     = El body

let append (El p) (El c) = _append p c
let setText (El e) t     = _setText e t
let setAttr (El e) k v   = _setAttr e k v
let setClass (El e) c    = _setClass e c
let clear (El e)         = _clear e
```

**Events.fs**

```fsharp
module FSharpWasm.Runtime.Events

[<Import("addClickListener", "./events.js")>] extern unit _onClick(Handle id, (Handle -> unit) cb)
[<Import("addInputListener", "./events.js")>] extern unit _onInput(Handle id, (string -> unit) cb)

let onClick (El id) f     = _onClick id (fun _ -> f())
let onInput (El id) f     = _onInput id f
```

#### 3. HTML DSL (src/Html/)

A simple, immediate-mode DSL (no vdom yet).

```fsharp
module FSharpWasm.Html

open Runtime.Dom
open Runtime.Events

type Node =
    | El of tag:string * attrs:Attr list * children:Node list
    | Txt of string
    | Empty

and Attr =
    | Class of string
    | Attr of key:string * value:string
    | OnClick of (unit -> unit)
    | OnInput of (string -> unit)

let rec render (parent: Element) (node: Node) =
    match node with
    | Empty -> ()
    | Txt t ->
        let el = Element.text t
        append parent el
    | El(tag, attrs, children) ->
        let el = Element.create tag
        attrs |> List.iter (fun a -> applyAttr el a)
        children |> List.iter (render el)
        append parent el

and applyAttr el = function
    | Class c       -> setClass el c
    | Attr(k,v)     -> setAttr el k v
    | OnClick f     -> onClick el f
    | OnInput f     -> onInput el f

let div attrs children = El("div", attrs, children)
let button attrs children = El("button", attrs, children)
let text s = Txt s
let class_ c = Class c
let onClick_ f = OnClick f
// ... more helpers as needed
```

#### 4. Minimal Elmish (src/Elmish/Core.fs)

```fsharp
module FSharpWasm.Elmish

type Dispatch<'msg> = 'msg -> unit
type Cmd<'msg> = (Dispatch<'msg> -> unit) list

module Cmd =
    let none = []
    let ofMsg msg = [fun d -> d msg]

type Program<'model,'msg> = {
    Init: unit -> 'model * Cmd<'msg>
    Update: 'msg -> 'model -> 'model * Cmd<'msg>
    View: 'model -> Dispatch<'msg> -> Node
}

let run (mountPointId: string) (program: Program<'model,'msg>) =
    let container = Element.ofBody // or get by id later
    let mutable model, cmds = program.Init()

    let rec dispatch msg =
        let newModel, newCmds = program.Update msg model
        model <- newModel
        render()
        newCmds |> List.iter (fun c -> c dispatch)

    and render () =
        clear container
        let view = program.View model dispatch
        render container view   // note: render is overloaded or renamed

    render()
    cmds |> List.iter (fun c -> c dispatch)
```

### Demos (Phase 1)

1. **Counter** — Classic increment/decrement/reset with Elmish MVU.
2. **Todo** — Add, toggle, remove todos (shows lists, input handling).
3. **Canvas Mandelbrot** — Pure F# computation rendering to `<canvas>` (shows WASM strength).
4. **Web Component** — F# registers a native `<fs-counter>` custom element that can be dropped into any HTML page.

### index.html

Single clean page using PicoCSS that hosts all demos side-by-side, with source links and clear messaging:  
“Application logic runs in WASM. Only thin JS glue for browser APIs.”

### Success Criteria for Phase 1

- All JS glue is < 150 lines total.
- Counter and Todo run smoothly with full MVU loop in F#.
- Mandelbrot renders correctly using only F# math + thin Canvas bridge.
- Web Component demo works when embedded in plain HTML.
- The whole demo page loads and runs from a static server with zero JS app code.
- README clearly explains the architecture and why the JS glue is the right approach.

This specification is deliberately **not** trying to build a full framework.  
It is the smallest credible demonstration that F# compiled to WasmGC can be a serious option for browser UIs.

Once this Phase 1 is solid and the compiler gains `async` + `seq`, we can naturally extend the runtime with fetch, WebSocket, better rendering, etc.

This is the focused, honest spec we should implement first.
