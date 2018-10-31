# Build system integration notes (17.10.2018)

## `build.rs`
Two prime examples:
* Codegen
* Native deps

We need `build.rs` to be more transparent - at least file inputs/outputs.

The codegen case is simple (well-defined transformation from inputs to outputs), one could imagine adding:
```toml
[codegen.bindgen]
inputs = [
	"in_file1", # ...
]
outputs = [
	"out_file1", # ...
]
```

Native deps are trickier - need a way to declaratively specify "I depend on OpenSSL" somehow.
Idea:
* invoke `build.rs` with "build me an OpenSSL"
* delegates to a build backend actually responsible for fetching an artifact

## Build backend
Split Cargo into
```
                       resolution/download
Cargo.toml -> [Cargo] ----------------------> abstract build plan -> [Build backend]
```
Would this mean that `cargo build` would shell out to a respective backend as part of its routine (e.g. call `cargo-buckend --plan=file.json`)?

Backends would be responsible for translating the rules to the ones the build executor understands (in this model `build.rs` would be an implementation detail for Cargo build executor).

### Other systems
#### Bazel
How is it doing it? Rust support seems to be lacking (Rust code not officially approved for the Google monorepo?)
However Google Cloud uses Bazel - ppl should be using Rust there?
#### GN
How Fuchsia folk integrate Rust?

[`rustc_library`](https://fuchsia.googlesource.com/build/+/master/rust/rustc_library.gni)
and
[`rustc_binary`](https://fuchsia.googlesource.com/build/+/master/rust/rustc_binary.gni)
rules (shared
[`rustc_artifact`](https://fuchsia.googlesource.com/build/+/master/rust/rustc_artifact.gni)
rules similar to Buck/Bazel)

### Resolution
Cargo resolves optional dependencies, concrete package versions and feature set as part of its resolution.

How does it affect integration (especially vendoring)?

Translating and commiting rules _a priori_ for the external build system presumably makes the maintenance easier but makes some Cargo-specific workflow harder.

External build system rules correspond to a Cargo post-resolution (lockfile) state.

How would `cargo update -p <pkg>` work? Many obstacles:
* We'd need to understand external rules
* synthesize an internal Cargo lockfile
* How to get package *constraints*? (for all we know all dep constraints could be of concrete `=1.2.3` format)
* Retranslate and clobber existing build system rules?
