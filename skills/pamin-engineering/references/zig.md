# Zig

How Zig code meets `core.md` and `performance.md`; read those first. Like the rest of this skill, it changes only in `pamin-labs/skills`.

## Toolchain and build

- **Version.** The Zig release is named once, as `minimum_zig_version` in `build.zig.zon`, and CI installs exactly it; moving to a new release (std APIs change between them) is its own change. Code is `zig fmt`-clean (`zig fmt --check .` in the gates).
- **Dependencies** are declared in `build.zig.zon` with a pinned `url` (a tag or commit, never a branch) and its `hash`; `zig fetch --save` writes both. Vendored C (a decoder, a header-only library) is fetched the same way and compiled by `build.zig` with the narrowest flags it needs; it is never copied into the tree. A new dependency's added clean build time and release binary size are measured and recorded in the PR.
- **Build modes.** Tests and local runs use Debug or ReleaseSafe. A shipped binary is ReleaseSafe unless a measurement on the product's entry point shows ReleaseFast or ReleaseSmall worth losing the safety checks; the choice and the numbers are in the PR and in `build.zig`, beside the target that uses it. A WebAssembly or size-bound target may default to ReleaseSmall on the same terms.
- **`build.zig` is the one build entry.** Steps (`test`, `bench`, `install`, each cross-compiled target) are named there; scripts and CI call `zig build <step>`, never the compiler with their own flags. Shared values (version, targets, output names) are defined once in `build.zig` or `build.zig.zon` and passed to code through `b.addOptions`.
- **Published packages** keep semver: a breaking change to a public declaration, an error set or a CLI or JSON output is called out in the PR and released as one.

## Structure: layers, domains and reuse

core.md's "Shape of the code" and "Layers, domains and decoupling" apply in full; this is how Zig code meets them.

- **A module per concern, a directory per layer.** Each `.zig` file is a module named for what it holds (`decode/png.zig`, `diff/alignment.zig`); a directory groups one layer or concern, and the repository's `AGENTS.md` names each directory and what may import it. A root module (`root.zig`) re-exports the public surface; everything else is reached through it, never by path from outside the package.
- **Layers point one way.** Entry (`main.zig`, the CLI, the WASM exports) → orchestration → domain logic → values; integrations (a C library, the file system, a decoder) sit behind the module that owns them. A test in `zig build test` walks each file's `@import`s and fails on one that points back up or skips the owner; a new directory updates that test's list in the same PR.
- **Entry points only translate.** The CLI and the WASM exports parse arguments or bytes into typed values, call one function, and shape its result as output; a rule (a threshold, a limit, a refusal) found there is a bug.
- **Interfaces are declared by their consumer** with only what it calls: a comptime parameter checked by `@hasDecl` for static dispatch, or a small struct of function pointers at a boundary crossed rarely. Functions return concrete types.
- **One rule, one place.** Every constant, threshold, limit, format version and default is declared once, in the module that owns it, and imported elsewhere; the build passes values the code and the build share through `b.addOptions`. A second copy of a loop, a pixel conversion or a parsing step is a bug: extend the owner's function instead.
- **Reuse before writing.** Climb core's ladder in Zig terms: the repository's own module, `std` (`std.mem`, `std.sort`, `std.hash`, `std.crypto`, `std.json`, `std.simd`), a package already in `build.zig.zon`, then a maintained one; the PR's "Reused" section says which.
- **Small functions with one job,** a public surface as small as the callers need (no `pub` by default), and no dead code: `zig build` compiles only what is reached, so an unreferenced function's errors stay hidden; a test or a caller references every `pub` declaration (`std.testing.refAllDecls` in the root module's test).

## Names and files

The [Zig Style Guide](https://ziglang.org/documentation/master/#Style-Guide) applies; these are its rules review checks.

- **Case.** Types and functions returning `type` are `TitleCase`; other functions `camelCase`; variables, fields, constants and namespace structs (no fields) `snake_case`. Acronyms are words (`PngDecoder`, `HttpClient`), established names (`ENOENT`) keep theirs. A file with top-level fields is `TitleCase.zig`, any other file and every directory `snake_case`.
- **No redundant names.** A name reads in its namespace: `png.Decoder`, not `png.PngDecoder`; no `Data`, `Value`, `Context`, `Manager`, `State`, `utils` or `misc` that would fit any type (core.md's module naming).
- **No underscore prefixes.** Zig has no private fields; a field is named for what it holds and its doc comment says how to use it. What is private is a declaration without `pub`.
- **Doc comments.** `//!` opens each file with what it holds; `///` on each `pub` declaration says what the name does not, and says who owns returned memory ("Caller owns the returned slice"). "Asserts" names an invariant checked in safe modes, "assumes" one that is not.
- **File order** follows core's: imports (`const std = @import("std");` first), then types, then constants, then `pub` functions before private ones, helpers in rough call order, `test` blocks last.

## Memory and resources

- **Allocators are passed in.** A function that allocates takes an `std.mem.Allocator` parameter (or holds one given at `init`); no global allocator in library code. The binary picks one in `main`: an arena for a run's short-lived work, `std.heap.smp_allocator` or `c_allocator` in release builds, `DebugAllocator` in Debug.
- **Ownership is explicit.** Every allocation has one owner and is freed by it: `defer` right after a successful acquire, `errdefer` for what a function returns on success but must release on error. A type that owns memory has `deinit` and its doc says so.
- **Allocate once.** Buffers are sized from what is known (`initCapacity`, `ensureTotalCapacity`) and reused across iterations (`clearRetainingCapacity`); no allocation inside a per-pixel, per-row or per-request loop; no list built only to iterate it once. Large inputs are streamed or memory-mapped rather than copied when the format allows.
- **Bounded.** Every buffer, queue and cache has a bound named once as a constant or option, and the code says what happens at it (refuse with an error, evict).

## Errors and safety

- **Explicit error sets at boundaries.** A public function returns a named error set (`DecodeError!Image`), merged with `||` where layers meet, so a caller can switch over every case; inferred `!T` only for private helpers. `anyerror` only where a callback or interface requires it. Errors travel with `try`; context the caller needs (which file, which pixel) goes in a diagnostics value the caller passes, since Zig errors carry no payload.
- **No illegal behaviour reachable from input.** `unreachable`, `catch unreachable`, `orelse unreachable`, `.?` on a value input decides, `@intCast`, `@truncate` or an index that input can push out of range, and overflowing `+ - *` stay off every path a file, argument or another program's output reaches. Check and return an error, or use `std.math.cast`, saturating (`+|`) or wrapping (`+%`) arithmetic where that is the meaning. They are allowed in tests and where an invariant rules the failure out, with a comment stating it.
- **`@setRuntimeSafety(false)`, `@optimizeFor`, `@ptrCast`, `@alignCast` and inline assembly** only where a measured kernel needs them, in the smallest scope, with a comment naming each invariant relied on and a test that exercises it.
- **C interop.** Every C call's result is checked; C memory is freed with its own allocator's free; a C library's error codes are mapped to the Zig error set at the binding, which is the only file that imports it (core's one integration module per service).

## Types

- **Types over integers and strings.** Ids, units and amounts are distinct types (`enum(u32) { _ }` for an id, a struct for a size or a colour), never a bare `u32` or `[]const u8` passed between modules; a closed set is an `enum`, matched with an exhaustive `switch` (no `else` where every case is known). Input is parsed into these at the boundary.
- **Typed JSON.** Output is serialised from structs with `std.json.Stringify` (or the release's equivalent), input parsed into structs that ignore unknown fields; JSON is never built with `print` or string concatenation.
- **comptime** removes repetition and fixes sizes known at build time; it is not used to build a second type system. Generic code (`anytype`, a function returning a type) states the interface it needs in a doc comment and fails with a clear `@compileError` when a type lacks it.

## Performance mechanisms

- **SIMD** through `@Vector` and `std.simd`, with a scalar path for the tail; the kernel's speed-up is measured against the scalar version on the product's inputs and recorded in the PR.
- **Static dispatch** through comptime parameters; an interface of function pointers (`std.mem.Allocator`, `std.Io.Writer`) only at boundaries crossed once per call or less.
- **I/O** is buffered and flushed once per output; output goes through `std.Io` writers, never one write per value.
- **Threads.** Work split across threads uses `std.Thread.Pool` (or the release's equivalent) sized from the CPU count and a cap, joins every thread it starts, and shares no mutable state without one owner.

## Tests

- **Beside the code.** `test` blocks sit at the end of the file they test; cross-module tests go in `tests/` and run from `zig build test`. Tests use `std.testing.allocator`, so a leak fails the run, and `std.testing.FailingAllocator` (`checkAllAllocationFailures`) for code whose allocation failure path matters.
- **Deterministic and independent:** no sleeps, wall clock or network; anything random uses a fixed seed; fixtures are committed small files. A fix comes with a test that fails without it.
- **Parsers and decoders are fuzzed** (`std.testing.fuzz`, run by `zig build test --fuzz`), and a crash it finds becomes a fixture.
