# Learning Bun From 0 → 100

A tutorial-style, debug-driven roadmap for understanding every corner of the Bun
codebase, picking up Zig + C++ + JavaScriptCore along the way, and building
your own toy "mini-Bun" in parallel so the knowledge actually sticks.

> Personal study plan. Not a contribution doc — see `CONTRIBUTING.md` for that.

---

## How to use this plan

- **Spend 70% reading + debugging real Bun code, 30% writing your own clone.**
  Reading without rebuilding is forgettable. Rebuilding without reading turns
  into a toy that drifts from reality.
- Each **Phase** ends with two checkboxes:
  - **"I can explain"** — a concept you should be able to whiteboard.
  - **"I can build"** — a small artifact in your parallel repo.
- Pick a parallel-project language *different* from Zig for the first pass
  (Rust or Go are great) so you're forced to translate ideas instead of copying
  code. Switch to Zig in Phase 4+ once Bun's idioms feel familiar.
- Keep a `notes/` folder per phase: failed hypotheses, surprising syscalls,
  commit hashes you grokked. The notes are the deliverable; the runtime is the
  byproduct.

### Tooling you'll lean on constantly

| Tool | What it's for |
| --- | --- |
| `bun bd` | Build + run debug Bun (`./build/debug/bun-debug`). Never use `bun test` directly. |
| `lldb` / `gdb` | Step through the Zig + C++ binary. Pretty-printers live in `misctools/lldb/` and `misctools/gdb/`. |
| `BUN_DEBUG_<scope>=1` | Enable scoped debug logs (`Output.scoped(.<scope>, .visible)`). Grep `Output.scoped` to discover scopes. |
| `BUN_JSC_validateExceptionChecks=1 BUN_JSC_dumpSimulatedThrows=1` | Catch JSC exception-scope mistakes early. |
| `bun run zig:check-all` | Cross-platform Zig compile sanity. |
| `rr` (Linux) | Time-travel debugging — godlike for "what mutated this pointer?". |
| `perf` / `samply` / Instruments | Hot-path profiling. |
| Ghidra / IDA (optional, late) | When you want to confirm what a release build *actually* emits. |

---

## Phase 0 — Mental model & prerequisites (week 1)

**Goal:** Stop being scared of the repo. Know what each top-level folder is for
and why Bun chose Zig + JSC instead of Rust + V8.

1. Read, in order, top-to-bottom:
   - `README.md`
   - `CLAUDE.md` (the canonical map of where things live — do not skip)
   - `AGENTS.md`, `CONTRIBUTING.md`
   - `package.json` (note the `bd`, `build`, `build:release`, `zig:check-all`
     scripts — these *are* the dev workflow)
   - `build.zig` and `scripts/build.ts` headers (skim, don't memorize)
2. Watch / read background:
   - JavaScriptCore architecture overview (WebKit blog: "Speculation in
     JavaScriptCore", "Introducing Riptide", "FTL"). You don't need to
     understand the JIT yet — you need to understand **VM, Heap,
     JSGlobalObject, JSCell, Structure, Identifier**.
   - Andrew Kelley's "Practical Data Oriented Design" talk. This is the
     mindset Bun is written in.
   - One libuv tutorial (Bun uses libuv on Windows, custom event loops on
     POSIX, but the *concepts* — handles, requests, the loop, the thread pool
     — transfer cleanly to `src/event_loop/`).
3. Install language toolchains:
   - Zig — match the version pinned in `vendor/zig/` / `flake.nix`. Don't use
     a newer Zig; the language is unstable and Bun pins exactly.
   - A modern C++ toolchain (clang preferred — Bun uses C++20).
   - Whatever language you pick for the parallel project.
4. Do `zig learn` properly: ziglearn.org chapters 0–3, plus the official
   language reference for `comptime`, `error sets`, `defer`/`errdefer`,
   `allocator`-passing convention, `struct` packing, and `extern`/`@cImport`.

- [ ] **I can explain:** what JSC's heap is, what a `JSGlobalObject` represents,
      and why Zig's "explicit allocators" matter for a runtime.
- [ ] **I can build:** a Zig "hello world" that takes an allocator parameter,
      runs an event loop with one timer using libuv (or your language's
      equivalent), and exits cleanly with no leaks under
      `valgrind --leak-check=full`.

---

## Phase 1 — Build it, run it, break it (week 2)

**Goal:** A working debug build, a working debugger session, and the muscle
memory to round-trip a code change.

1. Clone, then run the bootstrap script for your OS (`scripts/bootstrap.sh`
   or `scripts/bootstrap.ps1`). Read it as you go — it documents every system
   dep Bun needs.
2. `bun bd` — first build is slow, subsequent are incremental. While it
   compiles, read `scripts/build.ts` so you understand the arg routing
   (`--asan=off`, `--build-dir`, build-then-exec trailing args).
3. Run the debug binary three ways:
   - `bun bd -e 'console.log(Bun.version)'`
   - `bun bd test test/js/bun/util/version.test.ts` (or any small file)
   - `lldb -- ./build/debug/bun-debug -e 'throw new Error("hi")'` — set a
     breakpoint on `JSC::Exception::create`, hit it, inspect the stack.
4. Make a deliberately broken change in `src/cli.zig` (e.g. flip an `if`),
   rebuild, watch the test fail, revert. You now have the inner loop.
5. Read `src/codegen/` README-by-grep:
   - `generate-classes.ts` — turns `*.classes.ts` into Zig + C++ glue.
   - `bundle-modules.ts`, `bundle-functions.ts` — embed `src/js/` into the
     binary.
   - Modify a string in `src/js/bun/<something>.ts`, run `bun run build`
     (no Zig rebuild needed!), see the change.

- [ ] **I can explain:** what a "build-then-exec" command does, how `src/js/`
      gets into the binary, and which kinds of changes need a full `bd`
      rebuild vs. just a JS bundle rebuild.
- [ ] **I can build:** a one-line CLI in your parallel-project language that
      links to a small C library via FFI and prints its version — proving you
      can do "native binary that talks to native deps".

---

## Phase 2 — Memory, allocators, and the Bun "stdlib" (week 3)

**Goal:** Internalize how Bun manages memory, strings, and small data
structures. This is the layer everything else sits on.

Read in this order, taking notes:

1. `src/allocators/` — especially `MimallocArena.zig`, the bump allocators,
   and the "throwaway arena per request" pattern.
2. `src/string.zig`, `src/string_immutable.zig`, `src/bun.zig` (the prelude).
   Pay attention to `String`, `WTF::String` interop, and how Bun avoids
   allocating for short ASCII.
3. `src/collections/` and any `*.zig` containing `BabyList`, `BoundedArray`,
   `StringHashMap`, `IndexedSet`. Note the data-oriented bias: arrays of
   indices, not arrays of pointers.
4. `vendor/mimalloc/` — read the README, skim `mimalloc.h`. You don't need the
   internals, just the mental model of "thread-local segments + size classes".
5. `src/jsc/bindings/headers.h` and `src/jsc/bindings/ZigGlobalObject.cpp`
   to see the C++/Zig boundary.

Hands-on:

- In `lldb`, break on `bun.default_allocator.alloc` (or whatever the current
  symbol is) and watch a `bun -e '({a:1})'` execution. Where does the
  allocator come from? Who frees it?
- Write a Zig program in your notes that allocates 1M short strings two ways:
  one with `std.heap.page_allocator`, one with an arena. Time them. Feel why
  Bun is arena-happy.

- [ ] **I can explain:** when Bun uses an arena vs. mimalloc vs. the JSC heap,
      and what owns what across a `Bun.serve` request lifecycle.
- [ ] **I can build:** a small string library in your parallel language with
      inline-short-string optimization and a bump allocator.

---

## Phase 3 — The JS engine boundary (weeks 4–5)

**Goal:** Be able to add a new built-in function or class to Bun.

1. Read the skill `.claude/skills/implementing-jsc-classes-cpp/SKILL.md` and
   `.claude/skills/implementing-jsc-classes-zig/SKILL.md`. These are the
   official cheat-sheets for the boundary.
2. Read `.claude/skills/javascriptcore-garbage-collector/SKILL.md` end to end.
   Misunderstanding GC is the #1 source of nasty Bun bugs — `WriteBarrier`,
   `visitChildren`, `hasPendingActivity`, `addOpaqueRoot`, `IsoSubspace` are
   non-negotiable vocabulary.
3. Walk a single class top-to-bottom. Good first targets, in increasing size:
   - `Bun.MD5` / `Bun.SHA*` (`src/runtime/api/crypto.zig`) — small, no async.
   - `Bun.file()` → `Blob` (`src/runtime/webcore/Blob.zig`).
   - `Response` (`src/runtime/webcore/Response.zig`) — touches headers,
     body streams.
4. Trace one call end-to-end with `lldb`:
   - JS call → `*.classes.ts`-generated thunk → Zig `call` impl → result back.
   - Set breakpoints in both Zig and C++ to feel the boundary.
5. Add your own built-in: `Bun.greet(name: string): string`. Wire it through
   `src/bun.js/api/BunObject.classes.ts` (or wherever the current `Bun`
   namespace classfile lives), add a Zig impl, write a test in
   `test/js/bun/util/`, get it green under `bun bd test`.

- [ ] **I can explain:** what `*.classes.ts` generates, the difference between
      `JSDestructibleObject` and `JSNonFinalObject`, and why every C++ method
      that may throw needs an exception scope.
- [ ] **I can build:** in your parallel project, embed a JS engine
      (QuickJS is the easy mode pick; V8 if you want pain) and expose one
      native function with one async function. You don't need a class system
      yet — that comes in Phase 7.

---

## Phase 4 — Lex, parse, transpile (weeks 6–7)

**Goal:** Understand how Bun reads JS/TS/JSX faster than anyone else.

1. `src/js_lexer.zig` — note the SIMD-ish hot paths and the
   `comptime`-generated keyword tables. Read top-to-bottom.
2. `src/js_parser.zig` — this file is enormous; don't try to read linearly.
   Strategy:
   - Start at `parseStmt` and follow `parseExpr` recursively for a tiny
     program (`let x = 1;`).
   - Diff against esbuild's parser — Bun's is a fork with heavy mods, and
     reading Evan Wallace's esbuild blog posts ("Why is esbuild fast?") fills
     in the *why*.
3. `src/js_printer.zig` — the printer is where source-map mappings get
   emitted; understand the `Linker`/`Printer` split.
4. `src/transpiler.zig` — the public-facing wrapper. This is what
   `Bun.Transpiler` (JS-side) ultimately calls.
5. `src/resolver/` — module resolution. Read `resolver.zig`,
   `tsconfig_json.zig`, `package_json.zig`. Pay attention to how
   `node_modules` walking is cached.

Hands-on:

- Modify `src/js_parser.zig` to print every parsed function's name to stderr
  behind a debug scope. Run on a real project. Get a feel for parse counts.
- Write a tiny TS-subset parser in your parallel language: handle `const`,
  `function`, arrow functions, JSX. You don't need types — just a working AST.

- [ ] **I can explain:** why a recursive-descent parser with a hand-rolled
      lexer beats parser generators here, and what Bun's AST looks like in
      memory (it's not what you think — look at `Stmt`/`Expr` representations).
- [ ] **I can build:** a TS-subset → JS transpiler in your parallel language
      that handles JSX and produces working source maps for at least one
      mapping per statement.

---

## Phase 5 — The module loader & resolver in depth (week 8)

**Goal:** Know exactly what happens between `import x from "./y"` and
`y.js` executing.

1. `src/resolver/` (revisit) + `src/install/resolution.zig` — npm package
   resolution.
2. `src/bun.js/module_loader.zig` — the JS-side module loader. Watch how it
   interplays with JSC's own module pipeline.
3. CommonJS interop: search for `commonjs_export_names`, `cjs_module_lexer`.
   Bun's CJS-detection-without-eval is a quiet superpower; understand it.
4. ESM cache, HMR boundaries (`src/bake/`), `with { type: "..." }` import
   attributes.

Hands-on:

- Add `BUN_DEBUG_ModuleLoader=1` (find the actual scope via
  `grep -R 'Output.scoped(.module' src/`) and run a multi-file project. Read
  every line of output until it makes sense.
- In your parallel project, implement Node-compat resolution: `exports`
  conditions, `imports`, the scoped-package walk, the
  `directory + index.js` fallback. Test against the official
  [`resolve` test fixtures](https://github.com/lukeed/resolve.exports).

- [ ] **I can explain:** what "conditions" are, how Bun decides between
      `node` and `bun` exports, and why CJS lazy-binding (`module.exports = ...`
      after `require`) makes static analysis hard.
- [ ] **I can build:** a Node/Bun-compatible resolver that passes
      ≥80% of `resolve.exports` fixtures.

---

## Phase 6 — The bundler (weeks 9–10)

**Goal:** From AST to a tree-shaken, minified, code-split bundle.

1. `src/bundler/` — the entry is `bundle_v2.zig`. Read it like a book.
2. Tree-shaking: search for `IsLive`, `Symbol`, `ref`, `link`. Bun does
   esbuild-style symbol-graph linking. Read the relevant esbuild posts again
   in parallel.
3. CSS: `src/css/`. Skim — you don't need CSS in v1 of your clone.
4. HTML entrypoints: `src/bundler/bundle_html.zig` — recent and approachable.
5. Sourcemaps: `src/sourcemap/`. Understand VLQ encoding and the
   `mappings` segment layout.

Hands-on:

- Run `bun bd build ./test/snapshots/<small-fixture> --outdir=/tmp/out` under
  `lldb`, breakpoint on the linker entry, and step through one file.
- Read `test/bundler/` — every `itBundled` is a worked example of "input,
  options, expected output". Pick three and explain them in your notes.
- In your parallel project, implement a single-file bundler: parse → resolve →
  emit one IIFE bundle. Skip tree shaking; do it next iteration.

- [ ] **I can explain:** the difference between Bun's bundler IR and its
      runtime AST, what "splitting" buys you, and where source-map fidelity
      comes from.
- [ ] **I can build:** a multi-file bundler with basic tree-shaking
      (mark-and-sweep over a symbol graph) and source maps that load in
      Chrome DevTools.

---

## Phase 7 — Networking: HTTP server & client (weeks 11–12)

**Goal:** Understand how `Bun.serve` is the fastest JS HTTP server.

1. `src/runtime/api/server.zig` — the public API. Trace one request from
   `accept` to `respond`.
2. `packages/bun-usockets/` — Bun's fork of µSockets. C, fast, scary.
   Read `src/socket.c`, `src/eventing/epoll_kqueue.c`, `src/loop.c`.
3. `packages/bun-uws/` — fork of µWebSockets. The HTTP & WS protocol layer.
4. `src/http/` (client side), including `websocket_client/`.
5. `src/runtime/webcore/streams.zig`, `src/runtime/webcore/fetch.zig` — the
   bridge from sockets to `Response`/`ReadableStream`.

Hands-on:

- Run `bun bd -e 'Bun.serve({ port: 0, fetch: () => new Response("ok") })'`
  under `strace -f`. Count the syscalls per request. Compare to Node.
- In your parallel project: write an `epoll`/`kqueue` echo server in raw C
  or your chosen language's `mio`/`tokio` equivalent. Then layer HTTP/1.1
  parsing using `picohttpparser` (vendored at `vendor/picohttpparser/`).
- Add a `serve()` API on top of your JS-engine embedding from Phase 3.

- [ ] **I can explain:** how Bun avoids per-request allocations, what
      `corked` writes are, and why µWebSockets' state-machine HTTP parser is
      faster than a callback-per-header parser.
- [ ] **I can build:** a JS-callable `serve()` that does ≥100k req/s on
      `wrk` for a static `Response`. (You will not match Bun. That's fine.)

---

## Phase 8 — The package manager (weeks 13–14)

**Goal:** Be able to explain `bun install` from CLI flag to lockfile.

1. `src/install/` — entrypoint is `install.zig`. Map the modules:
   - `npm.zig` — registry client, manifest cache.
   - `lockfile/` — the binary lockfile format. Read the comment header at the
     top; it documents the on-disk layout.
   - `lifecycle_script_runner.zig` — `preinstall`/`postinstall` etc.
   - `resolution.zig` (revisit) + `dependency.zig`.
2. The `bun.lockb` (binary) → `bun.lock` (text) migration history is in git
   log. Read it; it explains a lot of design decisions.
3. Workspaces, overrides, peerDependency handling — search in `install.zig`.

Hands-on:

- `BUN_DEBUG_PackageManager=1 bun bd install` on a small repo. Then on a big
  one (Next.js example). Diff the call patterns.
- In your parallel project: write `install` that handles a single dependency
  with no transitive deps. Then add the manifest cache. Then add the
  resolver. Stop before lockfile — that's a multi-week side quest of its own.

- [ ] **I can explain:** Bun's content-addressable cache layout, why the
      lockfile is binary-then-text, and how the install graph parallelism is
      structured.
- [ ] **I can build:** `mini-install ./mypkg@1.2.3` that downloads the tarball,
      verifies the integrity hash, extracts it, and links it into
      `node_modules`.

---

## Phase 9 — Test runner, shell, FFI, SQL, runtime extras (weeks 15–16)

Pick the ones that interest you; do at least two deeply.

- `src/test/` — the Jest-compatible runner. Understand the snapshot pipeline,
  parallel file scheduling, and how `bun:test` is wired as a built-in module.
- `src/shell/` — cross-platform shell. The lexer/parser/AST/interpreter split
  here is a beautiful, smaller-scale mirror of the JS pipeline. Highly
  recommended as a "second pass" exercise.
- `src/runtime/api/FFI.zig` + `vendor/tinycc/` — TCC-as-a-JIT. Niche but
  mind-expanding.
- `src/sql/` — Postgres/MySQL/SQLite native clients. Real-world binary
  protocol parsers in Zig.
- `src/bake/` — server-side framework. The newest area; lots of churn.

- [ ] **I can explain:** at least two of the above subsystems well enough to
      diagram on a whiteboard from memory.
- [ ] **I can build:** a `bun:test`-style runner around your own JS
      embedding — `describe`, `test`, `expect`, snapshot, `--watch`.

---

## Phase 10 — Performance, profiling, and the long tail (weeks 17+)

**Goal:** Stop reading; start measuring.

1. `bench/` — every folder is a microbench worth understanding. Run them on
   `bun bd` (debug, slow) vs `bun run build:release` (real numbers).
2. Linux: `perf record -g`, then `perf report` on a release Bun running a
   workload. Find a hot frame. Read the surrounding code. Understand why.
3. JSC profiling: `--jsc-options=reportCompileTimes=1`,
   `dumpDFGGraph=1`. Massive output; learn to filter.
4. Read the closed bug PRs labelled `perf` over the last 6 months. They are
   the best free seminar in real-world systems perf.

- [ ] **I can explain:** one perf optimization in Bun's history at the level
      of "before this commit, X was Y ns; after, X was Z ns; the reason is W".
- [ ] **I can build:** a benchmark harness for your clone that produces
      stable numbers across runs (warm-up, GC control, statistical
      significance, not just "I ran it once").

---

## Parallel-project milestones (your "mini-Bun")

Track these alongside the phases above. Each milestone is a tag in your repo.

| Tag | Capabilities |
| --- | --- |
| `v0.1-hello` | CLI binary, FFI to one C lib, embedded JS engine that runs `console.log`. |
| `v0.2-strings` | Custom string + arena allocator. |
| `v0.3-class` | Add one native class (`MD5`) to the JS global. |
| `v0.4-tspile` | TS+JSX → JS transpiler with source maps. |
| `v0.5-resolve` | Node/Bun-compatible resolver, ≥80% on resolve.exports. |
| `v0.6-bundle` | Multi-file bundler with tree-shaking. |
| `v0.7-serve` | `serve()` doing ≥100k req/s on `wrk`. |
| `v0.8-install` | `install pkg@version` that links into `node_modules`. |
| `v0.9-test`   | `mini-test` runner with `expect` + snapshots. |
| `v1.0`        | A *short* blog post explaining what's still slow vs Bun and why. The blog post is the proof you understood. |

---

## Reading-list shortlist (no fluff)

- **Zig**: Loris Cro's blog (especially "What is Zig's Comptime?"); Andrew
  Kelley's talks; the Zig std lib source.
- **JSC / V8**: WebKit blog "Speculation in JavaScriptCore"; v8.dev
  "Understanding ECMAScript spec" series; "Crankshaft" / "TurboFan" /
  "Sparkplug" papers.
- **Esbuild**: Evan Wallace's "esbuild internals" blog series. Bun's bundler
  is a descendant.
- **µWebSockets / µSockets**: source comments + Alex Hultman's GitHub
  discussions. Sparse but golden.
- **Systems perf**: Brendan Gregg, *Systems Performance*, ch. 6, 9, 10.
- **Memory allocators**: mimalloc paper (Leijen et al., 2019).

---

## Anti-patterns to avoid

- **Reading top-to-bottom in `src/`.** Bun is too big. Always start from a
  user-visible behavior (`bun -e 'X'`) and follow it down.
- **"I'll come back to GC later."** No, you won't. Phase 3 GC reading is
  non-negotiable; everything after assumes it.
- **Cargo-culting Zig style.** Bun's Zig is *idiosyncratic* — heavy
  `comptime`, lots of unions, hand-rolled tables. Learn vanilla idiomatic
  Zig first (ziglearn + std lib), then read Bun's deviations as deliberate
  choices.
- **Skipping tests in your clone.** If you didn't test it, it doesn't work.
  Every milestone above ships with tests or it doesn't ship.
- **Comparing your clone's perf to release Bun on day one.** You are
  competing with years of the most aggressive runtime perf work in the
  ecosystem. Compare `your-clone vs your-clone-yesterday`.

---

## Weekly cadence template

```
Mon  90m read source for the week's phase
Tue  90m debugger session: trace one real call end-to-end
Wed  60m write notes (what surprised me, what I still don't get)
Thu  120m parallel-project work
Fri  60m write a test (in Bun OR your clone) that exercises today's concept
Sat  free / catch-up / blog draft
Sun  rest
```

Roughly 8 focused hours/week → ~17 weeks → end of Phase 10. Adjust to taste;
the order matters more than the speed.

---

*Last updated: drafted on `zhichli/learn`. Living document — edit as
understanding deepens.*
