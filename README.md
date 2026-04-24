# Getting Started
To initialize your device trees setup, use command:

```bash
git clone -b main https://github.com/Kyura-Ground/local_manifests.git .repo/local_manifests --depth=1
```

Then sync with ROM source or sync standalone with this command:
```bash
repo sync -c --force-sync --optimized-fetch --no-tags --no-clone-bundle --prune -j$(nproc --all)
```

**Credit:**
- [Crave](https://foss.crave.io)
- [sounddrill31](https://github.com/sounddrill31)
