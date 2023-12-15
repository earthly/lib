# lib/gradle

Earthly's official collection of Gradle [functions](https://docs.earthly.dev/docs/guides/functions).

First, import the library up in your Earthfile:
```earthfile
VERSION --global-cache 0.7
IMPORT github.com/earthly/lib/gradle:<version/commit> AS gradle
```
> :warning: Due to [this issue](https://github.com/earthly/earthly/issues/3490), make sure to enable `--global-cache` in the calling Earthfile, as shown above.

## +GRADLE_GET_MOUNT_CACHE

This function sets the following entries in the calling environment, so they can be used later to parametrize two global mount caches in `RUN` commands:
- `$EARTHLY_GRADLE_USER_HOME_CACHE`: Code of the mount cache for the [gradle user home](https://docs.gradle.org/current/userguide/directory_layout.html#dir:gradle_user_home) directory.
- `$EARTHLY_GRADLE_PROJECT_CACHE`: Code of the mount cache for the [gradle project root](https://docs.gradle.org/current/userguide/directory_layout.html#dir:project_root) directory.

### Arguments
- `cache_prefix`: To be used in both caches. By default: `${EARTHLY_TARGET_PROJECT_NO_TAG}#${OS_RELEASE}#earthly-gradle-cache`

### Example
```earthly
DO gradle+GRADLE_GET_MOUNT_CACHE
RUN --mount=$EARTHLY_GRADLE_USER_HOME_CACHE --mount=$EARTHLY_GRADLE_PROJECT_CACHE gradle --no-daemon build
```

### Scoping

If `cache_prefix` is not set, the two global caches have an Earthfile-scope, that is, they are not shared with other Earthfiles.
Otherwise, all Earthfiles using the same `cache_prefix` will share the cache mounts.

## Example
See [earthly-intellij-plugin](https://github.com/earthly/earthly-intellij-plugin/blob/main/Earthfile) for a complete example.