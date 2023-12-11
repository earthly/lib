# lib/gradle

Earthly's official collection of Gradle [functions](https://docs.earthly.dev/docs/guides/functions).

First, import the library up in your Earthfile:
```earthfile
VERSION --global-cache 0.7
IMPORT github.com/earthly/lib/gradle:<version/commit> AS gradle
```
> :warning: Due to [this issue](https://github.com/earthly/earthly/issues/3490), make sure to enable `--global-cache` in the calling Earthfile, as shown above.

## +INIT

This function stores the configuration required by the other functions in the build environment filesystem, and installs required dependencies.

It must be called once per build environment, to avoid passing repetitive arguments to the functions called after it, and to install required dependencies before the source files are copied from the build context.

### Usage

Call once per build environment:
```earthfile
DO gradle+INIT ...
```

### Arguments
#### `cache_id`
Overrides default ID of the global `$GRADLE_USER_HOME` cache. Its value is exported to the build environment under the entry: `$EARTHLY_GRADLE_HOME_CACHE_ID`.

## +RUN_WITH_CACHE

`+RUN_WITH_CACHE` runs the passed command with the GRADLE caches mounted.

Notice that in order to run this function, [+INIT](#init) must be called first. This function exports the target cache mount ID under the env entry: `$EARTHLY_GRADLE_PROJECT_CACHE_ID`.

### Arguments
#### `command (required)`
Command to run, can be any expression.

#### `gradle_home_cache_id`
ID of the gradle home cache mount. By default: `$EARTHLY_GRADLE_HOME_CACHE_ID` as exported by `+INIT`

#### `project_cache_id`
ID of the target cache mount. By default: `${EARTHLY_GRADLE_HOME_CACHE_ID}#${EARTHLY_TARGET_NAME}`

## Example
See [earthly-intellij-plugin](https://github.com/earthly/earthly-intellij-plugin/blob/main/Earthfile) for a complete example.