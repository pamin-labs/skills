# Performance

Rules for code on a request's path (a service's hot path, named in its `AGENTS.md`), in any language. Read `core.md` first: its "Parse once", "Hot paths keep their database round trips flat", bounded caches and state, "Calls to other services stay out of transactions" and "Measure before and after" rules apply too and are not repeated here. `rust.md`, `go.md` and `zig.md` say how their code meets these.

- **Hot paths allocate once.** No allocation or cloning inside loops: reuse buffers sized from what is known, never build an intermediate collection only to iterate it once, and never format a value only compared or written out.
- **No repeated parsing** on a hot path: a value is parsed once and its parsed form passed on.
- **Reuse existing optimised kernels** (the standard library's or a dependency's search, hashing or SIMD routines) rather than writing your own loop.
- **Where the runtime schedules async tasks, blocking work runs on a thread meant for it.**
- **Static dispatch on hot paths;** dynamic dispatch only at boundaries crossed once per request or less.
- **Connections:** a database pool's size is set explicitly.
- **Start-up** does not wait on other services: connecting, fetching keys, listing tools and loading what can wait happen on first use or in the background, unless the program can answer nothing without them.
- **Benchmarks** for a claim about production run at the deployed instance's size.
