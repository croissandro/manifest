# CroissAndro manifests

This repository contains the CroissAndro local-manifest overlays for two
independent Android source workspaces:

| File | Upstream workspace | Projects added |
|---|---|---|
| `aosp-manifest.xml` | AOSP platform | Android device/product configuration and published kernel prebuilts |
| `kernel-manifest.xml` | Android Common Kernel (ACK) | CroissAndro Kleaf configuration and build policy |

These files are overlays, not replacements for Google's manifests. Copy each
one into the corresponding checkout's `.repo/local_manifests` directory after
running `repo init` against the official upstream manifest.

The intended layout is:

```text
croissandro/
├── aosp/                       AOSP platform repo workspace
│   ├── device/croissandro/     from the AOSP overlay
│   └── prebuilts/kernel/
│       └── croissandro/6.18/   reviewed kernel artifacts
├── kernel/                     ACK repo workspace
│   ├── build/kernel/
│   ├── common/
│   ├── croissandro/            kernel-build repository
│   └── tools/bazel
└── manifest/                   this repository
```

Keeping AOSP and ACK as separate `repo` workspaces follows Google's build
model: the kernel tree owns sources and build tools, while the Android tree
consumes published kernel binaries.

## Prerequisites

- A supported 64-bit Linux host
- Git and Python 3
- Android `repo` 2.4 or newer
- Sufficient storage for separate AOSP and kernel checkouts and build outputs

For complete host requirements, see the official
[AOSP setup guide](https://source.android.com/docs/setup/start) and
[kernel build guide](https://source.android.com/docs/setup/build/building-kernels).

## AOSP platform workspace

Initialize the platform checkout using Google's current release manifest:

```sh
mkdir -p "$HOME/repos/croissandro/aosp"
cd "$HOME/repos/croissandro/aosp"

repo init --partial-clone \
  -b android-latest-release \
  -u https://android.googlesource.com/platform/manifest
```

Install the CroissAndro AOSP overlay and sync:

```sh
mkdir -p .repo/local_manifests
cp "$HOME/repos/croissandro/manifest/aosp-manifest.xml" \
  .repo/local_manifests/croissandro.xml

repo sync -c -j8
```

Configure and build the API-level 37 development product:

```sh
source build/envsetup.sh
lunch croissandro_hyperv_x86_64-trunk_staging-userdebug
m
```

Verify the projects supplied by the overlay:

```sh
repo list | grep -E 'device/croissandro|prebuilts/kernel/croissandro'
test -f device/croissandro/AndroidProducts.mk
```

## ACK kernel workspace

The kernel uses its own upstream manifest and checkout. Select an ACK branch
that is compatible with the Android product. The current bring-up checkout
uses `common-android-mainline`; pin a release branch and reviewed revisions
before publishing production prebuilts.

The AOSP overlay currently consumes the `kernel-prebuilts` repository under a
`6.18` path. Treat `common-android-mainline` as exploratory until the kernel
branch and that publication namespace are deliberately aligned; do not publish
a mainline 7.x artifact as if it were a 6.18 build.

```sh
mkdir -p "$HOME/repos/croissandro/kernel"
cd "$HOME/repos/croissandro/kernel"

repo init \
  -b common-android-mainline \
  -u https://android.googlesource.com/kernel/manifest
```

Install the kernel overlay and sync:

```sh
mkdir -p .repo/local_manifests
cp "$HOME/repos/croissandro/manifest/kernel-manifest.xml" \
  .repo/local_manifests/croissandro.xml

repo sync -c -j8
```

The overlay places the separate `kernel-build` Git repository at
`kernel/croissandro`, where its Bazel package can use the workspace's Kleaf
rules and ACK sources without patching `common` or `build/kernel`.

Build and validate from that project:

```sh
cd "$HOME/repos/croissandro/kernel/croissandro"
./check-config.sh
./build.sh
```

To choose the distribution directory explicitly:

```sh
DIST_DIR="$HOME/repos/croissandro/kernel-out" ./build.sh
```

After boot and compatibility testing, publish the reviewed artifacts to the
`kernel-prebuilts` repository consumed by the AOSP overlay. Record both the
ACK source revision and the `kernel-build` revision with every publication.

## Updating existing workspaces

After pulling this repository, refresh the appropriate overlay and resync.

AOSP:

```sh
cd "$HOME/repos/croissandro/aosp"
cp ../manifest/aosp-manifest.xml .repo/local_manifests/croissandro.xml
repo sync -c -j8
```

Kernel:

```sh
cd "$HOME/repos/croissandro/kernel"
cp ../manifest/kernel-manifest.xml .repo/local_manifests/croissandro.xml
repo sync -c -j8
```

Do not edit `.repo/manifests` or the generated `.repo/manifest.xml`; `repo`
owns those files. CroissAndro changes remain in this repository and the
projects declared by these local overlays.

## Troubleshooting

- If a CroissAndro project is missing, confirm that the correct overlay exists
  at `.repo/local_manifests/croissandro.xml`, then rerun `repo sync`.
- If kernel build analysis reports a missing toolchain project, sync the named
  path from the kernel workspace. For example:

  ```sh
  repo sync -c prebuilts/clang/host/linux-x86
  ```

- If a sync is interrupted, rerun the same command. Reduce `-j8` on a
  memory- or network-constrained host.
- Do not delete an entire checkout to repair one failed project; inspect and
  resync the specific path first.
