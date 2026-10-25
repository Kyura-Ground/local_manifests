# X00TD AxionOS Android 16 — Linux 4.4

This manifest is for the Linux **4.4** bringup. It is separate from the existing
`lineage-23.2` local-manifest branch that names Linux 4.19 trees.

The bringup is still being reviewed. Publish all referenced `axion-23.2-4.4`
source branches before using this manifest on a build server.

```bash
repo init -u https://github.com/AxionAOSP/android.git -b lineage-23.2 --git-lfs
git clone --depth=1 -b axion-23.2-4.4 https://github.com/Kyura-Ground/local_manifests.git .repo/local_manifests
repo sync -c --force-sync --optimized-fetch --no-tags --no-clone-bundle --prune -j8
```

The five ASUS projects use the 4.4 repositories and `axion-23.2-4.4` branches.
The shared signing-key project stays on the existing `lineage-24.0` branch.
The requested ROM variant is vanilla without GApps.

Reference: [AxionOS A16 manifest](https://github.com/AxionAOSP/android/tree/lineage-23.2).
