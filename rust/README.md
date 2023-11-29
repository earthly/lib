# lib/rust

Earthly's official collection of Rust [functions](https://docs.earthly.dev/docs/guides/functions).

First, import the library up in your Earthfile:
```earthfile
VERSION --global-cache 0.7
IMPORT github.com/earthly/lib/rust:<version/commit> AS rust
```
> :warning: Due to [this issue](https://github.com/earthly/earthly/issues/3490), make sure to enable `--global-cache` in the calling Earthfile, as shown above.

## +INIT

This function stores the configuration required by the other functions in the build environment filesystem, and installs required dependencies.

It must be called once per build environment, to avoid passing repetitive arguments to the functions called after it, and to install required dependencies before the source files are copied from the build context.

### Usage

Call once per build environment:
```earthfile
DO rust+INIT ...
```

### Arguments
#### `cache_id`
Overrides default ID of the global `$CARGO_HOME` cache. Its value is exported to the build environment under the entry: `$EARTHLY_CARGO_HOME_CACHE_ID`.

#### `keep_fingerprints (false)`
Instructs the following `+CARGO` calls to don't remove the Cargo fingerprints of the source packages. Use only when source packages have been COPYed with `--keep-ts `option.
Cargo caches compilations of packages in `target` folder based on their last modification timestamps.
By default, this function removes the fingerprints of the packages found in the source code, to force their recompilation and work even when the Earthly `COPY` commands used overwrote the timestamps.

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

## +RUN_WITH_CACHE

`+RUN_WITH_CACHE` runs the passed command with the CARGO caches mounted.

Notice that in order to run this function, [+INIT](#init) must be called first. This function exports the target cache mount ID under the env entry: `$TARGET_CACHE_ID`.

### Arguments
#### `command (required)`
Command to run, can be any expression.

#### `cargo_home_cache_id`
ID of the cargo home cache mount. By default: `$CARGO_HOME_CACHE_ID` as exported by `+INIT`

#### `target_cache_id`
ID of the target cache mount. By default: `${CARGO_HOME_CACHE_ID}#${EARTHLY_TARGET_NAME}`

### Example
Show `$CARGO_HOME` cached-entries size:

```earthfile
DO rust-udc+RUN_WITH_CACHE --command "du \$CARGO_HOME"
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
VERSION --global-cache 0.7

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
  
# all runs all other targets in parallel
all:
  BUILD +lint
  BUILD +build
  BUILD +test
  BUILD +fmt
  BUILD +check-dependencies
```

## Mount caches and parallelization

This library uses several mount caches per tuple of `{project, os_release}`:
- One cache mount for `$CARGO_HOME`, shared across all target builds without any locking involved. 
- A family of locked cache mounts for `$CARGO_TARGET_DIR`. One per target. 

Notice that:
- the previous targets builds might belong to one or multiple Earthly builds.
- builds will only be blocked by concurrent ones of the same target

For example, running `earthly +all` in the previous example will:
- run all targets (`+lint,+build,+test,+fmt,+check-dependencies`) in parallel without any blocking involved
- use a common cache mount for `$CARGO_HOME`
- use one individual `$CARGO_TARGET_DIR` cache mount per target 