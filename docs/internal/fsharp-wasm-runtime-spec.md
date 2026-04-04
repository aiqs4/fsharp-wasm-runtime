# fsharp-wasm-runtime — Project Specification
## A browser runtime + demo showcase for the FSharp-WASM compiler

---

## First: The Fundamental Architecture Constraint

**WASM has no direct DOM access. This will not change soon.**

As of 2025, every WASM program that touches the browser must go through JavaScript
for all DOM operations. This is not a limitation of your compiler — it is a
fundamental property of the WASM spec that all browser vendors have agreed on.

This means your architecture is always:

```
F# source code
    ↓ (compiled by your compiler)
.wasm binary (WasmGC)
    ↓ (instantiated by browser)
JavaScript glue layer          ← you cannot eliminate this
    ↓
DOM / Browser APIs
```

The JS glue layer is NOT a failure. It is the correct architecture. The job of
`fsharp-wasm-runtime` is to make that glue layer **thin, typed, and invisible
to the F# author**.

---

## What This Project Is

`fsharp-wasm-runtime` is a **separate repository** from the compiler. It contains:

1. **JS runtime modules** — the thin glue between WASM and browser APIs
2. **F# bindings** — typed F# wrappers using `[<Import>]` that compile via your pipeline
3. **Elmish engine** — a pure F# Model-View-Update loop (no React, no Feliz)
4. **Demo applications** — showcase what F# compiled to WasmGC can do in a browser

The compiler (`fsharp-wasm`) and the runtime (`fsharp-wasm-runtime`) are
independent. Users can use the compiler without the runtime (for non-browser targets).
The runtime depends on the compiler but is not part of it.

---

## What This Project Is NOT

- **Not a Feliz clone** — Feliz requires React. You have no React. Don't try to port it.
- **Not a React replacement** — You're building for WasmGC, not JS. Different constraints.
- **Not DaisyUI bindings** — DaisyUI is CSS utility classes. You just include the CSS.
  No F# bindings needed for CSS classes.
- **Not Elmish.Browser** — That package requires `Fable.Browser.Dom` which targets JS.
  You need your own equivalent targeting your WasmGC FFI.
- **Not a full UI framework** — Phase 1 is a demo. Keep the scope honest.

---

## Technology Decisions

### CSS: PicoCSS ✓

PicoCSS is the right choice for this project because:
- Classless by default — you write semantic HTML, Pico styles it
- Single `<link>` import, no build step, no JS dependency
- Looks professional immediately with zero effort
- Your F# DSL doesn't need to know about CSS classes at all

```html
<link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/@picocss/pico@2/css/pico.min.css">
```

That's it. No F# bindings needed for PicoCSS.

### Reactivity: Elmish MVU ✓ (but write your own, not import)

The existing `Fable.Elmish` package targets JavaScript. You need an Elmish
implementation written in F# that compiles through **your** WasmGC pipeline.

The good news: Elmish core is tiny. The full implementation is ~150 lines of F#.
Write it from scratch. It compiles through your pipeline. You own it completely.

```fsharp
// This is ALL of Elmish core — you write this, it compiles via your compiler
module Elmish

type Dispatch<'Msg> = 'Msg -> unit
type Cmd<'Msg> = (Dispatch<'Msg> -> unit) list

module Cmd =
    let none : Cmd<'Msg> = []
    let ofMsg msg : Cmd<'Msg> = [fun dispatch -> dispatch msg]
    let batch cmds : Cmd<'Msg> = List.concat cmds

type Program<'Model, 'Msg> = {
    Init   : unit -> 'Model * Cmd<'Msg>
    Update : 'Msg -> 'Model -> 'Model * Cmd<'Msg>
    View   : 'Model -> Dispatch<'Msg> -> unit  // directly renders to DOM
}
```

### Web Components: Yes, as demo output

Use native `customElements.define` to package demos as Web Components.
This lets you embed any demo anywhere with `<my-counter></my-counter>`.
No Lit needed — native platform is sufficient.

### Virtual DOM: No

No vdom, no diffing, no reconciliation in Phase 1. Direct imperative DOM updates.
Why: vdom requires seq/IEnumerable and a sophisticated diffing algorithm.
Those are Sprint 24+ in your compiler. Don't block the demo on features
that don't exist yet.

In Phase 1, when the model changes, re-render the relevant DOM nodes directly.
This is correct and fast enough for demos. Add vdom in Phase 3 once the compiler
supports seq.

---

## Repository Structure

```
fsharp-wasm-runtime/
│
├── runtime/                    ← JS glue modules (pure JS, no build step)
│   ├── core.js                 ← handle store (shared by all modules)
│   ├── dom.js                  ← DOM operations
│   ├── events.js               ← event listener management
│   ├── net.js                  ← fetch + WebSocket
│   ├── storage.js              ← localStorage / sessionStorage
│   ├── canvas.js               ← Canvas 2D context
│   └── pubsub.js               ← in-process pub/sub (for demo messaging)
│
├── src/                        ← F# source (compiled via your pipeline)
│   ├── Runtime/
│   │   ├── Core.fs             ← Handle type + [<Import>] declarations
│   │   ├── Dom.fs              ← Element type + typed DOM wrappers
│   │   ├── Events.fs           ← Event subscription helpers
│   │   ├── Net.fs              ← Fetch bindings
│   │   └── Storage.fs          ← Storage bindings
│   ├── Elmish/
│   │   ├── Core.fs             ← Program type, Cmd, dispatch loop
│   │   └── Dom.fs              ← DOM rendering loop (mount/update)
│   └── Html/
│       ├── Elements.fs         ← el, div, button, input DSL
│       ├── Attrs.fs            ← attr, class_, id_, style_ DSL
│       └── Events.fs           ← onClick, onChange, onInput DSL
│
├── demos/                      ← Demo applications
│   ├── counter/                ← Hello World: Elmish counter
│   ├── todo/                   ← MVU todo list
│   ├── web-component/          ← Native Web Component demo
│   ├── canvas/                 ← Canvas drawing (shows compute power)
│   └── fetch/                  ← Async HTTP fetch demo (once async works)
│
├── index.html                  ← Main demo page (PicoCSS + all demos)
└── README.md
```

---

## Phase 1: Core Runtime (Week 1–2)

### JS Layer: `runtime/core.js`

This is the object store shared by all modules. Every JS object that needs
to be referenced from WASM gets a numeric handle.

```javascript
// runtime/core.js
const store = new Map();
let nextId = 1;

export const save  = obj => { const id = nextId++; store.set(id, obj); return id; };
export const get   = id  => store.get(id);
export const drop  = id  => store.delete(id);

// Pre-register well-known IDs
store.set(0, null);         // handle 0 = null/invalid
store.set(1, document.body); // handle 1 = document.body (always available)
```

### JS Layer: `runtime/dom.js`

```javascript
// runtime/dom.js
import { save, get } from "./core.js";

export const createElement = tag      => save(document.createElement(tag));
export const createText    = text     => save(document.createTextNode(text));
export const setText       = (id, t)  => { get(id).textContent = t; };
export const setHtml       = (id, h)  => { get(id).innerHTML = h; };
export const appendChild   = (p, c)   => { get(p).appendChild(get(c)); };
export const removeChild   = (p, c)   => { get(p).removeChild(get(c)); };
export const remove        = id       => { get(id).remove(); drop(id); };
export const setAttribute  = (id, k, v) => { get(id).setAttribute(k, v); };
export const removeAttribute = (id, k) => { get(id).removeAttribute(k); };
export const setClass      = (id, c)  => { get(id).className = c; };
export const setStyle      = (id, k, v) => { get(id).style[k] = v; };
export const querySelector = (id, sel) => {
    const el = get(id).querySelector(sel);
    return el ? save(el) : 0;
};
export const getById       = id       => {
    const el = document.getElementById(id);
    return el ? save(el) : 0;
};
export const getValue      = id       => get(id).value || "";
export const setValue      = (id, v)  => { get(id).value = v; };
export const focus         = id       => { get(id).focus(); };
export const scrollIntoView = id      => { get(id).scrollIntoView({behavior:"smooth"}); };
```

### JS Layer: `runtime/events.js`

```javascript
// runtime/events.js
import { get } from "./core.js";

// listeners: Map<elementId, Map<eventName, handlerFn>>
// Enables removeEventListener later
const listeners = new Map();

export const on = (id, event, cb) => {
    const el = get(id);
    if (!el) return;
    const handler = e => {
        e.preventDefault && event === "submit" && e.preventDefault();
        cb(id);
    };
    el.addEventListener(event, handler);
    if (!listeners.has(id)) listeners.set(id, new Map());
    listeners.get(id).set(event, handler);
};

export const off = (id, event) => {
    const el = get(id);
    if (!el || !listeners.has(id)) return;
    const handler = listeners.get(id).get(event);
    if (handler) el.removeEventListener(event, handler);
};

// Value-returning event: passes current input value as string
export const onInput = (id, cb) => {
    const el = get(id);
    if (!el) return;
    el.addEventListener("input", e => cb(e.target.value));
};
```

### F# Layer: `src/Runtime/Core.fs`

```fsharp
module FSharpWasm.Runtime.Core

/// Opaque handle to a JS object in the runtime object store.
/// The JS side manages the actual object; F# only sees the integer ID.
type Handle = int

/// The invalid/null handle.
let inline nil : Handle = 0

/// document.body — always available as handle 1.
let inline body : Handle = 1
```

### F# Layer: `src/Runtime/Dom.fs`

```fsharp
module FSharpWasm.Runtime.Dom

open Core

// ─── Raw imports ──────────────────────────────────────────────────────────────

[<Import("createElement",  "./dom.js")>] extern Handle _createElement(string tag)
[<Import("createText",     "./dom.js")>] extern Handle _createText(string text)
[<Import("setText",        "./dom.js")>] extern unit   _setText(Handle id, string text)
[<Import("appendChild",    "./dom.js")>] extern unit   _appendChild(Handle parent, Handle child)
[<Import("removeChild",    "./dom.js")>] extern unit   _removeChild(Handle parent, Handle child)
[<Import("remove",         "./dom.js")>] extern unit   _remove(Handle id)
[<Import("setAttribute",   "./dom.js")>] extern unit   _setAttribute(Handle id, string key, string value)
[<Import("setClass",       "./dom.js")>] extern unit   _setClass(Handle id, string cls)
[<Import("setStyle",       "./dom.js")>] extern unit   _setStyle(Handle id, string prop, string value)
[<Import("getById",        "./dom.js")>] extern Handle _getById(string id)
[<Import("getValue",       "./dom.js")>] extern string _getValue(Handle id)
[<Import("setValue",       "./dom.js")>] extern unit   _setValue(Handle id, string value)
[<Import("focus",          "./dom.js")>] extern unit   _focus(Handle id)

// ─── Typed Element wrapper ────────────────────────────────────────────────────

/// Typed wrapper around a DOM element handle.
type Element = private El of Handle with
    member e.Handle = let (El h) = e in h

module Element =
    let create tag       = El (_createElement tag)
    let text s           = El (_createText s)
    let ofId domId       = El (_getById domId)
    let nil              = El 0
    let isValid (El h)   = h <> 0

// ─── DOM operations (work on Element, not raw Handle) ────────────────────────

let append (El parent) (El child) = _appendChild parent child
let remove (El id)                = _remove id
let removeFrom (El parent) (El child) = _removeChild parent child

let setText  (El id) text         = _setText id text
let setAttr  (El id) key value    = _setAttribute id key value
let setClass (El id) cls          = _setClass id cls
let setStyle (El id) prop value   = _setStyle id prop value

let getValue (El id)              = _getValue id
let setValue (El id) value        = _setValue id value
let focus    (El id)              = _focus id

let body = El 1  // document.body, pre-registered in core.js
```

### F# Layer: `src/Html/Elements.fs`

This is the DSL. The goal is that F# UI code reads like HTML structure.

```fsharp
module FSharpWasm.Html.Elements

open FSharpWasm.Runtime.Dom
open FSharpWasm.Runtime.Events

/// A node in the virtual (actually immediate-mode) tree.
/// In Phase 1 this is purely imperative — no vdom.
type Node =
    | El    of tag: string * attrs: Attr list * children: Node list
    | Text  of string
    | Empty

and Attr =
    | Class of string
    | Id    of string
    | Style of string * string
    | Attr  of string * string
    | OnClick  of (unit -> unit)
    | OnInput  of (string -> unit)
    | OnChange of (string -> unit)

/// Render a Node tree into a parent Element.
/// Phase 1: no diffing. Clears parent and re-renders on each call.
let rec renderInto (parent: Element) (node: Node) : Element =
    match node with
    | Empty -> Element.nil

    | Text t ->
        let el = Element.text t
        append parent el
        el

    | El(tag, attrs, children) ->
        let el = Element.create tag
        attrs |> List.iter (applyAttr el)
        children |> List.iter (fun child -> renderInto el child |> ignore)
        append parent el
        el

and applyAttr (el: Element) = function
    | Class c        -> setClass el c
    | Id i           -> setAttr el "id" i
    | Style(p, v)    -> setStyle el p v
    | Attr(k, v)     -> setAttr el k v
    | OnClick f      -> onClick el (fun _ -> f ())
    | OnInput f      -> onInput el f
    | OnChange f     -> onChange el f

/// Clear a parent element and re-render children.
let clearAndRender (parent: Element) (nodes: Node list) =
    // Phase 1: brute force clear + re-render
    // TODO Phase 3: vdom diffing here
    _setHtml parent.Handle ""
    nodes |> List.iter (fun n -> renderInto parent n |> ignore)

// ─── HTML element constructors ─────────────────────────────────────────────────

let inline el tag attrs children = El(tag, attrs, children)
let div    = el "div"
let span   = el "span"
let p      = el "p"
let h1     = el "h1"
let h2     = el "h2"
let h3     = el "h3"
let button = el "button"
let input  = el "input"
let form   = el "form"
let nav    = el "nav"
let header = el "header"
let main_  = el "main"
let footer = el "footer"
let article = el "article"
let section = el "section"
let ul     = el "ul"
let ol     = el "ol"
let li     = el "li"
let a      = el "a"
let img attrs = El("img", attrs, [])
let text s = Text s

// ─── Attribute constructors ───────────────────────────────────────────────────

let class_    c     = Class c
let id_       i     = Id i
let style_    p v   = Style(p, v)
let attr_     k v   = Attr(k, v)
let onClick_  f     = OnClick f
let onInput_  f     = OnInput f
let onSubmit_ f     = OnClick f  // Form submit routes through onClick with preventDefault
let href      url   = Attr("href", url)
let src       url   = Attr("src", url)
let type_     t     = Attr("type", t)
let placeholder t   = Attr("placeholder", t)
let value_    v     = Attr("value", v)
let disabled_ b     = if b then Attr("disabled","") else Attr("","")
```

---

## Phase 1 Demo: Elmish Counter

This is the first thing that should compile and run.

### `src/Elmish/Core.fs`

```fsharp
module FSharpWasm.Elmish.Core

type Dispatch<'Msg> = 'Msg -> unit
type Cmd<'Msg> = (Dispatch<'Msg> -> unit) list

module Cmd =
    let none : Cmd<'Msg> = []
    let ofMsg msg        = [fun d -> d msg]
    let batch            = List.concat

type Program<'Model, 'Msg> = {
    Init   : unit -> 'Model * Cmd<'Msg>
    Update : 'Msg -> 'Model -> 'Model * Cmd<'Msg>
    View   : 'Model -> Dispatch<'Msg> -> FSharpWasm.Html.Elements.Node
}

/// Run an Elmish program inside a DOM element.
/// Phase 1: synchronous, no async commands.
let run (mountId: string) (program: Program<'Model,'Msg>) =
    open FSharpWasm.Runtime.Dom
    open FSharpWasm.Html.Elements

    let container = Element.ofId mountId
    let mutable model, initialCmds = program.Init()

    let rec dispatch msg =
        let newModel, cmds = program.Update msg model
        model <- newModel
        render ()
        cmds |> List.iter (fun cmd -> cmd dispatch)

    and render () =
        let view = program.View model dispatch
        clearAndRender container [view]

    render ()
    initialCmds |> List.iter (fun cmd -> cmd dispatch)
```

### `demos/counter/Counter.fs`

```fsharp
module Counter

open FSharpWasm.Elmish.Core
open FSharpWasm.Html.Elements

type Model = { Count: int }
type Msg = Increment | Decrement | Reset

let init () = { Count = 0 }, Cmd.none

let update msg model =
    match msg with
    | Increment -> { model with Count = model.Count + 1 }, Cmd.none
    | Decrement -> { model with Count = model.Count - 1 }, Cmd.none
    | Reset     -> { model with Count = 0 },              Cmd.none
    , Cmd.none

let view model dispatch =
    article [class_ "counter-demo"] [
        header [] [h2 [] [text "Counter"]]
        section [] [
            p [class_ "count-display"] [text (string model.Count)]
            div [class_ "button-group"] [
                button [onClick_ (fun () -> dispatch Decrement)] [text "−"]
                button [onClick_ (fun () -> dispatch Reset)]     [text "Reset"]
                button [onClick_ (fun () -> dispatch Increment)] [text "+"]
            ]
        ]
    ]

let start () =
    run "counter-root" { Init = init; Update = update; View = view }
```

---

## Phase 1 Demo: Todo List

```fsharp
module Todo

open FSharpWasm.Elmish.Core
open FSharpWasm.Html.Elements

type Todo = { Id: int; Text: string; Done: bool }
type Model = { Todos: Todo list; Input: string; NextId: int }
type Msg =
    | InputChanged of string
    | AddTodo
    | ToggleTodo of int
    | RemoveTodo of int

let init () =
    { Todos = []; Input = ""; NextId = 1 }, Cmd.none

let update msg model =
    match msg with
    | InputChanged s  -> { model with Input = s }, Cmd.none
    | AddTodo when model.Input.Trim() <> "" ->
        let todo = { Id = model.NextId; Text = model.Input.Trim(); Done = false }
        { model with Todos = model.Todos @ [todo]; Input = ""; NextId = model.NextId + 1 }, Cmd.none
    | AddTodo -> model, Cmd.none
    | ToggleTodo id ->
        let todos = model.Todos |> List.map (fun t -> if t.Id = id then { t with Done = not t.Done } else t)
        { model with Todos = todos }, Cmd.none
    | RemoveTodo id ->
        { model with Todos = model.Todos |> List.filter (fun t -> t.Id <> id) }, Cmd.none

let viewTodo dispatch (todo: Todo) =
    li [class_ (if todo.Done then "done" else "")] [
        span [onClick_ (fun () -> dispatch (ToggleTodo todo.Id))]
             [text (if todo.Done then "☑ " else "☐ "); text todo.Text]
        button [class_ "secondary"; onClick_ (fun () -> dispatch (RemoveTodo todo.Id))]
               [text "×"]
    ]

let view model dispatch =
    article [class_ "todo-demo"] [
        header [] [h2 [] [text "Todo List"]]
        section [] [
            div [class_ "input-group"] [
                input [type_ "text"; value_ model.Input; placeholder "Add a todo..."
                       onInput_ (fun s -> dispatch (InputChanged s))]
                button [onClick_ (fun () -> dispatch AddTodo)] [text "Add"]
            ]
            ul [] (model.Todos |> List.map (viewTodo dispatch))
            p [class_ "hint"] [
                text (sprintf "%d of %d completed"
                    (model.Todos |> List.filter (fun t -> t.Done) |> List.length)
                    (model.Todos |> List.length))
            ]
        ]
    ]
```

---

## Phase 2: Web Component Demo

This shows that F# compiled to WASM can define native browser custom elements.

```javascript
// runtime/components.js
export function defineComponent(name, mountFn) {
    customElements.define(name, class extends HTMLElement {
        connectedCallback() {
            mountFn(save(this));
        }
    });
}
```

```fsharp
// demos/web-component/MyCounter.fs
module MyCounter

[<Import("defineComponent", "./components.js")>]
extern unit _defineComponent(string name, (int -> unit) factory)

let register () =
    _defineComponent("fs-counter", fun handleId ->
        // Mount the Elmish counter into this element
        // (same Counter.fs, different mount point)
        Counter.runInto handleId)
```

Usage in HTML — zero JavaScript required by the page author:
```html
<fs-counter></fs-counter>
<fs-counter></fs-counter>
<fs-counter></fs-counter>
```

---

## Phase 2: Canvas Demo

Shows compute power — things WASM is actually good at.

```fsharp
// demos/canvas/Mandelbrot.fs
module Mandelbrot

[<Import("getContext2D", "./canvas.js")>] extern int _getCtx(int canvasId)
[<Import("fillRect",     "./canvas.js")>] extern unit _fillRect(int ctx, int x, int y, int w, int h)
[<Import("setFillStyle", "./canvas.js")>] extern unit _setFill(int ctx, string color)
[<Import("putImageData", "./canvas.js")>] extern unit _putImage(int ctx, int imgId, int x, int y)

let iterate (cx: float) (cy: float) (maxIter: int) : int =
    let mutable x = 0.0
    let mutable y = 0.0
    let mutable i = 0
    while i < maxIter && x*x + y*y < 4.0 do
        let xn = x*x - y*y + cx
        y <- 2.0*x*y + cy
        x <- xn
        i <- i + 1
    i

let render (canvasId: int) (width: int) (height: int) =
    let ctx = _getCtx canvasId
    for py in 0 .. height - 1 do
        for px in 0 .. width - 1 do
            let cx = float px / float width * 3.5 - 2.5
            let cy = float py / float height * 2.0 - 1.0
            let iter = iterate cx cy 100
            let c = 255 - iter * 255 / 100
            _setFill ctx $"rgb({c},{c},{c})"
            _fillRect ctx px py 1 1
```

This is important for the narrative: **pure F# computation running in WASM,
outputting to Canvas through a thin JS bridge**. No React. No framework.
The JS glue is 5 lines.

---

## Phase 3: What Requires Compiler Features

Do NOT attempt these until the compiler supports the features:

| Demo | Requires |
|------|---------|
| `fetch` HTTP | `async { }` (Sprint 25+) |
| WebSocket live updates | `async { }` (Sprint 25+) |
| `seq`-based list rendering | `seq { }` / IEnumerable (Sprint 24+) |
| localStorage reactive | Works now (synchronous) |
| vdom diffing | `seq`/`Map` + GC pressure testing |
| Worker threads | SharedArrayBuffer + WASM threads |

---

## `index.html` — The Demo Page

```html
<!DOCTYPE html>
<html lang="en" data-theme="light">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>F# → WebAssembly GC — Live Demo</title>
  <link rel="stylesheet"
        href="https://cdn.jsdelivr.net/npm/@picocss/pico@2/css/pico.min.css">
  <style>
    .demo-grid { display: grid; grid-template-columns: repeat(auto-fit, minmax(320px, 1fr)); gap: 2rem; }
    .count-display { font-size: 3rem; text-align: center; font-weight: bold; }
    .button-group { display: flex; gap: .5rem; justify-content: center; }
    .done { text-decoration: line-through; opacity: .6; }
    .done span { cursor: pointer; }
    canvas { width: 100%; border-radius: var(--pico-border-radius); }
    header nav { display: flex; justify-content: space-between; align-items: center; }
    .badge { font-size: .75rem; background: var(--pico-primary); color: white;
             padding: .2em .6em; border-radius: 1em; }
  </style>
</head>
<body>
  <header class="container">
    <nav>
      <strong>F# → WasmGC</strong>
      <span class="badge">Live Demo</span>
    </nav>
    <hgroup>
      <h1>F# compiled to WebAssembly GC</h1>
      <p>These demos run entirely in WASM. No JavaScript application code.</p>
    </hgroup>
  </header>

  <main class="container">
    <div class="demo-grid">
      <div id="counter-root"></div>
      <div id="todo-root"></div>
      <div id="canvas-root">
        <article>
          <header><h2>Mandelbrot (WASM compute)</h2></header>
          <canvas id="mandelbrot" width="400" height="300"></canvas>
          <footer><button id="render-btn">Render</button></footer>
        </article>
      </div>
    </div>

    <section>
      <h2>Web Components</h2>
      <p>These are native <code>&lt;custom-elements&gt;</code> defined from F#:</p>
      <div style="display:flex;gap:1rem;flex-wrap:wrap">
        <fs-counter></fs-counter>
        <fs-counter></fs-counter>
      </div>
    </section>
  </main>

  <footer class="container">
    <small>
      Source: <a href="https://github.com/fsharp-wasm/fsharp-wasm-runtime">fsharp-wasm-runtime</a>
      · Compiler: <a href="https://github.com/fsharp-wasm/fsharp-wasm">fsharp-wasm</a>
    </small>
  </footer>

  <script type="module">
    // Boot: instantiate the WASM module, then call F# entry points
    import { instantiate } from "./dist/runtime.js";

    const wasm = await instantiate("./dist/app.wasm");

    // F# entry points — these are exported functions from your compiled WASM
    wasm.exports.Counter_start();
    wasm.exports.Todo_start();
    wasm.exports.Mandelbrot_setup();
    wasm.exports.MyCounter_register(); // register <fs-counter>

    document.getElementById("render-btn")
      .addEventListener("click", () => wasm.exports.Mandelbrot_render());
  </script>
</body>
</html>
```

---

## Build System

No bundler needed for Phase 1. Use `wasm-tools` (already in your pipeline) plus
a simple ES module loader.

```bash
# Compile F# to WASM
dotnet fable src/App.fsproj --target wasmgc --output dist/

# Validate
wasm-tools validate dist/app.wasm

# Serve locally (any static server)
npx serve .
# or
python3 -m http.server 8080
```

The JS runtime modules are plain ES modules — no build step, no npm install
required for the runtime itself. The only npm dependency is the static server
for local dev.

---

## What Makes This Demo Compelling

The story for anyone watching:

1. **Counter** — "This is pure F# MVU. The entire application logic is WASM. The
   JS is only 30 lines of DOM glue."

2. **Todo** — "Pattern matching, list operations, discriminated unions. All compiled
   to WasmGC structs and functions."

3. **Mandelbrot** — "This is where WASM shines. Pure F# numeric computation,
   no JS in the hot loop."

4. **Web Components** — "F# defines a native browser custom element.
   Any HTML page can use `<fs-counter>` without knowing it's WASM."

5. **PicoCSS** — "One CSS link. No CSS framework knowledge needed. Semantic HTML
   gets styled automatically."

The narrative is: **F# → WASM → Browser. Real language. Real runtime. No JavaScript
in your application code.**

---

## Sprint Schedule

| Week | Deliverable |
|------|-------------|
| 1 | JS runtime modules (core, dom, events) + F# bindings compiling |
| 1 | Counter demo running in browser |
| 2 | Todo demo + Elmish Core |
| 2 | HTML DSL (Elements, Attrs) |
| 3 | Web Component demo (`<fs-counter>`) |
| 3 | Canvas + Mandelbrot demo |
| 3 | PicoCSS + index.html polished |
| 4 | README + GitHub Pages deployment |

After compiler Sprint 25+ (async):
| — | Fetch demo (HTTP GET → render results) |
| — | WebSocket live demo |
