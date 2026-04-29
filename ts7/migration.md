---
title: TypeScript 7 migration notes
description: Notes on TypeScript 7, typescript-go, and migration approaches for projects that transpile TypeScript to Lua.
---

qx7m

Exploratory notes on how projects that transpile TypeScript to Lua might migrate to TypeScript 7, the Go rewrite of `tsc`. Perf comparisons here are early guesses; better numbers would need actual measurement.

Most of this thinking comes out of two prototypes: `dunder` (Option 1, IPC consumer with bulk-resolution) and `tslua` (Option 3, in-process Go). Option 2 hasn't been explored in much depth here. The substrategies that get treated as tractable, and the costs that get treated as pressing, reflect those two vantage points. The protocol-change proposals later come out of `dunder`'s pain points specifically. Read with that in mind.

iSentinel raised the shared-core framing in the TSTL Discord on April 3, 2026, in the cross-community conversation about TS 7. This post is partly a response to that prompt.

## Sources

### Announcements

- [TypeScript's Native Port](https://devblogs.microsoft.com/typescript/typescript-native-port/), Mar 2025.
- [Progress on TypeScript 7](https://devblogs.microsoft.com/typescript/progress-on-typescript-7-december-2025/), Dec 2, 2025.
- [Announcing TypeScript 6.0 Beta](https://devblogs.microsoft.com/typescript/announcing-typescript-6-0-beta/), Feb 11, 2026.
- [Announcing TypeScript 6.0 RC](https://devblogs.microsoft.com/typescript/announcing-typescript-6-0-rc/), Mar 6, 2026.
- [Announcing TypeScript 6.0](https://devblogs.microsoft.com/typescript/announcing-typescript-6-0/), Mar 23, 2026.
- [Announcing TypeScript 7.0 Beta](https://devblogs.microsoft.com/typescript/announcing-typescript-7-0-beta/), Apr 21, 2026.

6.0 is the bridge release between the JS line and 7.0. There is no planned 6.1, only limited 6.0.x servicing, so Option 0 reads less like "do nothing" and more like the compatibility path the TS team is explicitly preserving during the transition. The 7.0 beta post also says the stable TS 7 programmatic API is a 7.1-or-later item, which matters most to Option 1.

### Repo, discussions, issues

- [microsoft/typescript-go](https://github.com/microsoft/typescript-go), staging repo for the Go port.
- [Discussion #455, "What is the API story?"](https://github.com/microsoft/typescript-go/discussions/455). Official answer points at IPC; API is "curated," not exhaustive.
- [Discussion #481, "Public Go API for embedding?"](https://github.com/microsoft/typescript-go/discussions/481). "Unlikely" initially, softened in June 2025 to "thinking about it."
- [Discussion #458, "Will this run in browser (WASM)?"](https://github.com/microsoft/typescript-go/discussions/458). No plan; size and Go-WASM perf flagged.
- [Discussion #514, "Go WASM performance"](https://github.com/microsoft/typescript-go/discussions/514). Community forks reuse Program objects for steady-state perf; unofficial.
- [Issue #516, "Transformer plugin / compiler API"](https://github.com/microsoft/typescript-go/issues/516). What happens to compiler plugins.
- [Issue #3610, "Additional Checker methods for transpiler consumers"](https://github.com/microsoft/typescript-go/issues/3610) (filed by me). Call-count data from a real transpiler, plus checker methods not yet exposed.

### Code and community implementations

- [`internal/api/proto.go`](https://github.com/microsoft/typescript-go/blob/main/internal/api/proto.go) and [`session.go`](https://github.com/microsoft/typescript-go/blob/main/internal/api/session.go). Current IPC API, with batch variants for hot methods. Queries return opaque handles, not serialized type graphs.
- [rictic's WASM playground fork](https://ts-go-playground.rictic.com/). `tsgo` exposed to JS via WASM with an ad-hoc Compiler API surface.

### External writeups

- [Oxlint type-aware linting blog post, August 2025](https://oxc.rs/blog/2025-08-17-oxlint-type-aware). Oxlint's architecture (Rust CLI + Go binary with shimmed `typescript-go` internals); IPC was explored and abandoned.
- [oxc Discussion #2855, "Type-aware linting via IPC"](https://github.com/oxc-project/oxc/discussions/2855). Earlier IPC-bridge attempt from Rust to a Node TS host; concluded not viable.

## Options

A few directions surfaced while reading around. Two kinds of extension point come up in the bullets and are worth distinguishing: *transformers*, the pre-emit TS-to-TS passes that run before the transpiler proper (the `compilerOptions.plugins` shape), and *plugins*, the transpiler-internal hooks that operate on the target AST or the printer.

At a glance:

- **Option 0: Stay on TypeScript 6.** Keep depending on the JavaScript Compiler API; don't migrate.
- **Option 1: IPC API with batched queries.** JS/TS host talks to a `tsgo` subprocess over IPC.
- **Option 2: WASM tsgo, in-process.** JS/TS host links a WASM build of `tsgo` and calls it across the JS-WASM boundary.
- **Option 3: In-process Go.** Rewrite the transpiler in Go and link `tsgo` directly.

### Option 0: Stay on TypeScript 6 (for now)

Don't migrate. Keep depending on `import "typescript"` and the JavaScript Compiler API for as long as TS 6 is maintained.

- TS 7 may evolve non-trivially before it stabilizes; waiting lets others surface integration issues first.
- Microsoft hasn't announced an EOL for TS 6.
- No migration cost.
- The JS Compiler API is stable, well-documented, and unchanged.
- Many transpilers don't depend on TS-language features past a certain point, so missing future TS 7+ language additions may not matter for the transpile pipeline itself.
- The cost is gradual ecosystem drift: editor tooling, type definitions, and dependencies will bias toward TS 7 over time, and projects upstream of the transpiler may eventually require TS 7 to typecheck. Pace of drift is slow but real.

### Option 1: IPC API with batched queries

Stay in JS/TS. Speak to a `tsgo` subprocess via the [IPC API](https://github.com/microsoft/typescript-go/blob/main/internal/api/proto.go). Several different substrategies for getting type info out of `tsgo` efficiently fit under this umbrella; see [Inside Option 1](#inside-option-1-ipc-substrategies) below.

- Intended upstream direction, but the TS team says the stable programmatic API is a 7.1-or-later item rather than part of 7.0 itself.
- Plausibly the ecosystem hub long-term: other Compiler-API consumers are most likely to converge here, with the usual network effects (shared tooling, examples, conventions).
- Keeps the host in JS/TS, which preserves the general integration shape. Existing plugins and transformers still depend on what the eventual TS 7 programmatic API actually exposes.
- Handles (opaque IDs the server resolves on later calls) avoid serializing type graphs across the wire.
- API surface still being designed; some checker methods used by current transpilers (parts of [#3610](https://github.com/microsoft/typescript-go/issues/3610), emit-resolver-shaped queries) may not be exposed.
- Per-call IPC overhead is the main perf risk; how much it matters depends heavily on the substrategy chosen.

Precedents:

- The IPC API ships as `@typescript/native-preview`, but most npm dependents invoke it as a `tsgo --noEmit` CLI rather than as a Compiler-API consumer.
- [`dunder`](https://github.com/RealColdFry/dunder) (my own experiment) is a consumer-side prototype using `@typescript/native-preview/async` with batched calls and a typed-AST-walk pattern.
- Couldn't find other projects using the IPC API in a Compiler-API-consumer shape; pointers welcome.

### Option 2: WASM tsgo, in-process

Compile `tsgo` to WebAssembly and link it into a JS host. Checker calls cross the JS-WASM boundary instead of a process boundary.

- Per-call overhead should be lower than IPC, since there is no process boundary to cross, but not sure how much.
- Preserves the existing transpiler shape almost unchanged: visitor walks the TS AST and asks the checker just as it does today, only the checker lives in WASM instead of in-process JS.
- Keeps the host in JS/TS, and in principle could preserve more of today's integration shape than an out-of-proc IPC design. Existing plugins and transformers still depend on having a sufficiently TS-6-like API surface available from the WASM build.
- Not an official Microsoft artifact today ([#458](https://github.com/microsoft/typescript-go/discussions/458), [#514](https://github.com/microsoft/typescript-go/discussions/514)). This means depending on community WASM builds, or building one with the necessary Compiler API surface exposed.
- Go compiled to WASM is slower than native Go at steady state, and the bundle is big; wants measurement. Rough size guess is around 40 MB: `tslua`'s WASM build is 35 MB, and a naive full build of `tsgo` with unneeded things included is 48 MB.
- Distribution becomes a hybrid Node/TS package plus a WASM artifact, with the build-pipeline and bundle-size wrinkles that come with that.

Feels like an interesting path to explore.

Precedents:

- [`sxzz/tsgo-wasm`](https://github.com/sxzz/tsgo-wasm) builds `tsgo` to WASM with a CLI-shaped entrypoint and runs the full type checker. The hosted playground takes ~1.4s on a small file, though that may partly reflect the CLI-shaped surface and other architectural differences rather than WASM steady-state perf in general.
- [`rictic/typescript-go-playground`](https://ts-go-playground.rictic.com/) is a fork of sxzz's playground exploring optimized emit. Its readme notes that it skips type checking and uses a modified `typescript-go` with a new WASM entrypoint that creates `Program` objects efficiently; emits land in ~10ms. Type-error diagnostics in the playground UI come from Monaco's TS language service, not from this build.
- No transpiler, linter, or codegen tool consumes WASM-tsgo today.
- A contrasting data point: [`tslua`](https://github.com/RealColdFry/tslua) (my own experiment, primarily a native Go binary; the WASM build exists for an in-browser playground) runs full type-check plus transpile in ~140-160ms steady-state median on a small file (browser, with ~280ms cold first call). The gap to sxzz's ~1.4s is most likely CLI startup and JS-backed VFS reads rather than WASM execution itself.

### Option 3: In-process Go, with linkname today and a public Go API later

Port the transpiler to Go and link `tsgo` directly. Today this would mean [`go:linkname` shims](https://github.com/oxc-project/tsgolint) that expose `typescript-go/internal/*` packages despite Go's package privacy. If [#481](https://github.com/microsoft/typescript-go/discussions/481) lands a public Go API, some of those shims could likely turn into normal imports, though the exact migration depends on what surface actually gets exposed.

- Should be the fastest of the three, since there is no IPC and no WASM layer.
- Single binary distribution.
- JS-based plugins break. Go binaries can't load arbitrary JS code, so existing target-AST and printer hooks don't transfer. Transformers could perhaps still be supported by shelling out to `tsgo` IPC for the transformer pass and feeding the transformed AST back into the Go pipeline, though that's a fair amount of extra plumbing.
- Tracking unexported `typescript-go` internals is an ongoing maintenance cost; breaks on upstream refactors. Less of a concern if [#481](https://github.com/microsoft/typescript-go/discussions/481) eventually ships.

Precedents:

- [`auvred/tsgolint`](https://github.com/auvred/tsgolint) pioneered the `go:linkname` shim pattern.
- [`oxc-project/tsgolint`](https://github.com/oxc-project/tsgolint) is VoidZero's fork powering Oxlint's type-aware rules, with a two-binary architecture (oxlint in Rust drives the CLI, tsgolint in Go does type-aware checks via shimmed `typescript-go` internals).
- [Rslint](https://github.com/rslint/rslint) is another fork aimed at an ESLint-compatible TypeScript-first linter.
- The [Oxlint writeup](https://oxc.rs/blog/2025-08-17-oxlint-type-aware) notes IPC was evaluated and abandoned on perf grounds before settling on shims (see also [oxc #2855](https://github.com/oxc-project/oxc/discussions/2855) for the earlier IPC-bridge attempt that was concluded not viable).
- [`tslua`](https://github.com/RealColdFry/tslua) (my own experiment) is an in-process-Go TS-to-Lua transpiler that links `tsgo` directly via `go:linkname` shims. On a real mid-sized project (~180 source files), a clean transpile takes ~550ms; watch-mode incremental rebuilds land at ~6-20ms (using `typescript-go`'s `incremental.NewProgram` for snapshot-diffed program updates, with type-check diagnostics running async off the hot path). Useful as a "what in-process Go can do" data point for thinking about IPC-class options.

## Inside Option 1: IPC substrategies

The "type-collect pass + transform pass" shape leaves a lot of design room. What follows is loose brainstorming, mostly my own speculation rather than a survey of existing approaches.

- **Bulk-resolve upfront with an IR.** First pass walks the TS AST, resolves all type positions via batch checker calls, and lowers into an IR that carries the type info inline. Later passes run against the IR. Roughly what `dunder` does.
- **Speculative/eager prefetch with cache.** Visitor walks once, fires batch queries for every plausibly-needed position, transform pass reads from a cache.
- **Naive in-line queries.** Existing transpiler shape, checker queries fired inline as the visitor needs them.
- **Fully async, branching visitor.** Async-aware tree walk: checker calls hit an async layer, other subtrees keep transpiling while a query is in flight.
- **Multi-pass with holes.** First pass produces a target AST with holes wherever type info is needed; later passes fill them, possibly generating more.
- **DSL-driven variation of the above.** Encode transpiler logic as a DAG of dependencies rather than imperative if/then/recurse, so a "query driver" can inspect the whole graph and plan optimal batching against `tsgo`, similar in spirit to SQL query planning.
- **Local-heuristic shortcut layer.** Resolve obviously-cheap cases without IPC; escalate hard cases to `tsgo`.
- **Worker-pool sharding.** Multiple `tsgo` subprocesses, transpiler fans out. `tsgo` is already concurrent internally, so whether this helps depends on where the bottleneck actually sits.

These aren't mutually exclusive; a real implementation probably combines a couple.

Caveat for the concurrency-heavy substrategies (async branching, worker-pool sharding, large pipelined batches):

- high in-flight queue depths could saturate the JSON-RPC client, and
- `tsgo` currently appears to use a per-handler checker pool that may hand different checker instances to concurrent handlers, potentially breaking type-identity across parallel calls.

Both are hunches from poking around, not measured.

## Possible protocol changes

A couple of `tsgo` IPC API ideas surfaced while thinking through Option 1. Both would shift what's possible; both apply broadly, not just transpiling TS to Lua.

- **Batch envelope with shared checker checkout.** A `"batch"` IPC method that takes an array of inner requests and returns an array of results. The handler dispatches each inner request through the existing per-method handlers, with one `program.GetTypeChecker(ctx)` checkout shared across the whole batch. Two effects: N round trips collapse into one frame, and every call in the batch sees the same checker instance (which also fixes the cross-checker-identity hazard mentioned above). Plausibly a small upstream PR.
- **Server-side topology query, e.g. `describeSubtree(root, fields[])`.** Server walks the AST internally and returns selected metadata for asked-for node kinds in a single frame. One frame per pass over a file, instead of one frame per (node(s), query) pair. Especially useful for the bulk-resolve substrategy, which today has to reconstruct this client-side via per-method array overloads. Larger in scope.

## Other directions that came up

- *Fork typescript-go entirely*. Maintain a long-running fork with whatever API surface the transpiler needs exposed, rebasing against upstream on a regular cadence. The TS team mentions forking in [#481](https://github.com/microsoft/typescript-go/discussions/481) as something one could do.
- *WASM via WASI as a subprocess*. Run a WASM build of `tsgo` under a WASI runtime and talk to it over IPC, instead of running a native `tsgo` process directly.
- *Custom-AST IPC*. Feed externally-parsed ASTs into the IPC API via `AcquireDocument`, instead of having `tsgo` parse the source itself. Aimed at tools with their own parser ([oxc](https://github.com/oxc-project/oxc), [biome](https://github.com/biomejs/biome)).

## Open threads

### Working on

- [`dunder`](https://github.com/RealColdFry/dunder): building out the bulk-resolve-with-IR substrategy for Option 1.
- [`tslua`](https://github.com/RealColdFry/tslua): running production-shaped numbers for the in-process-Go-as-WASM path.

### Watching upstream

- [Discussion #481, "Public Go API for embedding?"](https://github.com/microsoft/typescript-go/discussions/481): public Go API would turn shims into normal imports.
- [Issue #516, "Transformer plugin / compiler API"](https://github.com/microsoft/typescript-go/issues/516): transformer/plugin story.
- [Issue #3610, "Additional Checker methods for transpiler consumers"](https://github.com/microsoft/typescript-go/issues/3610): would unblock more of Option 1.

### Planned

- **API-gap tracking.** What consumer-side IPC API users need vs. what `tsgo` exposes today.
- **IPC call-strategy benchmarks.** Capture an instrumented call log from the existing JS `tsc` (every `ts.*` Compiler API and checker call, with rough call-graph topology), then replay it under simulated cost models for different IPC strategies: naive in-line, bulk-resolve, speculative prefetch, memoization layers, async branching, etc. Goal is to project perf bounds for the substrategies in [Inside Option 1](#inside-option-1-ipc-substrategies) before building any of them out. Not sure how realistic the cost model needs to be, whether the captured call log generalizes across codebase shapes, and whether the JS `tsc` call shape is even a good proxy for what `tsgo` IPC consumers will actually do.
