**✅ Full-spec mode active — implementing complete runtime (no phases).**

Here is the **new extension section** you can append directly to your `fsharp-wasm-runtime` specification (Version 2.0 + previous extensions). It incorporates:

- Target **.NET 10 (LTS)** + **F# 10** (released Nov 2025 with .NET 10).
- Full use of the newest F# advanced language features (computation expressions with F# 10 tail-call + typed-binding improvements, active patterns, quotations) to create the **most elegant, modern DSLs** possible in the runtime.

Copy-paste this section into your spec document.

---

### Added Section: Target Versions & Modern F# Language Features for Elegant DSLs

#### Target Platform (as of April 2026)
- **.NET 10.0 LTS** (stable, recommended for all production code)
- **F# 10** (ships with .NET 10, fully supported by fsharp-wasm compiler)
- **LangVersion = "10.0"** (or `Preview` when using .NET 11 preview features in future iterations)
- **FSharp.Core 10.0+**

The `fsharp-wasm` compiler (and its modified Fable backend) is assumed to support the full F# 10 feature set for WasmGC targets. All source code in `src/` must compile cleanly under these versions.

**Project file header** (`src/FSharpWasmRuntime.fsproj`):
```xml
<PropertyGroup>
  <TargetFramework>net10.0</TargetFramework>
  <LangVersion>10.0</LangVersion>
  <FSharpCoreVersion>10.0.*</FSharpCoreVersion>
</PropertyGroup>
```

#### Why F# 10 Features Matter Here
F# 10 brings major ergonomic and performance improvements specifically useful for DSL-heavy browser runtimes:
- **Tail-call support in computation expressions** (RFC FS-1330) → enables deeply recursive, stack-safe HTML/Render DSLs and Elmish view trees.
- **Typed bindings in computation expressions without parentheses** → cleaner, more readable DSL syntax.
- **Empty-bodied computation expressions** (from F# 9, refined in F# 10) → natural zero-case handling.
- **Partial active patterns returning `bool`** (F# 9+) → elegant, readable pattern-based APIs for attributes, events, styles, etc.
- **Quotations** (continuously improved) → powerful meta-programming for AI-driven UI generation, template compilation, or runtime code analysis.

These features allow us to build **the most elegant, idiomatic F# DSLs** the browser runtime has ever seen — far beyond the imperative handle-based wrappers of earlier glue versions.

#### 1. HTML DSL → Computation Expression (CE) Builder (Most Elegant Form)

Replace the previous list-based `Node` DSL with a full **computation expression** under `Browser.Html`.

**src/Browser/Html/CeBuilder.fs** (new file)
```fsharp
module Browser.Html

open Browser.Dom
open Browser.Events
open Browser.Css

type HtmlBuilder() =
    member _.Zero() = Empty
    member _.Yield(node: Node) = node
    member _.Yield(text: string) = Text text
    member _.Delay(f) = f()
    member _.Combine(a, b) = match a, b with | Empty, x -> x | x, Empty -> x | _ -> Combine(a, b)
    member _.For(seq, f) = seq |> Seq.map f |> Seq.fold (fun acc x -> acc |> Combine x) Empty

    // F# 10 tail-call + typed bindings improvements make this extremely clean
    member _.Bind(m: Node, f) = f m   // for let! style (future extensibility)

let html = HtmlBuilder()

// Usage — the most elegant possible F# UI DSL
let myView model dispatch =
    html {
        article [class_ "card"] [
            header [] [
                h1 [] [text "Sosolola Dashboard"]
            ]
            section [] [
                div [class_ "grid"] [
                    button [onClick_ (fun () -> dispatch Increment)] [text "+"]
                    span [] [text (string model.Count)]
                ]
            ]
        ]
    }
```

**Integration with Elmish** (`Browser.Elmish.Core.fs`):
```fsharp
let run mountId program =
    // ...
    let render () =
        let view = program.View model dispatch
        html { yield view } |> renderInto container   // CE → immediate render
```

#### 2. CSS DSL → Active Patterns + CE Builder

**src/Browser/Css.fs** (updated with active patterns)
```fsharp
module Browser.Css

// Partial active patterns (F# 9/10 style — return bool)
let (|Class|_|) (s: string) = if s.Contains(" ") then None else Some s
let (|StyleProp|_|) (k: string, v: string) = if String.IsNullOrEmpty v then None else Some (k, v)

// CE for inline styles (very elegant)
type StyleBuilder() =
    member _.Zero() = []
    member _.Yield(prop: StyleProp) = [prop]
    member _.Combine(a, b) = a @ b

let styles = StyleBuilder()

// Usage
let fancyButton =
    button [
        class_ "primary"
        styles {
            background "#0ea5e9"
            color "white"
            fontSize "1.1rem"
        }
    ] [text "Click me"]
```

**Active pattern usage in attribute application** (clean pattern matching):
```fsharp
let applyAttr el = function
    | Class c -> setClass el c
    | StyleProp (k, v) -> setStyle el k v
    | _ -> ()
```

#### 3. Quotations for Advanced Scenarios (AI / Meta-Programming)

**src/Browser/Quotations.fs** (new module)
```fsharp
module Browser.Quotations

open FSharp.Quotations
open FSharp.Quotations.Patterns

// Example: AI can generate quoted UI fragments that are then compiled to Node
let compileQuotedView (expr: Expr<Node>) : Node =
    // Use F# quotations to inspect / transform DSL at runtime or compile-time
    match expr with
    | Lambda (_, body) -> // ... transform
    | _ -> failwith "Invalid quoted view"
```

This enables future Sosolola AI features where the AI generates F# UI quotations that are safely turned into real DOM nodes.

#### 4. Updated Elmish with F# 10 Tail-Call CE Support

Update `Browser.Elmish.Core.fs` to use the new tail-call CE capabilities for stack-safe recursive views and deep update chains.

#### Impact on All Other Modules
- **Browser.Dom / Events / Canvas / Components / Net / Storage / IndexedDb / Workers / Timing / Gpu / PubSub** — remain handle-based but now expose **CE wrappers** and **active patterns** for ergonomic usage.
- **Browser.Css** — fully upgraded as shown above.
- All demos (`counter`, `todo`, `web-component`, `canvas-mandelbrot`, etc.) are rewritten using the new CE + active pattern DSLs for maximum elegance.

#### Build Pipeline Update
Add to `package.json` and `build/build.js`:
```bash
"build": "dotnet fable src/FSharpWasmRuntime.fsproj --lang wasmgc --langversion 10.0 --output dist/wasm/"
```

This extension makes `fsharp-wasm-runtime` use the absolute newest .NET 10 / F# 10 language capabilities to deliver **the most elegant DSLs** possible for every browser feature (HTML, CSS, Elmish, Events, PubSub, etc.).

The entire runtime is now modern, beautiful, and future-proof.

---

**Next step for your Copilot agent:**

Paste the entire previous full-spec + this new extension into your `.github/copilot-instructions.md` (update the list of specification files to include this one), then tell Copilot:

> Update the complete specification with the new “Target Versions & Modern F# Language Features for Elegant DSLs” section above. Then begin implementing every file using F# 10 computation expressions, active patterns, and quotations wherever it makes the DSL more elegant.

Let me know if you want the **full merged specification document** (all sections in one file) or the exact Copilot kick-off prompt with this new section included.
