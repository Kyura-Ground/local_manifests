# X00TD AxionOS A16 source references

Candidate: AxionOS Android 16, vanilla, ASUS Linux 4.4.302-CIP115.
Android initial boot is confirmed with user-supplied kernel #5 and the metadata
hotfix: setup/home, boot_completed=1, registered RANGING permission and an empty
post-boot crash logcat buffer. No local compilation or CI was run. The earlier
allocator/BPF, integration and recovery failures are documented below, along
with the current common-device fix 8668a65da69d. Hardware and repeat-boot tests
remain pending.

Read [opencode-review.md](opencode-review.md) for findings, commit mappings,
README coverage, and limits. `opencode-review.json` records 21 passing source
contract checks. Compilation success does not establish boot or API correctness.

`source-lock.json` records the ASUS source heads and the Axion common fork.
It is a source snapshot, not a revision-locked manifest of the complete platform.
The device metadata is for the user's confirmed 4 GB / 13+5 MP rear / 8 MP front
X00TD variant. The SoC is Snapdragon 636 despite the shared sdm660 platform name.

`kernel-cherry-picks.jsonl` records 1,541 upstream-derived commits and one local
correction (1,542 signed new kernel commits). Upstream author, author date and
message bodies were verified. Prefix/path adaptations and partial picks are
explicitly recorded. Empty/already implemented changes are journaled separately.
The ASUS CIP115 base, existing blobs and signing identity remain unchanged.

Review follow-ups:

- `83576aa45670`: Linux ce91f6ee5b3b, kvmalloc must not use vmalloc fallback
  with GFP_NOFS or other incompatible flags.
- `4e34303b063b`: local correction of the older msm8998-derived ancestor helper,
  returning the same 64-bit kernfs ID format as the current-cgroup helper.
- `59588a8e31a9`: complete Linux 7723628101aa skb ancestor helper and TC dispatch
  omitted by the earlier cgroup header prerequisite pick.
- Device `2acde968f0da`: Axion metadata and Kernel Manager XML/init permissions.

The Axion common fork `e80a73d4f4af` correctly removes duplicate non-legacy
QCOM sysfs type references while keeping the plain legacy types and the explicitly
defined vendor_sysfs_kgsl_gpuclk. It remains selected by local_manifest.xml.

`validation.json` includes current provenance, Kconfig, source comparison,
workflow and follow-up review results. Of 54 compared BPF files, 49 match the
msm8998 reference exactly. Five deliberate deltas are described: JIT blank lines,
retained core allocator behavior, the corrected ancestor ID, completed skb helper,
and its UAPI documentation. Header or source equality is not API/runtime proof.

Kernel reference:
[xiaomi-msm8998-devs/android_kernel_xiaomi_msm8998](https://github.com/xiaomi-msm8998-devs/android_kernel_xiaomi_msm8998/tree/lineage-23.2),
snapshot 15e2687e40a47a7e6bb86dbf746a37c657b2092f.
Original Linux prerequisites and the GFP follow-up are linked in the journal;
`additional-linux-history.json` preserves the latter's canonical metadata.

HAL FCM is 6, kernel FCM is 5, and VINTF kernel enforcement is retained.
ro.bpf.kver_override=5.4.299 uses Axion's existing feature-version mechanism;
it does not prove complete Linux 5.4 API coverage. The actual kernel remains 4.4.

The signing inheritance fix is documented in signing-inheritance-fix.json.
Axion common.mk loads the user's existing keys.mk once; no gk -s or key generation
was used. Optional AxionFx, HBM, bypass charging, doze extensions and custom
performance hints were not enabled without device evidence. Power HAL remains
active and Kernel Manager reads supported clocks from sysfs.

Original upstream Test lines describe upstream testing. New source needs a
server rebuild and device checks for boot, BPF loading, VINTF, Wi-Fi across cold
boot/reboot, encryption, camera/audio/modem and Kernel Manager in SELinux enforcing.

## WIPEONFORK flash/recovery failure follow-up

A16 bionic aborts at arc4random when MADV_WIPEONFORK is unavailable. Read-only
ADB logs confirm updater signal 6 after successful OTA signature verification;
pstore also confirms the A16 recovery init panic. The missing feature is now
backported in kernel commits 73f0cf362c07 and 630e76d696e7.
See [wipeonfork-fix.md](wipeonfork-fix.md) in the published bringup records.
The old ZIP remains unchanged. Rebuild kernel/boot/recovery/ZIP on the server,
then use the rebuilt recovery to install the rebuilt ZIP. No local build or
device image write was performed during that diagnosis. Kernel-only image builds are also possible
but require integrating both new images into the package/install procedure.

## SELinux recovery failure follow-up

The rebuilt kernel #4 passed arc4random but aborted loading the actual A16
recovery policy. The Android M compatibility heuristic mistakes modern NLMSG
xperms for M's older encoding, desynchronizing avtab records. Upstream
34114e4c2536 was cherry-picked unchanged as 51750c3aeac1. The actual policy
contains 100 NLMSG xperm rules; twelve static/evidence checks pass.
See [selinux-recovery-fix.md](selinux-recovery-fix.md) and
`selinux-compat-audit.json`. The user restored the working old recovery.
Kernel #4 and the earlier repack remain failing artifacts; rebuild boot/recovery
on the server, then validate the new images. SELinux remains enforced; this
format fix does not add modern NLMSG xperm enforcement to the old kernel.

## Android initial boot confirmed

Kernel #5 passed SELinux. The next system_server crash was RANGING permission
lookup after aconfig failed without /metadata. LuK1337 d7b10b07e338 was picked
as common device commit 8668a65da69d, adapting the two paths while retaining
author/date/raw message body/Change-Id and the same four added lines. The
subject is adapted to the ASUS sdm660-common tree. The equivalent
installed-image hotfix reached setup/home and ADB boot_completed=1.
See [metadata-aconfig-fix.md](metadata-aconfig-fix.md) and
`metadata-fix-audit.json`. Hardware, repeat boots and persistent metadata
services still need validation; the 10 MiB tmpfs is volatile.

## Post-boot follow-ups, 2026-10-26

Kernel head e572bc741676 includes the reviewed cpuset mode and close_range
backports; common head 84b1432de1ee removes the missing libRScpp pin entry.
These changes need a server rebuild. The running build #5 lacks them.
Hotspot root cause is confirmed by strace: close_range returns ENOSYS and the
netd child exits 127 before dnsmasq starts. Random warm-boot WLAN firmware
absence remains a separate unresolved problem. Read
[postboot-hotspot-fixes.md](postboot-hotspot-fixes.md) for evidence and limits.

## Commit subject cleanup

Five common-device titles and one manifest title were normalized for ASUS
scope and consistent wording. Raw message bodies, author identities/dates,
Change-Id and Signed-off-by trailers are unchanged. No source code changed.
See [commit-subject-cleanup.md](commit-subject-cleanup.md) and its old/new hash
mapping. All 1,540 kernel commits remain unchanged.

## Google login bitmap crash

Kernel head f58902fb4299 adds F_SEAL_FUTURE_WRITE and the upstream MAP_PRIVATE
COW correction. On-device strace confirms Google login fails when the running
#5 kernel rejects immutable bitmap sealing with EINVAL. These commits require
a server bootimage rebuild and runtime validation. See
[google-login-fix.md](google-login-fix.md). No Google app data was cleared.
