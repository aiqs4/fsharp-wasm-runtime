Let me think through this carefully before writing anything. Let me check what exists in the Fable ecosystem first.Now I have a complete, honest picture. Let me write the real spec.Die wichtigsten Antworten zuerst, dann kannst du die Spec lesen:

**Ja, separates Repo.** `fsharp-wasm-runtime` ist komplett unabhängig vom Compiler. Der Compiler ist das Werkzeug, das Runtime ist was man damit baut. Saubere Trennung.

**WASM kann nicht direkt auf DOM zugreifen — das wird sich nicht ändern.** Das ist kein Bug deines Compilers, das ist ein fundamentales Property des WASM-Standards. Die JS-Glue-Schicht ist nicht das Problem — sie ist die richtige Architektur. Dein Job ist, sie dünn und unsichtbar zu machen.

**Feliz/DaisyUI importieren: Nein.** Feliz braucht React. React ist kein WASM. Du brauchst deine eigene DOM-DSL, die durch deinen Compiler compiliert. PicoCSS ist ein `<link>`-Tag — da brauchst du keine F# Bindings. Fertig.

**Elmish: Schreib es selbst.** `Fable.Elmish` zielt auf JavaScript. Dein Elmish-Core sind ~150 Zeilen F# die durch deinen eigenen Compiler laufen. Das ist der Beweis: dein Compiler compiliert Elmish selbst.

**Virtual DOM: Nicht in Phase 1.** Vdom braucht seq/IEnumerable (Sprint 24+). Phase 1 macht clearAndRender — brute-force clear + re-render. Für Demos ist das völlig ausreichend und 10x einfacher.

**Der echte Demo-Narrative:** Counter + Todo (MVU in WASM), Mandelbrot (Compute-Power in WASM), Web Components (F# definiert native Custom Elements). Das ist überzeugend. Kein Backend nötig, kein Async nötig, alles läuft heute mit deinem Sprint-21-Compiler.
