# Library rules

Rules for the standalone libraries in `helix-bedrock` (`nbt`, `leveldb`,
`worldgen`, `pathfinding`, `wasm-host` and any added later). They exist so that
other projects can use them too, so they are held to a stricter bar than engine
internals.

## Independence

1. **A library never imports the engine** (`helix`) or `helix-api`. The engine
   depends on libraries, never the other way round.
2. Libraries may depend on each other only when it is unavoidable (for example
   `leveldb` using `nbt`). No dependency cycles.
3. **Minimal third-party dependencies.** Standard library first. Each new
   dependency needs approval from 2 core maintainers, and must be maintained,
   license-compatible and pure Go (no cgo).

## API

1. Small, documented public API. Unexport everything you can.
2. Every exported identifier has a doc comment, and each package has a
   `doc.go` with an overview and an example.
3. Accept interfaces, return concrete types. Use `io.Reader` and `io.Writer`
   for streams.
4. No global state, `init()` side effects or package-level configuration.
   Configuration goes through options structs or functional options.
5. No logging unless the caller passes a `*slog.Logger`. Never print to
   stdout or stderr.
6. Return errors, never panic on bad input. Define sentinel errors or error
   types callers can check with `errors.Is` / `errors.As`.
7. Safe for concurrent use, or clearly documented as not safe.
8. Stay at `v0.x` until the API has been reviewed by 2 core maintainers. After
   `v1.0.0`, breaking changes require a new major version (`/v2`).

## Performance

1. Hot paths must not allocate in the steady state. Provide `Append`-style or
   buffer-reusing variants (`AppendEncode(dst []byte, ...)`) next to
   convenience functions.
2. Benchmarks with `b.ReportAllocs()` for every hot path. PRs that touch one
   include before and after numbers (use `benchstat`).
3. No reflection in hot paths.

## Testing

1. Unit tests for every exported function, run with `-race`.
2. **Fuzz tests for every decoder of untrusted input** (`nbt`, `leveldb`,
   anything reading network or disk data). Fuzz seeds live in
   `testdata/fuzz/`.
3. Golden-file tests against real vanilla data (BDS worlds, real packets) where
   possible, stored in `testdata/`.
4. Aim for at least 80% coverage; decoders aim for 90%.

## Licensing

1. MIT by default, so anyone can use the library.
2. **If the library contains code ported from a GPL or LGPL project, it must
   be GPL-3.0.** Decide this before porting, not after.
3. Every ported piece is recorded in `CREDITS.md` (see
   [`PORTING.md`](PORTING.md)).

## Go versions

Support the latest two Go releases. Set `go.mod` to the older of the two. CI
tests both.
