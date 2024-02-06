> :fire: Checkout our [blog post](https://earthly.dev/blog/incremental-rust-builds/) about this library!

# lib/rust

Earthly's official collection of Rust [functions](https://docs.earthly.dev/docs/guides/functions).

First, import the library up in your Earthfile:
```earthfile
VERSION 0.8
IMPORT github.com/earthly/lib/rust:<version/commit> AS rust
```
> *Due to [this issue](https://github.com/earthly/earthly/issues/3490), make sure to enable `--global-cache` in the calling Earthfile, as shown above.*

## +INIT 

INIT sets some entries in the calling environment (to be used by rest of the functions later on), in particular the ones related to mounting the cargo caches:
- `EARTHLY_RUST_CARGO_HOME_CACHE`: Definition of the mount cache for the cargo home.
- `EARTHLY_RUST_TARGET_CACHE`: Definition of the mount cache for the target folder.

It must be called once per build environment.

It is recommended that all interaction with Cargo is done with the previous caches mounted.

### Usage

Call once per build environment:
```earthfile
DO rust+INIT
```

### Arguments
#### `cache_prefix`
Sets the prefix to be used in the IDs of the two mount caches. By default: `${EARTHLY_TARGET_PROJECT_NO_TAG}#${OS_RELEASE}#earthly-cargo-cache#${EARTHLY_GIT_BRANCH}`
Its value is exported in the build environment as: `$EARTHLY_CARGO_CACHE_PREFIX`. 

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

### Example
```earthfile
release:
  FROM ...
  DO rust+CARGO --args="build --release"  --output="release/[^\./]+" # Keep all the files in /target/release that don't have any extension.
```

## COPY_OUTPUT
This function copies files out of the target cache into the image layers.
Use it when you want to perform `SAVE ARTIFACT` from the target folder (mounted cache), trying to minimize the total size of the copied fileset.
Notice that in order to run this function, `+INIT` must be called first.

### Arguments
#### `output` 
Regex matching output artifacts files to be copied to `./target` folder in the caller filesystem (image layers).

### Example
```earthfile
release:
  DO rust+INIT
  RUN --mount=$EARTHLY_RUST_CARGO_HOME_CACHE --mount=$EARTHLY_RUST_TARGET_CACHE cargo build --release
  DO rust+COPY_OUTPUT --output="release/[^\./]+" # Keep all the files in /target/release that don't have any extension.
```

## SWEEP
This function runs cargo-sweep to clean build artifacts that haven't been accessed for a number of days.
Notice that in order to run this function, `+INIT` must be called first.

### Arguments
#### `days`
Number of days. Default value: 4

### Example
```earthfile
sweep:
  DO rust+INIT
  DO rust+SWEEP --days=10
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
  # Call +INIT before copying the source file to avoid installing dependencies every time source code changes. 
  # This parametrization will be used in future calls to functions of the library
  DO rust+INIT

source:
  FROM +install
  # Always copy with --keep-ts for Cargo to detect changes
  COPY --keep-ts Cargo.toml Cargo.lock ./
  COPY --keep-ts deny.toml ./
  COPY --keep-ts --dir package1 package2  ./
  DO rust+CARGO --args="check"

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
  
# all runs all other targets in parallel
all:
  BUILD +lint
  BUILD +build
  BUILD +test
  BUILD +fmt
  BUILD +check-dependencies
```

## Mount caches and parallelization

This library uses two mount caches per tuple of `{project, branch, os_release}`:
- One cache mount for `$CARGO_HOME`, shared across all target builds without any locking involved.
- One cache mount for `$CARGO_TARGET_DIR`, shared across all target builds without any locking involved.

Notice that:
- the previous targets builds might belong to one or multiple Earthly builds.
- Earthly will perform no locking across the builds. Instead, Cargo locking will be in place, the same way as if those builds were concurrent Cargo processes in a local machine.

For example, running `earthly +all` in the previous example will:
- run all targets (`+lint,+build,+test,+fmt,+check-dependencies`) in parallel without any blocking involved
- use a common cache mount for `$CARGO_HOME`
- use a common cache mount for  `$CARGO_TARGET_DIR`

## Explicitly mounting the caches

In that scenarios where running via `DO rust+CARGO` is not feasible, you can alternative mount the caches as follows:
```earthfile
RUN --mount=$EARTHLY_RUST_CARGO_HOME_CACHE --mount=$EARTHLY_RUST_TARGET_CACHE ...
``` 

### Example

```earthfile
cross:
  ...
  WITH DOCKER
    RUN --mount=$EARTHLY_RUST_CARGO_HOME_CACHE --mount=$EARTHLY_RUST_TARGET_CACHE cross build --target $TARGET --release
  END
```
