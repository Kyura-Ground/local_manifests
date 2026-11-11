# Experimental X00TD AOSPA/CLO setup

Use a separate build-server checkout for the non-NFC X00TD with kernel
4.19. The original `AOSPA` and `Infinity` branches are independent.

```bash
repo init -u https://github.com/AOSPA/manifest -b calcite
git clone -b AOSPA-CLO https://github.com/Kyura-Ground/local_manifests .repo/local_manifests
repo sync -c --no-tags -j4
./rom-build.sh X00TD
```

The manifest uses native AOSPA/CLO Qualcomm dependencies. Its legacy HALs
and vendor policy are older official AOSPA branches adapted by the device
configuration. The local audio branch retains the X00TD speaker fixes.

Existing ASUS blobs, imported legacy Bluetooth interfaces, the basic USB
service and the dummy perf client remain compatibility components. This
does not remove the original LineageOS ancestry of the device and kernel
trees. The signing-key repository path and revision retain their existing
names to preserve signing identity.

Full Android 17 builds, policy compilation, partition sizing and hardware
boot tests are still required. Do not treat host configuration checks as
evidence that an image is ready to flash.

**Credit:**
- [Crave](https://foss.crave.io)
- [sounddrill31](https://github.com/sounddrill31)
