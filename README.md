# CroissAndro AOSP manifest file

```sh
repo init --partial-clone -b main -u https://github.com/croissandro/manifest
```

```sh
repo sync -c -j8
```

```sh
source build/envsetup.sh
```

```sh
lunch croissandro_x86_64_linux-trunk_staging-userdebug
```

```sh
m -j48
```
