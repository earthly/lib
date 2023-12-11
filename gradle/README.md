# lib/gradle

Earthly's official collection of Gradle [functions](https://docs.earthly.dev/docs/guides/functions).

First, import the library up in your Earthfile:
```earthfile
VERSION --global-cache 0.7
IMPORT github.com/earthly/lib/gradle:<version/commit> AS gradle
```
> :warning: Due to [this issue](https://github.com/earthly/earthly/issues/3490), make sure to enable `--global-cache` in the calling Earthfile, as shown above.

## +GRADLE_GET_MOUNT_CACHE

This function sets the following entries in the calling environment, so they can be used later on `RUN` commands
- `$EARTHLY_GRADLE_USER_HOME_CACHE`: Code of the mount cache for the gradle home.
- `$EARTHLY_GRADLE_PROJECT_CACHE`: Code of the mount cache for the .gradle folder.

### Arguments:
- `cache_prefix`: To be used in both caches. By default: `${EARTHLY_TARGET_PROJECT_NO_TAG}#${OS_RELEASE}#earthly-gradle-cache`

### Example:
```earthly
DO gradle+GRADLE_GET_MOUNT_CACHE
RUN --mount=$EARTHLY_GRADLE_USER_HOME_CACHE --mount=$EARTHLY_GRADLE_PROJECT_CACHE gradle --no-daemon build
```

## Example
See [earthly-intellij-plugin](https://github.com/earthly/earthly-intellij-plugin/blob/main/Earthfile) for a complete example.