# Croissandro manifest

This repository wraps the upstream Android Open Source Project (AOSP)
manifests. Upstream files are vendored under `aosp/`; the root `default.xml`
is the Repo entry point for local configuration.

## Vendor and update AOSP

```bash
cd ~/repos/croissandro/manifest
git remote add aosp https://android.googlesource.com/platform/manifest
git fetch aosp master
git subtree add --prefix=aosp aosp master --squash
```

The root `default.xml` should include the vendored manifest:

```xml
<include name="aosp/default.xml"/>
```

Keep product-specific changes in the root manifest using `<extend-project>`,
`<remove-project>`, or additional `<project>` entries. Do not edit files under
`aosp/` directly.

To update the vendored AOSP manifest:

```bash
git fetch aosp master
git subtree pull --prefix=aosp aosp master --squash
git push origin feat/setup
```

Use a pinned AOSP branch or tag instead of `master` for reproducible builds.

## Download the Android source

```bash
cd ~
mkdir -p aosp
cd aosp
repo init --partial-clone -b feat/setup -u https://github.com/croissandro/manifest
repo sync -c -j8
```

The manifest repository must contain `aosp/default.xml` before running Repo.

## Build the code

```bash
source build/envsetup.sh
lunch aosp_cf_x86_64_only_phone-aosp_current-userdebug
m
```

The first build can take several hours. Build artifacts are written to
`OUT_DIR`, normally `out`.

## Troubleshooting

```bash
cd ~/repos/croissandro/aosp
grep -n 'include\|submanifest' .repo/manifests/default.xml
test -f .repo/manifests/aosp/default.xml && echo "AOSP manifest present"
git -C .repo/manifests log -1 --oneline
```

The `/usr/bin/repo` not-writable warning only prevents the launcher from
self-updating; it is unrelated to manifest resolution.
