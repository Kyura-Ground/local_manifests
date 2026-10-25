# X00TD AxionOS A16 source references

Candidate: AxionOS Android 16, vanilla, ASUS Linux 4.4.302-CIP115.
The selected source backports and static checks are complete. A full ROM
compilation has been attempted on ServerHive but currently stops in vendor
sepolicy; assembled-image VINTF checks and device boot/runtime validation
remain untested.

`source-lock.json` records the ASUS source heads and inspected platform manifest.
It is a source snapshot, not a revision-locked manifest for every Android project.
`device-upstream-references.json` records device/common adaptations and source URLs.

`kernel-cherry-picks.jsonl` maps upstream commits to local commits and describes
manual adaptations. The kernel contains 1,531 new, locally signed commits; author,
author date and upstream message bodies were verified. Defconfig device names
were adapted in commit titles. Empty/already implemented changes are journaled
without creating empty commits. The kernel remains based on the ASUS CIP115 tree;
the Xiaomi device configuration and uname-spoof changes were not imported.

The kernel compile passed on ServerHive. The full ROM build then stopped in
vendor sepolicy on the legacy-um `ax_diagnostics` rules: checkpolicy reported
`unknown type vendor_sysfs_battery_supply`. The Axion common sepolicy fork at
`Kyura-Ground/android_device_axion_common` commit `e80a73d4f4af` drops the
vendor-prefixed duplicate rules that only exist on non-legacy QCOM devices.
`local_manifest.xml` points `device/axion/common` at that fork. A rebuild and
`Image.gz-dtb`/ROM zip are still pending; no local build was run.

Kernel reference:
[xiaomi-msm8998-devs/android_kernel_xiaomi_msm8998](https://github.com/xiaomi-msm8998-devs/android_kernel_xiaomi_msm8998/tree/lineage-23.2),
snapshot `15e2687e40a47a7e6bb86dbf746a37c657b2092f`.
Namespace prerequisites came from their original
[Linux commits](https://github.com/torvalds/linux/commits/master/fs/nsfs.c).

`validation.json` records the checks and their limits. Of 54 compared BPF files,
52 match the reference exactly. ARM64 JIT differs only by blank lines; core
allocation differences retain the ASUS HIGHMEM flags and module allocator path
with `MODULES=y` on ARM64. Kconfig was evaluated in Python with external commands
blocked. No compiler, make, .config generation, CI, ADB or device modification
was used for these checks. Workflow tests used mocked build/upload commands.

The BPF property `ro.bpf.kver_override=5.4.299` selects the reviewed backported
feature level through Axion's existing KernelUtils mechanism. The actual kernel
version remains 4.4. HAL FCM is 6, kernel FCM is 5, and VINTF kernel enforcement
is retained. Existing proprietary blobs and signing identity are preserved.

Original upstream `Test:` lines in preserved commit messages describe upstream
testing. They do not mean those tests were run on this port.

The first server build exposed duplicated signing-certificate directories.
The device-level keys.mk inheritance has been removed using upstream commit
`f53b5f8de71d318783183455c1befc0fc3ebdf10`; Axion common.mk already includes it.
`signing-inheritance-fix.json` records a focused evaluation using Axion's actual
product/node macros: the duplicated Bluetooth certificate path becomes one
directory, and the existing testkey identity path is preserved. This evaluation
has no build recipes and invokes no compiler or Soong. Server compilation has
not been rerun by the source audit.
