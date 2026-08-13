# CroissAndro AOSP manifest

This repository is a local manifest for building CroissAndro on top of the
Android Open Source Project (AOSP). It adds the CroissAndro device project at
`device/croissandro` and provides the `croissandro_x86_64_linux-trunk_staging`
build target.

The setup below follows Google's [AOSP development setup guide](https://source.android.com/docs/setup/start).
For background, Git manages individual source repositories, `repo` coordinates
the many Git repositories in an AOSP checkout, and the manifest XML files map
those repositories into the source tree.

## Prerequisites

### Hardware requirements

- 64-bit x86 Linux workstation
- At least 64 GB of RAM
- At least 400 GB of free disk space:
  - Approximately 250 GB for the source checkout
  - Approximately 150 GB for the build output
- A first build can take several hours

### Software requirements

- 64-bit Linux distribution with glibc 2.17 or later
- Git
- Python 3
- OpenJDK
- Make
- Android `repo` version 2.4 or newer

The latest AOSP release branch provides prebuilt OpenJDK, Make, and Python 3,
so they do not need to be installed separately.

On Ubuntu 18.04 or later, install the AOSP host packages:

```sh
sudo apt-get update
sudo apt-get install git-core gnupg flex bison build-essential zip curl \
  zlib1g-dev libc6-dev-i386 x11proto-core-dev libx11-dev lib32z1-dev \
  libgl1-mesa-dev libxml2-utils xsltproc unzip fontconfig
```

Install the `repo` launcher from the package manager if it is available:

```sh
sudo apt-get install repo
repo version
```

The launcher should report version 2.4 or newer. If your distribution does not
provide a recent launcher, install it manually and verify its signature:

```sh
repo_tmp="$(mktemp /tmp/repo.XXXXXXXXX)"
curl -o "$repo_tmp" https://storage.googleapis.com/git-repo-downloads/repo
gpg --recv-keys 8BB9AD793E8E6153AF0F9A4416530D5E920F5C65
curl -s https://storage.googleapis.com/git-repo-downloads/repo.asc \
  | gpg --verify - "$repo_tmp"
mkdir -p "$HOME/bin"
install -m 755 "$repo_tmp" "$HOME/bin/repo"
export PATH="$HOME/bin:$PATH"
repo version
```

For authenticated syncing or quota issues, see the AOSP guide to
[fixing sync quota barriers](https://source.android.com/docs/setup/download/troubleshoot-sync#fix-quota-barriers).

## Get started

Choose a checkout directory. The commands below use `~/repos/croissandro/aosp`;
you can substitute any absolute path.

```sh
mkdir -p "$HOME/repos/croissandro/aosp"
cd "$HOME/repos/croissandro/aosp"
```

When prompted by `repo`, enter the name and email address you use for Git. Set
them before initializing the checkout:

```sh
git config --global user.name "Your Name"
git config --global user.email "you@example.com"
```

Initialize the AOSP source checkout:

```sh
repo init --partial-clone \
  -b android-latest-release \
  -u https://android.googlesource.com/platform/manifest
```

Add this local manifest to the AOSP checkout. Clone or download this repository
first if it is not already present at `~/repos/croissandro/manifest`:

```sh
mkdir -p $HOME/repos/croissandro/aosp/.repo/local_manifests
cp "$HOME/repos/croissandro/manifest/manifest.xml" \
  $HOME/repos/croissandro/aosp/.repo/local_manifests/croissandro.xml
```

Sync the source tree:

```sh
repo sync -c -j8
```

If the sync is interrupted, run the same command again. For a slower or
resource-constrained host, use fewer jobs. On a faster host, increase the
number carefully because parallel syncs use more memory and network bandwidth.

## Configure and build

Run these commands from the AOSP checkout root:

```sh
cd "$HOME/repos/croissandro/aosp"
source build/envsetup.sh
lunch croissandro_hyperv_x86_64-trunk_staging-userdebug
m
```

The `userdebug` variant is intended for development and debugging. Build
outputs are written under `out/`, with the precise locations depending on the
AOSP branch and build system configuration.

The first build is expected to take hours; subsequent incremental builds are
usually much faster.

## Verify the manifest

After syncing, confirm that the CroissAndro project is present:

```sh
repo list | rg 'device/croissandro'
test -f device/croissandro/AndroidProducts.mk
```

To see the available lunch targets, initialize the build environment and run:

```sh
source build/envsetup.sh
lunch
```

## Troubleshooting

- `repo: command not found`: add the directory containing `repo` to `PATH` and
  start a new shell, or run `export PATH="$HOME/bin:$PATH"` again.
- Package installation fails: confirm that the host is a supported 64-bit
  Linux distribution and that the Ubuntu package list matches your release.
- Sync fails with quota or authentication errors: configure Git credentials and
  follow the AOSP [quota-barrier guidance](https://source.android.com/docs/setup/download/troubleshoot-sync#fix-quota-barriers).
- The CroissAndro target is missing: make sure
  `.repo/local_manifests/croissandro.xml` exists, then rerun `repo sync`.
- A partial sync leaves the checkout inconsistent: rerun
  `repo sync -c -j8`; if the error names a specific project, inspect that
  project before retrying rather than deleting the whole checkout.
- The build runs out of memory: reduce the parallelism, for example
  `m -j8`, and ensure sufficient swap is available.

## Updating

Pull changes to this manifest repository, copy the updated `manifest.xml` into
`.repo/local_manifests/croissandro.xml`, and sync again:

```sh
cp "$HOME/repos/croissandro/manifest/manifest.xml" \
  .repo/local_manifests/croissandro.xml
repo sync -c -j8
```

The local manifest is intentionally kept separate from AOSP's upstream
manifest, so it can be updated without modifying files under `.repo/manifests`.
