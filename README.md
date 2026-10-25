# X00TD AxionOS Android 16 — Linux 4.4

This manifest is for the Linux **4.4** bringup. It is separate from the existing
`lineage-23.2` local-manifest branch that names Linux 4.19 trees.

This is a candidate for the first server build. The selected upstream kernel
backports and static source checks are complete; ROM compilation and device
boot have not been validated. All referenced source branches must be available
on `axion-23.2-4.4` before syncing.

```bash
repo init -u https://github.com/AxionAOSP/android.git -b lineage-23.2 --git-lfs
git clone --depth=1 -b axion-23.2-4.4 https://github.com/Kyura-Ground/local_manifests.git .repo/local_manifests
repo sync -c --force-sync --optimized-fetch --no-tags --no-clone-bundle --prune -j8
```

The five ASUS projects use the 4.4 repositories and `axion-23.2-4.4` branches.
The shared signing-key project stays on the existing `lineage-24.0` branch.
The requested ROM variant is vanilla without GApps.

The [bringup records](bringup/README.md) contain upstream commit mappings,
ASUS source hashes and the scope of static validation.

Reference: [AxionOS A16 manifest](https://github.com/AxionAOSP/android/tree/lineage-23.2).
