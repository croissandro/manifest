# CroissAndro AOSP manifest file

```sh
repo init --partial-clone -b main -u https://github.com/croissandro/manifest
```

```sh
repo sync -c -j8
```

```sh
export BASE_DIR="$HOME/repos/croissandro/manifest/out/aosp-tmp"
export JAVA_TOOL_OPTIONS="-Djava.io.tmpdir=$BASE_DIR -Dorg.sqlite.tmpdir=$BASE_DIR"
export TMPDIR="$BASE_DIR"
```

```sh
source build/envsetup.sh
```

```sh
lunch croissandro_x86_64_linux-trunk_staging-userdebug
```

```sh
m BUILD_BROKEN_NINJA_USES_ENV_VARS=JAVA_TOOL_OPTIONS -j48
```
