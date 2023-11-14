# lib/rust

Earthly's official collection of rust [UDCs](https://docs.earthly.dev/docs/guides/udc).

First, import the UDC up in your Earthfile:
```earthfile
IMPORT github.com/earthly/lib/rust:<version/commit> AS rust
```

## +INIT

This UDC stores the configuration required by the other UDCs in the build environment filesystem, and installs required dependencies.

It is meant to be called once per build environment, to avoid passing repetitive arguments to the following UDCs called after it, and to install required dependencies before the source files are copied from the build context.  

### Usage

Call once per build environment:
```earthfile
DO rust+INIT ...
```

### Arguments
#### `cache_id`
Overrides default ID of the global `$CARGO_HOME` cache. Its value is exported to the build environment under the entry: `$CARGO_HOME_CACHE_ID`.

#### `keep_fingerprints (false)`
Instructs the following `+CARGO` calls to don't remove the Cargo fingerprints of the source packages. Use only when source packages have been COPYed with `--keep-ts `option.
Cargo caches compilations of packages in `target` folder based on their last modification timestamps.
By default, this UDC removes the fingerprints of the packages found in the source code, to force their recompilation and work even when the Earthly `COPY` commands used overwrote the timestamps.

#### `sweep_days (4)`
`+CARGO` calls use cargo-sweep to clean build artifacts that haven't been accessed for this number of days.

## +CARGO

This UDC runs the cargo command `cargo $args` caching the contents of `$CARGO_HOME/registry`, `$CARGO_HOME/git` and `target` for future builds of the same calling target. 

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
This UDC should is thread safe. Parallel builds of targets calling this UDC should be free of race conditions.

## Examples:

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
  # Call +INIT before copying the source file to avoid installing depencies every time source code changes
  DO rust+INIT

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
