# lib/rust

Earthly's official collection of rust [UDCs](https://docs.earthly.dev/docs/guides/udc).

## +CARGO

This UDC runs the cargo command `cargo $args` caching the contents of `$CARGO_HOME/registry`, `$CARGO_HOME/git` and `target` for future builds of the same calling target. 

### Usage

First, import the UDC up in your Earthfile:
```earthfile
VERSION 0.7
IMPORT github.com/earthly/lib/rust:<version/commit> AS rust
```

Then, just use it in your own targets and UDCs:
```earthfile
DO rust+CARGO ...
```

### Thread safety
This UDC should be thread safe. Parallel builds of targets using it should be free of race conditions.

### Arguments

#### `args`
Cargo subcommand and its arguments. Required.

#### `keep_fingerprints (false)`
Do not remove source packages fingerprints. Use only when source packages have been `COPY`ed with `--keep-ts` option.

Cargo caches compilations of packages in `target` folder based on their last modification timestamps. 

By default, this UDC removes the fingerprints of the packages found in the source code, to force their recompilation and work even when the Earthly `COPY` commands used overwrote the timestamps.

#### `sweep_days (4)`
The UDC uses [cargo-sweep](https://github.com/holmgr/cargo-sweep) to clean build artifacts that haven't been accessed for this number of days.

#### `output`
Regex to match the files within the target folder to be copied from the cache to the caller filesystem (image layers). 

Use this argument when you want to `SAVE ARTIFACT` from the target folder (mounted cache), always trying to minimize the total size of the copied fileset. 

For example `--output="release/[^\./]+"` would keep all the files in `/target/release` that don't have any extension.

### Examples:

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
VERSION 0.7
ARG --global debian=bookworm

# Importing UDC definition from default branch (in a real case, specify version or commit to guarantee immutability)
IMPORT github.com/earthly/lib/rust AS rust

install:
  FROM rust:1.73.0-$debian
  RUN apt-get update -qq
  RUN apt-get install --no-install-recommends -qq autoconf autotools-dev libtool-bin clang cmake bsdmainutils
  RUN cargo install --locked cargo-deny
  RUN rustup component add clippy
  RUN rustup component add rustfmt

source:
  FROM +install
  COPY --keep-ts Cargo.toml Cargo.lock ./
  COPY --keep-ts deny.toml ./
  COPY --dir package1 package2  ./

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
```
