# CroissAndro manifests

This repository is the entry point for CroissAndro source setup. It contains a
versioned super manifest for both the AOSP and ACK/Kleaf workspaces:

| File | Upstream workspace | CroissAndro projects added |
|---|---|---|
| `default.xml` | AOSP + ACK + CroissAndro | Complete source setup from one `repo init` |
`default.xml` is the recommended team and CI entry point. It composes the
complete upstream AOSP and ACK manifests through `repo` submanifests and adds
the CroissAndro projects directly.

## Repository family

| Repository | Responsibility |
|---|---|
| [`manifest`](https://github.com/croissandro/manifest) | Adds CroissAndro projects to the AOSP and ACK workspaces — this repository |
| [`croissandro`](https://github.com/croissandro/croissandro) | Android product, board and device policy |
| [`kernel-build`](https://github.com/croissandro/kernel-build) | Hyper-V kernel configuration and Kleaf build logic |
| [`kernel-prebuilts`](https://github.com/croissandro/kernel-prebuilts) | Reviewed kernel artifacts consumed by AOSP |

The upstream `aosp`, `kernel`, and `cuttlefish` checkouts are external and are
not modified by this repository.

## Workspace layout

```text
croissandro/
├── manifest/                   this repository
├── aosp/                       AOSP repo workspace
│   ├── device/croissandro/     Android device repository
│   └── prebuilts/kernel/
│       └── croissandro/6.18/   kernel-prebuilts repository
├── kernel/                     ACK repo workspace
│   ├── build/kernel/           upstream Kleaf
│   ├── common/                 upstream ACK source
│   └── croissandro/            kernel-build repository
└── manifest/                   this repository
```

The kernel workspace owns source and build tools. The AOSP workspace consumes
reviewed binaries. Normal Android builds must not depend directly on
`kernel/out`.

## Prerequisites

- supported 64-bit Linux host;
- Git, Python 3, and Android `repo` 2.4 or newer;
- enough storage for separate AOSP and ACK checkouts and their outputs.

See the official [AOSP setup guide](https://source.android.com/docs/setup/start)
and [kernel build guide](https://source.android.com/docs/setup/build/building-kernels)
for complete host requirements.

## Initialize the CroissAndro super workspace

The super manifest places the AOSP and ACK checkouts below `aosp/` and
`kernel/`, allowing one `repo init` entry point to produce the complete layout.

```sh
mkdir -p "$HOME/repos/croissandro/source"
cd "$HOME/repos/croissandro/source"

repo init --partial-clone \
  -b main \
  -u https://github.com/croissandro/manifest

repo sync -c -j8
```

Build and validate the current PI-0 API-37 product:

```sh
cd aosp
source build/envsetup.sh
lunch croissandro_hyperv_x86_64-trunk_staging-userdebug
device/croissandro/tools/validate-pi0.sh
m
device/croissandro/tools/validate-pi0.sh --artifacts
```

PI-0 produces only:

```text
out/target/product/croissandro_hyperv_x86_64/system.img
```

## Initialize an ACK kernel workspace separately

The current bring-up uses `common-android-mainline`. It is exploratory; select
and pin a compatible ACK release branch before publishing kernel prebuilts.

```sh
mkdir -p "$HOME/repos/croissandro/kernel"
cd "$HOME/repos/croissandro/kernel"

repo init \
  -b common-android-mainline \
  -u https://android.googlesource.com/kernel/manifest

repo sync -c -j8
```

Validate and build the Hyper-V kernel policy:

```sh
cd "$HOME/repos/croissandro/kernel/croissandro"
./check-config.sh
DIST_DIR="$HOME/repos/croissandro/kernel-out" ./build.sh
```

The AOSP overlay currently consumes prebuilts under a `6.18` path. Do not
publish mainline 7.x artifacts there. Align the ACK branch, publication path,
and Android device contract together before PI-1.

## Verify the super workspace

AOSP workspace:

```sh
repo list | grep -E 'device/croissandro|prebuilts/kernel/croissandro'
test -f device/croissandro/AndroidProducts.mk
```

Kernel workspace:

```sh
repo list | grep 'croissandro'
test -f croissandro/BUILD.bazel
test -x tools/bazel
```

## Update an existing super workspace

```sh
cd "$HOME/repos/croissandro/source"
repo sync -c -j8
```

Existing standalone AOSP or ACK workspaces should be migrated to the super
workspace and initialized with the command above.

Do not edit `.repo/manifests` or generated `.repo/manifest.xml`; `repo` owns
those files. Pin release manifests to reviewed commits or tags instead of
following mutable `main`.

## Troubleshooting

- Missing CroissAndro project: verify that the workspace was initialized from
  this repository's `default.xml`, then rerun `repo sync`.
- Missing kernel toolchain project: sync the named path from the ACK workspace,
  for example `repo sync -c prebuilts/clang/host/linux-x86`.
- Interrupted sync: rerun the same command and reduce `-j8` if necessary.
- One broken project: inspect and resync that path rather than deleting the
  entire workspace.
