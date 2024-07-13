> :fire: Checkout our [blog post](https://earthly.dev/blog/incremental-rust-builds/) about this library!

# lib/rust

Earthly's official collection of Rust [functions](https://docs.earthly.dev/docs/guides/functions).

First, import the library up in your Earthfile:
```earthfile
VERSION 0.8
IMPORT github.com/earthly/lib/rust:<version/commit> AS rust
```

## +INIT

This function sets some configuration in the environment (used by following functions), and installs required dependencies.
It must be called once per build environment, to avoid passing repetitive arguments to the functions called after it, and to install required dependencies before the source files are copied from the build context.
Note that this function changes `$CARGO_HOME` in the calling environment to point to a cache mount later on. 
It is recommended then that all interaction with cargo is done throug the `+CARGO` function or using cache mounts returned by `+SET_CACHE_MOUNTS_ENV`.

### Usage

Call once per build environment:
```earthfile
DO rust+INIT ...
```

### Arguments
#### `cache_prefix`
Overrides cache prefix for cache IDS. Its value is exported to the build environment under the entry: `$EARTHLY_CACHE_PREFIX`. 
By default `${EARTHLY_TARGET_PROJECT_NO_TAG}#${OS_RELEASE}#earthly-cargo-cache`

#### `keep_fingerprints (false)`

By default `+CARGO` removes the [compiler fingerprints](https://doc.rust-lang.org/nightly/nightly-rustc/cargo/core/compiler/fingerprint/struct.Fingerprint.html) of those packages found in your source code (not their dependencies), to force their recompilation and work even when the Earthly `COPY` commands overwrote file mtimes (by default).

Set `keep_fingerprints=true` to keep the source packages fingerprints and avoid their recompilation, when source packages have been copied with `--keep-ts `option.

#### `sweep_days (4)`
`+CARGO` calls use cargo-sweep to clean build artifacts that haven't been accessed for this number of days.

## +CARGO

This function runs the cargo command `cargo $args` caching the contents of `$CARGO_HOME` and `target` for future builds of the same calling target. See #mount-caches-and-parallelization below for more details.

Notice that in order to run this function, [+INIT](#init) must be called first.

### Usage

After calling `+INIT`, use it to wrap cargo commands:

```earthfile
DO rust+CARGO ...
```
### Arguments

#### `args`
Cargo subcommand and its arguments. Required.

#### `output`
Regex to match the files within the target folder to be copied from the cache to the caller filesystem (image layers).

Use this argument when you want to `SAVE ARTIFACT` from the target folder (mounted cache), always trying to minimize the total size of the copied fileset.

For example `--output="release/[^\./]+"` would keep all the files in `/target/release` that don't have any extension.

### Thread safety
This function is thread safe. Parallel builds of targets calling this function should be free of race conditions.

## +SET_CACHE_MOUNTS_ENV

Sets the following entries in the environment, to be used to mount the cargo caches.
 - `EARTHLY_RUST_CARGO_HOME_CACHE`: Code of the mount cache for the cargo home.
 - `EARTHLY_RUST_TARGET_CACHE`: Code of the mount cache for the target folder.

Notice that in order to run this function, [+INIT](#init) must be called first.

### Arguments

#### `target_cache_suffix` 
Optional cache suffix for the target folder cache ID.

### Example

```earthfile
clean-target:
  ...
  DO rust+SET_CACHE_MOUNTS_ENV
  RUN --mount=$EARTHLY_RUST_TARGET_CACHE rm -rf target
```

## +COPY_OUTPUT
This function copies files out of the target cache into the image layers.
Use it function when you want to `SAVE ARTIFACT` from the target folder (mounted cache), always trying to minimize the total size of the copied fileset.

Notice that in order to run this function, `+SET_CACHE_MOUNTS_ENV` or `+CARGO` must be called first.

### Arguments
#### `output` 
Regex matching output artifacts files to be copied to `./target` folder in the caller filesystem (image layers).

### Example
```earthfile
DO rust+SET_RUST_CACHE_MOUNTS
RUN --mount=$EARTHLY_RUST_CARGO_HOME_CACHE --mount=$EARTHLY_RUST_TARGET_CACHE cargo build --release
DO rust+COPY_OUTPUT --output="release/[^\./]+" # Keep all the files in /target/release that don't have any extension.
```
## +CROSS 

Runs the [cross](https://github.com/cross-rs/cross) command: `cross $args --target $target` .

Notice that:
- This function makes use of `WITH DOCKER`, and hence parallelization might be tricky to achieve ([earthly#3808](https://github.com/earthly/earthly/issues/3808)). 
- In order to run this function, [+INIT](#init) must be called first.

### Arguments

#### `target`
Cross [target](https://github.com/cross-rs/cross?tab=readme-ov-file#supported-targets). Required.

#### `args`
Cross subcommand and its arguments. By default: `build --release`

#### `output`
Regex matching output artifacts files to be copied to `./target` folder in the caller filesystem (image layers). By default: `$target/release/[^\./]+`

### Example

```earthfile
cross:
  ...
  DO rust+SET_CACHE_MOUNTS_ENV
  DO rust+CROSS --target aarch64-unknown-linux-gnu
  DO rust+COPY_OUTPUT --output="release/[^\./]+" # Keep all the files in /target/release that don't have any extension.
```

## Complete example

Suppose the following project:
```
.
├── Cargo.lock
├── Cargo.toml
├── deny.toml
├── Earthfile
├── package1
│   ├── Cargo.toml
│   └── src
│       └── ...
└── package2
    ├── Cargo.toml
    └── src
        └── ...
```

The Earthfile would look like:

```earthfile
VERSION 0.8

# Imports the library definition from default branch (in a real case, specify version or commit to guarantee immutability)
IMPORT github.com/earthly/lib/rust AS rust

install:
  FROM rust:1.73.0-bookworm
  RUN apt-get update -qq
  RUN apt-get install --no-install-recommends -qq autoconf autotools-dev libtool-bin clang cmake bsdmainutils
  RUN cargo install --locked cargo-deny
  RUN rustup component add clippy
  RUN rustup component add rustfmt
  # Call +INIT before copying the source file to avoid installing depencies every time source code changes. 
  # This parametrization will be used in future calls to functions of the library
  DO rust+INIT --keep_fingerprints=true

source:
  FROM +install
  COPY --keep-ts Cargo.toml Cargo.lock ./
  COPY --keep-ts deny.toml ./
  COPY --keep-ts --dir package1 package2  ./

# build builds with the Cargo release profile
build:
  FROM +source
  DO rust+CARGO --args="build --release" --output="release/[^/\.]+"
  SAVE ARTIFACT ./target/release/ target AS LOCAL artifact/target

# test executes all unit and integration tests via Cargo
test:
  FROM +source
  DO rust+CARGO --args="test"

# fmt checks whether Rust code is formatted according to style guidelines
fmt:
  FROM +source
  DO rust+CARGO --args="fmt --check"

# lint runs cargo clippy on the source code
lint:
  FROM +source
  DO rust+CARGO --args="clippy --all-features --all-targets -- -D warnings"

# check-dependencies lints our dependencies via `cargo deny`
check-dependencies:
  FROM +source
  DO rust+CARGO --args="deny --all-features check --deny warnings bans license sources"

# cross performs cross compilation
cross:
  FROM +source
  ARG --required target
  DO rust+CROSS --target=$target
  SAVE ARTIFACT target/$target AS LOCAL dist/$target
    
# all runs all other targets in parallel
all:
  BUILD +lint
  BUILD +build
  BUILD +test
  BUILD +fmt
  BUILD +check-dependencies
```

## Mount caches and parallelization

As of today, local Cargo builds cannot run in parallel for a given project, given that the output artifact folder is globally locked for the whole build.
This library overcomes such limitation by using different mount caches for the target folder, one per Earthly target.
While multiple concurrent builds of the same Earthly target will still block, the user now has the choice of creating new Earthly targets instead of reusing the same to increase parallelization.  

Hence, this library uses several mount caches per tuple of `{project, os_release}`:
- One cache mount for `$CARGO_HOME`, shared across all target builds without any locking involved. 
- A family of locked cache mounts for `$CARGO_TARGET_DIR`. One per Earthly target. 

Notice that:
- the previous targets builds might belong to one or multiple Earthly builds, given that the caches involved are global.
- builds will only be blocked by concurrent ones of the same target.
- Earthly target builds are atomic in the sense that the artifacts returned are guaranteed to be the ones generated by that build.

For example, running `earthly +all` in the previous example will:
- run all targets (`+lint,+build,+test,+fmt,+check-dependencies`) in parallel without any blocking involved.
- use a common cache mount for `$CARGO_HOME`.
- use one individual `$CARGO_TARGET_DIR` cache mount per Earthly target.
