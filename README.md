# X00TD AxionOS Android 16 — Linux 4.4

This manifest is for the Linux **4.4** bringup. It is separate from the existing
`lineage-23.2` local-manifest branch that names Linux 4.19 trees.

The user reports compilation succeeded before the current source review.
That review found allocator and BPF cgroup issues, fixed them, and completed
the Axion device metadata and Kernel Manager setup. These new commits require
a server rebuild; boot and hardware behavior have not been validated here.
See the [detailed review](bringup/opencode-review.md) for findings and limits.

```bash
repo init -u https://github.com/AxionAOSP/android.git -b lineage-23.2 --git-lfs
git clone --depth=1 -b axion-23.2-4.4 https://github.com/Kyura-Ground/local_manifests.git .repo/local_manifests
repo sync -c --force-sync --optimized-fetch --no-tags --no-clone-bundle --prune -j8
```

The five ASUS projects use the 4.4 repositories and `axion-23.2-4.4` branches.
`device/axion/common` is overridden to the `axion-23.2-4.4` fork with the
legacy-um sysfs sepolicy fix. The shared signing-key project stays on the
existing `lineage-24.0` branch. The requested ROM variant is vanilla without
GApps.

The [bringup records](bringup/README.md) contain upstream commit mappings,
ASUS source hashes and the scope of static validation.

Reference: [AxionOS A16 manifest](https://github.com/AxionAOSP/android/tree/lineage-23.2).

The current A16 flash failure (updater signal 6) and recovery init panic are
caused by missing kernel MADV_WIPEONFORK. Source fix is published; the existing
ZIP requires a rebuild. See [diagnosis](bringup/wipeonfork-fix.md).
