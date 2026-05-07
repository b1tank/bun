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
- **Use Rust as your parallel-project language.** Bun already has an
  in-progress Zig→Rust port on the `claude/phase-a-port` branch with
  battle-tested translation docs. You get an authoritative cheat-sheet for
  every construct you'll meet — see
  ["Zig → Rust port reference"](#zig--rust-port-reference) below. (Go works
  too if you must, but you lose the docs leverage.)
- **Prereqs are deliberately small**: basic Rust (you can write `cargo new`,
  `Result<T, E>`, `Box<T>`, a `match`) and basic JS (you've called
  `console.log`). Everything else — Zig, JSC, libuv, mimalloc — you pick up
  *as you need it*, not upfront. The plan starts with a hello-world milestone
  on Day 1, before any reading.
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

## Zig → Rust port reference

If your parallel language is **Rust**, do not invent translation rules. The
`claude/phase-a-port` branch contains the canonical Zig→Rust playbook used to
port Bun itself. Read these once at the start, then revisit per phase:

```bash
git show claude/phase-a-port:docs/PORTING.md                    > PORTING.md
git show claude/phase-a-port:docs/rust-rewrite-plan.md          > rust-rewrite-plan.md
git show claude/phase-a-port:docs/.rust-rewrite-verified-claims.md > rust-rewrite-verified-claims.md
git show claude/phase-a-port:docs/CYCLEBREAK.md                 > CYCLEBREAK.md
git show claude/phase-a-port:docs/LIFETIMES.tsv                 > LIFETIMES.tsv
git show claude/phase-a-port:docs/rust-migration-tree.md        > rust-migration-tree.md
```

| Doc | What you get | When to read it |
| --- | --- | --- |
| `PORTING.md` | Per-construct Zig→Rust idiom map: types, errors, `comptime`, `defer`/`errdefer`, allocators, strings, collections, dispatch, pointers, concurrency. | Phases 2–6 — you'll consult it constantly. |
| `rust-rewrite-plan.md` | Architecture: why Rust, JSC GC model, codegen contract, calling conventions, six GC liveness primitives, `bun_sys`, allocators. | Phase 0 (skim §1–2), Phase 3 (§2–6 deep), Phase 7 (§7). |
| `.rust-rewrite-verified-claims.md` | Adversarially-verified facts (each survived 3-vote refutation against `file:line`). The source of truth behind the plan. | Use as a reference — search by symbol when something in the plan looks surprising. |
| `CYCLEBREAK.md` | The 89-crate dependency graph and how it was untangled — tier ordering, `MOVE_DOWN`/`FORWARD_DECL`/`TYPE_ONLY` classification, hot dispatch list, debug-hook registration pattern. | Phase 5–6 — when you start splitting your clone into crates. |
| `LIFETIMES.tsv` | Pre-computed per-field lifetime classification (`OWNED`/`SHARED`/`BORROW_PARAM`/`STATIC`/`JSC_BORROW`/`BACKREF`/`INTRUSIVE`/`FFI`/`ARENA`/`UNKNOWN`) for every Zig pointer field. Trust it over local guessing. | Phase 2–3 when typing struct fields. |
| `rust-migration-tree.md` | Crate dependency tree (`bun_alloc` → `bun_core` → ...) showing the bottom-up port order. | Phase 5–6 for crate split. |

**Live ports.** The branch also has paired `.zig`/`.rs` files (e.g.
`src/jsc/RuntimeTranspilerStore.{zig,rs}`,
`src/bundler/HTMLImportManifest.{zig,rs}`,
`src/bundler/linker_context/scanImportsAndExports.{zig,rs}`). **Diff these
pairs side-by-side**: they are worked examples of every rule in `PORTING.md`
applied to real Bun code. `git log claude/phase-a-port -- '*.rs'` will surface
more as the port progresses.

**The 5 rules to internalize first** (paraphrased from `PORTING.md` ground rules):

1. **No `tokio`/`async fn`/`std::fs`/`std::net`/`std::process`.** Bun owns its
   event loop and syscalls; the port wraps them in `bun_sys`/`bun_aio`. Async
   stays callbacks + state machines.
2. **Errors are a `Copy` `NonZeroU16` newtype (`bun_core::Error`), never
   `anyhow`/`Box<dyn Error>`.** This preserves `@errorName` snapshot
   compatibility, fits into `#[repr(C)]` payloads, and matches Zig's payload-free errors.
3. **Bytes are `&[u8]`/`Vec<u8>`, not `&str`/`String`.** Paths, source code,
   HTTP, env vars, module specifiers — all WTF-8/arbitrary bytes. Inserting
   UTF-8 validation is a perf tax *and* a correctness bug.
4. **`defer x.deinit()` → delete the line; let `Drop` do it.** `errdefer` on a
   local you just allocated → also delete it; `?` already drops the `Vec`/`Box`
   on the error path. Keep `errdefer` only for side effects (refcount
   rollback, map unregistration) — use `scopeguard` then.
5. **AST/parser crates keep arenas (`bumpalo`/`typed-arena`); everything else
   uses the global mimalloc.** Don't thread `&mut Allocator` through your
   whole clone — `Box`/`Vec` already hit mimalloc via `#[global_allocator]`.

---

## Day 0 — Set up your dev environment (1–2 hours, before everything)

**Goal:** A clean `~/banli/` workspace with every tool installed, Bun's
source cloned and bootstrapped, and a placeholder Rust crate where you'll
write your own runtime. By the end, every later step in this doc Just Works
without "oh I need to install X first."

Doing this from scratch on a clean Linux/macOS box, allow ~1–2 hours — most
of it is the initial Bun build downloading and compiling C++ deps.

### Folder layout (the "banli" workspace)

Everything you write lives in `~/banli/`. The cloned Bun checkout is your
read-only reference; your code is the sibling crate also called `banli`.

```
~/banli/
├── bun/             ← cloned upstream Bun (reference; you read it, you don't fork it)
├── banli/           ← YOUR Rust crate (the parallel mini-Bun you build through phases)
├── notes/           ← per-phase markdown notes (notes/phase-0.md, phase-1.md, ...)
└── README.md        ← one-pager: "this is a learning workspace, see learn.md in bun/"
```

Adjust the parent path if `~/banli` doesn't suit you (e.g. `~/code/banli`,
`~/dev/banli`) — the rest of this doc uses `~/banli` literally; substitute
your path everywhere.

### 1. Install Rust (5 min)

Use [rustup](https://rustup.rs/). It's the only supported installer:

```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
# Accept defaults. Then either restart your shell or:
source "$HOME/.cargo/env"

rustc --version    # should print rustc 1.85+ (anything from 2025 onward)
cargo --version
```

Windows: download `rustup-init.exe` from rustup.rs and follow the GUI; the
rest of this doc assumes a POSIX shell (WSL2 is the easy mode on Windows).

Install a couple of components you'll want repeatedly:

```bash
rustup component add rust-src rust-analyzer clippy rustfmt
```

### 2. Install Bun's build prerequisites (10–30 min)

Bun ships a one-shot bootstrap script that installs the system deps it
needs (clang, cmake, ninja, ccache, Python, etc.). It's idempotent and safe
to re-run.

```bash
mkdir -p ~/banli && cd ~/banli
git clone https://github.com/oven-sh/bun.git
cd bun
bash scripts/bootstrap.sh    # Linux/macOS. Windows: scripts/bootstrap.ps1
```

Read the script as it runs — it's the canonical list of every system
dependency Bun needs and what command installs it on each OS.

> If `bootstrap.sh` fails, fix the *first* error it reports and re-run.
> Don't paper over it with manual installs — the script is the spec.

### 3. Build Bun once (30–60 min, mostly unattended)

This is the slow part. Subsequent rebuilds are incremental and fast.

```bash
cd ~/banli/bun
bun bd                       # builds ./build/debug/bun-debug
bun bd -e 'console.log("hi from bun")'
# hi from bun
```

Don't set a timeout on the first build — it can take 30–60 min depending
on hardware. While it compiles, read `CLAUDE.md` (in the bun checkout); it's
the canonical map of where everything lives.

If you get `bun: command not found` here: `bun` itself is a separate install
from rustup. The `bootstrap.sh` script installs it; if it's not on PATH yet,
`curl -fsSL https://bun.sh/install | bash` and reopen your shell.

### 4. Scaffold the `banli` workspace (5 min)

```bash
cd ~/banli

# Your Rust crate (will grow into the parallel mini-Bun across phases).
cargo new banli --bin

# Notes folder — one markdown file per phase, this is where the LEARNING happens.
mkdir notes
cat > notes/README.md <<'EOF'
# Phase notes

One file per phase. Each file ends with:
- 3 things that surprised me
- 1 thing I still don't get
- The commit hash / file:line I was reading
EOF

# Top-level README so future-you remembers what this folder is.
cat > README.md <<'EOF'
# banli — Bun-from-scratch learning workspace

- `bun/`    upstream Bun source (read-only reference)
- `banli/`  my parallel-project Rust crate (mini-Bun)
- `notes/`  per-phase notes

Follow `bun/learn.md`.
EOF

# Initialize git for the workspace itself (banli/ already has its own .git from cargo new).
git init
cat > .gitignore <<'EOF'
bun/                 # don't track upstream Bun in your workspace repo
banli/target/
EOF
git add -A && git commit -m "banli workspace scaffold"
```

Final layout check:

```bash
cd ~/banli && ls -la
# README.md  bun/  banli/  notes/  .git/  .gitignore
```

### 5. Editor setup (5 min, optional but recommended)

VS Code or Cursor with these extensions covers everything you need:

- **rust-analyzer** — Rust LSP (the one true choice).
- **CodeLLDB** — graphical debugger, works on the `banli` Rust binary AND
  on `bun-debug`. Set breakpoints, step through, inspect.
- **Zig** (ziglang.vscode-zig) — syntax highlighting only; you're reading
  Zig, not writing it. You can install this in Phase 1, not now.

Open the workspace as a multi-root project:

```bash
code ~/banli ~/banli/bun ~/banli/banli
```

### 6. Verify everything works

Three quick sanity checks. All three must pass before you start Day 1:

```bash
# Rust: hello-world from the freshly-scaffolded crate
cd ~/banli/banli && cargo run
# Hello, world!

# Bun: debug build runs JS
cd ~/banli/bun && bun bd -e 'console.log(1 + 1)'
# 2

# Bun: tests pass on a tiny file (proves bd test works)
cd ~/banli/bun && bun bd test test/js/bun/util/version.test.ts
# (... pass)
```

- [ ] **My environment is ready:** `cargo run` in `~/banli/banli` prints
      hello-world; `bun bd -e ...` in `~/banli/bun` runs JS; `bun bd test`
      passes on at least one small test.
- [ ] **My workspace exists:** `~/banli/{bun,banli,notes}/` all present;
      `git status` in `~/banli` is clean.

You're now ready for Day 1. **Don't skip the verify step** — catching a
broken toolchain now saves an hour of confusion later.

---

## Day 1 — Hello world (1–3 hours, today)

**Goal:** Two binaries on disk that both print `hi` from JavaScript — one is
Bun's debug build, the other is a 30-line Rust crate you wrote in `~/banli/banli`.
You ship `v0.1-hello` and you can come into Phase 0 with the entire stack
already feeling concrete instead of theoretical.

No Zig, no JSC, no allocators. Just "JS goes in, output comes out."

> Prereq: Day 0 is done. If `cargo run` in `~/banli/banli` doesn't already
> print hello-world, go finish Day 0 first.

### 1. Run JS through Bun (~2 min — you've already done this in Day 0)

```bash
cd ~/banli/bun
bun bd -e 'console.log("hi from bun")'
```

That's the whole loop. `bd` is the package.json script that builds the
debug binary and execs it with your trailing args.

### 2. Run JS through `banli` (~30–60 min)

We'll embed [`rquickjs`](https://crates.io/crates/rquickjs) — a safe Rust
wrapper around the QuickJS C engine. It's the smallest possible JS engine
that lets you call native functions from JS and is the right pick for
`v0.1`. (Later milestones can swap to `rusty_v8` or to JavaScriptCore
directly.)

```bash
cd ~/banli/banli
cargo add rquickjs --features="loader"
```

```rust
// ~/banli/banli/src/main.rs
use rquickjs::{Context, Function, Runtime};

fn main() -> rquickjs::Result<()> {
    let rt = Runtime::new()?;
    let ctx = Context::full(&rt)?;

    ctx.with(|ctx| -> rquickjs::Result<()> {
        // Wire `console.log` to Rust's `println!`.
        let global = ctx.globals();
        let console = rquickjs::Object::new(ctx.clone())?;
        console.set(
            "log",
            Function::new(ctx.clone(), |msg: String| println!("{msg}"))?,
        )?;
        global.set("console", console)?;

        // The user's program (later: read from argv / stdin / a file).
        let src = std::env::args().nth(1).unwrap_or_else(|| {
            "console.log('hi from banli')".to_string()
        });
        ctx.eval::<(), _>(src)?;
        Ok(())
    })?;
    Ok(())
}
```

```bash
cd ~/banli/banli
cargo run
# hi from banli
cargo run -- 'console.log(2 + 2)'
# 4
cargo run -- 'for (const x of [1,2,3]) console.log(x*x)'
# 1
# 4
# 9
```

### 3. Tag the milestone

```bash
cd ~/banli/banli
git add -A && git commit -m "v0.1-hello: embed rquickjs, wire console.log"
git tag v0.1-hello
```

### What you just learned (without me telling you)

- A JS runtime is **embedding a JS engine** + **wiring native functions to
  JS globals** + **running the user's source through `eval`**. The rest of
  Bun — the parser, the bundler, the HTTP server, the package manager — is
  scaffolding around this 30-line core.
- `console.log` isn't magic; it's just a global Object whose `log` property
  is a native function. Every "built-in" in Bun follows this shape.
- You did not need to understand allocators, GC, or the event loop to get
  here. Each phase below adds **one** missing capability.

- [x] **I can explain:** what "embedding a JS engine" means.
- [x] **I can build:** `v0.1-hello` (CLI binary, embedded JS engine,
      `console.log` wired through to native).

---

## Phase 0 — Mental model (week 1)

**Goal:** Stop being scared of the repo. Know what each top-level folder is
for and why Bun chose Zig + JSC instead of Rust + V8. No coding this week
— just orientation.

1. Read, in order, top-to-bottom:
   - `README.md`
   - `CLAUDE.md` (the canonical map of where things live — do not skip)
   - `AGENTS.md`, `CONTRIBUTING.md`
   - `package.json` (note the `bd`, `build`, `build:release`, `zig:check-all`
     scripts — these *are* the dev workflow)
   - `build.zig` and `scripts/build.ts` headers (skim, don't memorize)
2. Watch / read background (skim, don't drill):
   - JavaScriptCore architecture overview (WebKit blog: "Speculation in
     JavaScriptCore", "Introducing Riptide", "FTL"). You don't need to
     understand the JIT yet — just register the vocabulary: **VM, Heap,
     JSGlobalObject, JSCell, Structure, Identifier**.
   - Andrew Kelley's "Practical Data Oriented Design" talk. This is the
     mindset Bun is written in.
   - One libuv tutorial (the *concepts* — handles, requests, the loop, the
     thread pool — transfer cleanly to `src/event_loop/`).
3. **(Rust track)** Skim `PORTING.md` end-to-end — don't try to memorize,
   just register what's there. Read `rust-rewrite-plan.md` §1–2 carefully
   ("Approach" + "JSC Garbage Collector"); it sets up the constraints every
   later phase has to respect.
4. Toolchains you already have (`rustc`, a C/C++ compiler from Day 1's
   `bootstrap.sh`) are enough. **Don't install Zig yet** — you'll do that
   in Phase 1, the moment you actually need to read a `.zig` file.

- [ ] **I can explain:** what JSC's heap is, what a `JSGlobalObject`
      represents, and why Bun chose Zig + JSC over Rust + V8.
- [ ] **I have:** `notes/phase-0.md` with one paragraph each on those three
      questions, plus a list of every Bun top-level folder + a one-line
      description of what's in it (cross-checked against `CLAUDE.md`).

---

## Phase 1 — Build it, run it, break it (week 2)

**Goal:** A working debug build (you have this from Day 1), a working
debugger session, the muscle memory to round-trip a code change, and just
enough Zig to read Bun — not to write it.

1. Re-run `bun bd` if anything has changed since Day 1. While it compiles,
   read `scripts/build.ts`'s header so you understand the arg routing
   (`--asan=off`, `--build-dir`, build-then-exec trailing args).
2. Run the debug binary three ways:
   - `bun bd -e 'console.log(Bun.version)'` (you've done this)
   - `bun bd test test/js/bun/util/version.test.ts` (or any small file)
   - `lldb -- ./build/debug/bun-debug -e 'throw new Error("hi")'` — set a
     breakpoint on `JSC::Exception::create`, hit it, inspect the stack.
3. **Learn just enough Zig to read Bun.** Don't do a full Zig course — you
   only need *reading* fluency. Spend 90 min on:
   - ziglearn.org chapter 1 (basics, `defer`, `errdefer`, optional types).
   - The Zig language reference sections on **`comptime`**, **error sets**,
     **allocator passing**, **`struct`/`union(enum)`**, **`extern`**. Skip
     async, skip metaprogramming-as-codegen — you'll meet those in context.
   - `PORTING.md` is the dictionary that tells you what each Zig construct
     means in Rust. You don't need to memorize Zig idioms; you need to
     **recognize** them.
4. Make a deliberately broken change in `src/cli.zig` (e.g. flip an `if`
   or change a literal string), rebuild, watch the test fail, revert. You
   now have the inner loop.
5. Read `src/codegen/` README-by-grep:
   - `generate-classes.ts` — turns `*.classes.ts` into Zig + C++ glue.
   - `bundle-modules.ts`, `bundle-functions.ts` — embed `src/js/` into the
     binary.
   - Modify a string in `src/js/bun/<something>.ts`, run `bun run build`
     (no Zig rebuild needed!), see the change.
6. Grow your Day 1 clone: have it read JS from a file path argument
   (`banli script.js` — from `~/banli/banli`, that's `cargo run -- script.js`)
   and exit with a non-zero code on JS exception. This is the smallest step
   that makes it feel like a real CLI.

- [ ] **I can explain:** what a "build-then-exec" command does, how `src/js/`
      gets into the binary, and which kinds of changes need a full `bd`
      rebuild vs. just a JS bundle rebuild.
- [ ] **I can read** (not write): a Zig file with `defer`/`errdefer`,
      `comptime`, error unions, and `*Allocator` parameters — and translate
      it in my head to the Rust equivalent using `PORTING.md`.
- [ ] **`banli` runs `cargo run -- path/to/file.js`** and propagates exception
      exit codes correctly. (Tag: `v0.1.1-file-input`.)

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

**Rust-track companion reading** (`PORTING.md` + `rust-rewrite-plan.md`):

- `PORTING.md` §Allocators, §Strings, §Collections, §Pointers & ownership.
- `rust-rewrite-plan.md` §4 (`JSValue`, `WTFStringImpl`, `ZigString`/`BunString`,
  SIMD scans), §8 (`mimalloc`/`MimallocArena`/`NewStore`/`HiveArray`/`BabyList`/
  `MultiArrayList`/`RefCount`/`TaggedPointerUnion`).
- `LIFETIMES.tsv` — grep for any `?*T`/`*T` field in a struct you're porting;
  the `rust_type` column tells you `Box`/`Rc`/`Arc`/`&'static`/raw-ptr without
  guessing.
- Decision rules to commit to memory:
  - `[]const u8` field → look at `deinit`: freed → `Box<[u8]>`/`Vec<u8>`;
    not freed, only literals → `&'static [u8]`; arena → raw `*const [u8]`.
  - `bun.String` stays `bun_str::String` (5-variant `#[repr(C)]` tagged union,
    NOT a Rust `enum` — C++ mutates `tag` and `value` independently). Don't
    "simplify" to `Arc<str>` or you lose zero-copy JSC interop.
  - `bun.ptr.RefCount` → default to `Rc<T>`/`Arc<T>`; only stay intrusive
    when `*mut T` crosses FFI and C++ calls `ref()`/`deref()` on it.
  - `BabyList<T>` → `ThinVec<T>` only on the 6 hot AST fields; `Vec<T>`
    everywhere else.
  - `TaggedPointerUnion` → stay packed (`#[repr(transparent)] u64`); don't
    expand to a Rust `enum` (8→16B is load-bearing in arrays/hashes).

Hands-on:

- In `lldb`, break on `bun.default_allocator.alloc` (or whatever the current
  symbol is) and watch a `bun -e '({a:1})'` execution. Where does the
  allocator come from? Who frees it?
- In `~/banli/banli`, allocate 1M short strings two ways: with the default
  `String`, and with `bumpalo::Bump`. Time them. Feel why Bun is
  arena-happy.
- Add a `BanliString` type to your crate: 24 bytes, with the small-string
  optimization (≤23 ASCII bytes inline, otherwise heap). Use `&[u8]`
  internally, not `&str` — you're following the rule from `PORTING.md`
  §Strings.

- [ ] **I can explain:** when Bun uses an arena vs. mimalloc vs. the JSC heap,
      and what owns what across a `Bun.serve` request lifecycle.
- [ ] **I can build:** `v0.2-strings` — a `banli::str` module with
      `BanliString` (inline-short-string) and a `Bump`-arena helper. Tests
      cover the inline↔heap boundary.

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
   - **(Rust track)** Read `rust-rewrite-plan.md` §2 (the GC table), §3
     (codegen contract — the symbol table for what `.classes.ts` emits),
     §5 (the six liveness primitives: `hasPendingActivity`, `JSRef`,
     `Strong`, `protect`/`unprotect`, `KeepAlive`, `MarkedArgumentBuffer`),
     §6 (Category A vs B objects). The JSC GC rules are language-agnostic;
     these sections give you the *Rust mapping* of every rule the SKILLs
     state in C++/Zig terms.
   - Key sentinels worth memorizing: `JSValue` is `i64`, `Copy`, `!Send`,
     non-moving. `Strong::get()` is a zero-FFI direct deref, NOT a call.
     Conservative stack scan means stack `JSValue`s are auto-rooted; heap
     storage needs `WriteBarrier` via the generated `*SetCachedValue` extern.
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
6. In `~/banli/banli`, add a `Banli.md5(input: string): string` global,
   calling Rust's `md5` crate or `ring`. This is the smallest useful native
   class — takes a JS string, returns a JS string.
7. **(Rust track)** Pick a small Zig file with a `.classes.ts` partner and a
   ported `.rs` sibling on `claude/phase-a-port` (`git ls-tree -r --name-only
   claude/phase-a-port | grep '\.rs$'`). Diff the pair line-by-line; map every
   construct back to `PORTING.md`'s idiom table. Then port a *different*
   small Zig file yourself (no `.rs` sibling yet) and check your work against
   the patterns you saw.

- [ ] **I can explain:** what `*.classes.ts` generates, the difference between
      `JSDestructibleObject` and `JSNonFinalObject`, and why every C++ method
      that may throw needs an exception scope.
- [ ] **I can build:** `v0.3-class` — `Banli.md5("hello")` returns the
      correct hex digest in your crate, with at least one test.

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

> **(Rust track)** This is the densest `comptime` zone in Bun. Read
> `PORTING.md` §"Comptime reflection" *before* you start, plus
> `rust-rewrite-plan.md` §8.1–8.3 (`NewStore`, `Expr`/`Stmt` layout, `Ref`)
> — the AST node store is *the* hard part. Heads-up: the `'ast` lifetime
> threading is the dominant cost; `StoreRef<T>` deliberately exposes
> `unsafe fn as_mut` rather than safe `&mut T` because the visitor aliases
> children mid-recursion.

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
- In `banli`, write a tiny TS-subset parser: handle `const`, `function`,
  arrow functions, type annotations (just strip them), JSX. Recursive
  descent, hand-rolled lexer; allocate AST nodes in a `bumpalo::Bump`
  per file. You don't need types — just a working AST that round-trips
  back to JS.

- [ ] **I can explain:** why a recursive-descent parser with a hand-rolled
      lexer beats parser generators here, and what Bun's AST looks like in
      memory (it's not what you think — look at `Stmt`/`Expr` representations).
- [ ] **I can build:** `v0.4-tspile` — `banli transpile foo.tsx` produces
      working JS with at least one source-map mapping per statement.

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
- In `banli`, implement Node-compat resolution: `exports` conditions,
  `imports`, the scoped-package walk, the `directory + index.js` fallback.
  Test against [`resolve.exports`](https://github.com/lukeed/resolve.exports)
  fixtures.

- [ ] **I can explain:** what "conditions" are, how Bun decides between
      `node` and `bun` exports, and why CJS lazy-binding (`module.exports = ...`
      after `require`) makes static analysis hard.
- [ ] **I can build:** `v0.5-resolve` — a Node/Bun-compatible resolver that
      passes ≥80% of `resolve.exports` fixtures.

---

## Phase 6 — The bundler (weeks 9–10)

**Goal:** From AST to a tree-shaken, minified, code-split bundle.

> **(Rust track)** The bundler is where the port has the most ported `.rs`
> files — `src/bundler/HTMLImportManifest.rs`,
> `src/bundler/barrel_imports.rs`, `src/bundler/linker_context/*.rs`. Pick
> one and diff against its `.zig` sibling; this is the single best worked
> example of multi-file porting in the repo. Also read `CYCLEBREAK.md` once
> here — you'll start to feel why crate boundaries had to be redrawn.

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
- In `banli`, build the bundler in two passes: first a single-file
  IIFE bundler (parse → resolve → emit), then add mark-and-sweep tree
  shaking on top. Don't try to do both at once.

- [ ] **I can explain:** the difference between Bun's bundler IR and its
      runtime AST, what "splitting" buys you, and where source-map fidelity
      comes from.
- [ ] **I can build:** `v0.6-bundle` — a multi-file bundler with mark-and-sweep
      tree-shaking and source maps that load in Chrome DevTools.

---

## Phase 7 — Networking: HTTP server & client (weeks 11–12)

**Goal:** Understand how `Bun.serve` is the fastest JS HTTP server.

> **(Rust track)** Read `rust-rewrite-plan.md` §7 (`bun_sys`) before this
> phase. The table comparing `bun.sys` to `std::fs`/`std::net` is the
> single most important argument in the whole port: errno semantics, FD
> tagging, EINTR retry, sentinel paths, `MAX_COUNT` clamping — all reasons
> Rust's `std` would silently break Bun. Your clone's `serve()` should
> wrap raw syscalls the same way, not lean on `tokio::net`.

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
- In `banli`: write an `epoll`/`kqueue` echo server (use `mio` directly,
  **not** `tokio` — you want to feel the loop). Then layer HTTP/1.1 parsing
  using `picohttpparser` (vendored in Bun at `vendor/picohttpparser/`) or a
  pure-Rust equivalent.
- Add a `serve()` global on top of your JS-engine embedding from Phase 3:
  the JS callback returns a `Response`-shaped object, your Rust loop
  serializes it back to the socket.

- [ ] **I can explain:** how Bun avoids per-request allocations, what
      `corked` writes are, and why µWebSockets' state-machine HTTP parser is
      faster than a callback-per-header parser.
- [ ] **I can build:** `v0.7-serve` — a JS-callable `serve()` that does
      ≥100k req/s on `wrk` for a static `Response`. (You will not match
      Bun. That's fine.)

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
- In `banli`: write `install` in three stages. Stage 1: handle a single
  dependency with no transitive deps. Stage 2: add a manifest cache.
  Stage 3: add the resolver. **Stop before lockfile** — that's a multi-week
  side quest of its own and not on the critical path.

- [ ] **I can explain:** Bun's content-addressable cache layout, why the
      lockfile is binary-then-text, and how the install graph parallelism is
      structured.
- [ ] **I can build:** `v0.8-install` — `banli install pkg@version` that
      downloads the tarball, verifies the integrity hash, extracts it, and
      links it into `node_modules`.

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
- [ ] **I can build:** `v0.9-test` — a `bun:test`-style runner around
      `banli`'s JS embedding: `describe`, `test`, `expect`, snapshot,
      `--watch`.

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
- [ ] **I can build:** `v1.0` — a benchmark harness for `banli` that
      produces stable numbers (warm-up, GC control, statistical
      significance, not "I ran it once"), plus a short blog post explaining
      what's still slow vs Bun and why. The blog post is the proof you
      understood.

---

## Parallel-project milestones (your `banli` crate)

Track these alongside the phases above. Each milestone is a tag in
`~/banli/banli`'s git history.

| Tag | Capabilities |
| --- | --- |
| `v0.1-hello` | CLI binary, embedded JS engine (rquickjs), `console.log` wired through to native. |
| `v0.1.1-file-input` | Reads JS from a file path arg; non-zero exit on uncaught exception. |
| `v0.2-strings` | `BanliString` (inline-short-string) + arena allocator. |
| `v0.3-class` | One native class (`Banli.md5`) on the JS global. |
| `v0.4-tspile` | TS+JSX → JS transpiler with source maps. |
| `v0.5-resolve` | Node/Bun-compatible resolver, ≥80% on resolve.exports. |
| `v0.6-bundle` | Multi-file bundler with tree-shaking. |
| `v0.7-serve` | `serve()` doing ≥100k req/s on `wrk`. |
| `v0.8-install` | `install pkg@version` that links into `node_modules`. |
| `v0.9-test`   | `banli-test` runner with `expect` + snapshots. |
| `v1.0`        | A *short* blog post explaining what's still slow vs Bun and why. The blog post is the proof you understood. |

---

## Reading-list shortlist (no fluff)

- **Zig**: Loris Cro's blog (especially "What is Zig's Comptime?"); Andrew
  Kelley's talks; the Zig std lib source.
- **Rust (port-relevant)**: the Rustonomicon (chapters on `repr`, FFI,
  subtyping/variance); strict-provenance RFC #3559; the `bumpalo` and
  `typed-arena` docs; `parking_lot` README. Skip async-Rust material — it's
  irrelevant to Bun.
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
- **(Rust track) Reaching for `tokio`/`async fn`/`anyhow`/`String` because
  the Rust crowd does.** All four are explicitly forbidden by `PORTING.md`
  and each one breaks something concrete: `tokio` competes with the
  uSockets loop; `async fn` hides the state machines you need to debug;
  `anyhow` heap-allocates and breaks `@errorName` snapshots; `String`
  forces UTF-8 validation on bytes that may not be UTF-8 (Linux paths,
  WTF-16 surrogates).
- **(Rust track) `unwrap()` on `from_utf8` of external bytes.** Fastest way
  to ship a CVE on day one.
- **(Rust track) `Box::leak`/`mem::forget`/`ManuallyDrop` to silence the
  borrow checker.** If the Zig freed it, the Rust must too. Restructure
  ownership instead.

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

### Phase ↔ milestone map

| Phase | Time | Parallel-project milestone you ship at the end |
| --- | --- | --- |
| Day 0 | 1–2 hours | Dev env ready: `~/banli/` workspace, Rust toolchain, Bun source cloned + bootstrap.sh run. No code yet. |
| Day 1 | 1–3 hours | `v0.1-hello` — `cargo run` evaluates `console.log("hi")` via an embedded JS engine, alongside `bun bd -e` doing the same. |
| 0 | week 1 | (no artifact — just notes + mental model) |
| 1 | week 2 | A working `bun bd` build, an `lldb` session, and a deliberately broken-then-fixed Zig change. |
| 2 | week 3 | `v0.2-strings` — string library with inline-short-string optimization + bump allocator. |
| 3 | weeks 4–5 | `v0.3-class` — one native class (`MD5`) wired into your JS global. |
| 4 | weeks 6–7 | `v0.4-tspile` — TS+JSX → JS transpiler with source maps. |
| 5 | week 8 | `v0.5-resolve` — Node/Bun-compatible resolver passing ≥80% of `resolve.exports` fixtures. |
| 6 | weeks 9–10 | `v0.6-bundle` — multi-file bundler with mark-and-sweep tree-shaking. |
| 7 | weeks 11–12 | `v0.7-serve` — a `serve()` doing ≥100k req/s on `wrk`. |
| 8 | weeks 13–14 | `v0.8-install` — `install pkg@version` that links into `node_modules`. |
| 9 | weeks 15–16 | `v0.9-test` — `banli-test` runner with `expect` + snapshots. |
| 10 | week 17+ | `v1.0` — a short blog post explaining what's still slow vs Bun and why. |

---

## Appendix A — Zig → Rust quick-reference (Rust track only)

This is a *skim-for-recognition* index. The authoritative version is
`PORTING.md` on `claude/phase-a-port`; these are the rows you'll re-read
weekly. Keep both open.

### Types

| Zig | Rust |
| --- | --- |
| `[]const u8` (param/return) | `&[u8]` — never `&str` |
| `[]const u8` (struct field) | `Box<[u8]>` if freed; `&'static [u8]` if literal-only; arena → raw `*const [u8]`. Consult `LIFETIMES.tsv`. |
| `[:0]const u8` | `&ZStr` (`bun_str::ZStr`); len excludes trailing NUL |
| `?T` / `?*T` | `Option<T>` / `Option<&T>` (or per-field rust_type from `LIFETIMES.tsv` for struct fields) |
| `anyerror!T` | `Result<T, bun_core::Error>` (`Error` = `Copy` `NonZeroU16` newtype) |
| `OOM!T` | `Result<T, bun_alloc::AllocError>` |
| `bun.JSError!T` | `bun_jsc::JsResult<T>` |
| `Maybe(T)` (`bun.sys`) | `bun_sys::Result<T>` |
| `JSC.JSValue` | `bun_jsc::JSValue` (`#[repr(transparent)] i64`, `Copy`, `!Send`) |
| `bun.String` | `bun_str::String` (5-variant `#[repr(C)]` tagged union — NOT a Rust enum) |
| `bun.PathBuffer` | `bun_paths::PathBuffer` |
| `extern struct` / `enum(uN)` | `#[repr(C)] struct` / `#[repr(uN)] enum` |
| `union(enum)` | Rust `enum` with payload variants (Rust enums *are* tagged unions) |
| `packed struct(uN)` | `bitflags!` if all bools; else `#[repr(transparent)] struct(uN)` with manual accessors |

### Idioms

| Zig | Rust |
| --- | --- |
| `defer x.deinit()` | **Delete the line.** `impl Drop for T` runs at scope exit. |
| `pub fn deinit` | `impl Drop` (delete body if it just frees owned fields — `Box`/`Vec` self-drop) |
| `errdefer x.deinit()` (just-allocated local) | **Delete it.** `?` drops the `Vec`/`Box` on the error path. |
| `errdefer { rollback }` (real side effects) | `let g = scopeguard::guard(...); ScopeGuard::into_inner(g)` on success |
| `comptime T: type` | plain generic `<T>` with a trait bound for the methods called |
| `comptime flag: bool` | `<const FLAG: bool>`; demote to runtime + `// PERF(port)` if only forwarded |
| `try x` | `x?` |
| `x catch unreachable` | `x.expect("unreachable")` (NEVER `?`, NEVER `unwrap_unchecked`) |
| `orelse` | `.unwrap_or` / `.ok_or(..)?` / `let Some(x) = .. else { .. }` |
| `if (x) \|y\|` / `while (it.next()) \|x\|` | `if let Some(y) = x` / `while let Some(x) = it.next()` |
| `for (slice, 0..) \|x, i\|` | `for (i, x) in slice.iter().enumerate()` |
| `for (a, b) \|x, y\|` | `for (x, y) in a.iter().zip(b)` + `debug_assert_eq!(a.len(), b.len())` |
| `@memcpy(dst, src)` | `dst.copy_from_slice(src)` |
| `@intCast(x)` (narrowing) | `T::try_from(x).unwrap()` — NEVER bare `as` for narrowing |
| `@truncate(x)` | `x as T` (intentional wrap) |
| `@tagName(e)` / `@errorName(e)` | `<&'static str>::from(e)` via `#[derive(strum::IntoStaticStr)]` |
| `a +\| b` / `a +% b` | `.saturating_add(b)` / `.wrapping_add(b)` — never bare `+` |
| `bun.assert(x)` | `debug_assert!(x)` |
| `unreachable` | `unreachable!()` |
| `threadlocal var X: T` | `thread_local! { static X: Cell<T> = const { Cell::new(init) }; }` |

### Allocators (the rule that shocks Zig devs)

- **AST/parser/bundler/CSS crates**: keep arenas. `MimallocArena` →
  `bumpalo::Bump`. `ASTMemoryAllocator` → `typed-arena`. Thread `'bump` /
  `'ast` lifetimes.
- **Everything else**: delete the `Allocator` param. `Box`/`Vec`/`String`
  hit mimalloc via `#[global_allocator]`. `bun.default_allocator` → just
  delete the expression. `bun.handleOom(x)` → `x` (Rust aborts on OOM by
  default).

### Concurrency

- `Lock + bool + data` (lazy init) → `OnceLock<T>` / `LazyLock<T>`.
- `Lock` around defensive state on the JS thread → **delete it**; the type
  is `!Sync` (contains `JSValue`/`*mut JSGlobalObject`); compiler proves it.
- Genuinely cross-thread → `parking_lot::Mutex<T>` (owns `T`) or `RwLock<T>`.
  Never `std::sync::Mutex` (poisoning is noise).
- Atomics → `core::sync::atomic::Atomic*` with the same orderings
  (`.acquire`→`Acquire`, etc.).

### Pointers & ownership

- `bun.ptr.Owned(T)` → `Box<T>`
- `bun.ptr.Shared(*T)` → `Rc<T>` (single-thread)
- `bun.ptr.AtomicShared(*T)` → `Arc<T>`
- `bun.ptr.RefCount` → default to `Rc`/`Arc`; only stay intrusive when
  `*mut T` crosses FFI and C++ calls `ref()`/`deref()` on the raw pointer.
- `bun.ptr.TaggedPointerUnion` → `bun_collections::TaggedPtrUnion` (stays
  packed `u64` — DO NOT expand to `enum`).
- `@fieldParentPtr("field", ptr)` → `core::mem::offset_of!(Parent, field)`
  (stable since 1.77) + `unsafe` raw-ptr arithmetic with `// SAFETY:`.

### Forbidden in the port (will fail review)

- `tokio` / `async fn` / `Future` / `Waker` / `rayon` / `hyper` / `async-trait`
- `std::fs` / `std::net` / `std::process` (use `bun_sys`)
- `String` / `&str` for paths / source / HTTP / env (use `&[u8]` / `Vec<u8>`)
- `anyhow::Error` / `Box<dyn Error>` (use `bun_core::Error`)
- `Box::leak` / `mem::forget` / `ManuallyDrop` to satisfy `'static` or appease borrowck
- `todo!()` / `unimplemented!()` / `#[cfg(any())]` as stubs
- Re-implementing C/C++ libs in Rust (BoringSSL, simdutf, highway, mimalloc,
  libarchive, lol-html, etc.) — link them via `extern "C"`.

### When you're stuck on a field type

```bash
# In your worktree of claude/phase-a-port:
awk -F'\t' -v f="src/path/to/file.zig" -v s="MyStruct" \
  '$1==f && $2==s' docs/LIFETIMES.tsv
```

The `rust_type` column is your answer. The `class` column tells you why:
`OWNED`, `SHARED`, `BORROW_PARAM`, `STATIC`, `JSC_BORROW`, `BACKREF`,
`INTRUSIVE`, `FFI`, `ARENA`, `UNKNOWN`. If `UNKNOWN`, leave
`Option<NonNull<T>>` + `// TODO(port): lifetime` and move on — Phase B
is when you narrow.

---

*Last updated: drafted on `zhichli/learn`, augmented with `claude/phase-a-port`
porting docs as the Rust translation reference. Living document — edit as
understanding deepens.*
