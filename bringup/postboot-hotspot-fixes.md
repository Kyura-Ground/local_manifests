# Post-boot hotspot and cpuset follow-ups (2026-10-26)

The running kernel is still build #5. Source fixes below have not been built or
flashed. No local compiler, Android build or CI was invoked.

## Hotspot: dnsmasq cannot be spawned

On the good cold boot, the WLAN firmware is ready. The first two hotspot attempts
create wlan2 successfully and reach AP-ENABLED. netd then reports a broken pipe
when sending DNS forwarder state; tethering tears down the AP. Later attempts
also report ENOSPC while creating an AP interface. That later error alone is not
the initiating failure, and no interface-name override was applied.

A bounded strace of netd and an isolated tether start/status/stop reproduce:

```text
syscall_0x1b4(0x3, 0xffffffff, 0x4, ...) = -1 ENOSYS
exit_group(127)
```

Syscall 0x1b4 is close_range(436). Android 16 bionic's POSIX_SPAWN_CLOEXEC_DEFAULT
calls close_range(3, UINT_MAX, CLOSE_RANGE_CLOEXEC) and exits 127 on failure,
before execve(dnsmasq). netd returns spawn success but has no pipe reader.
The isolated tether service was returned to stopped; no Wi-Fi configuration,
SIM setting, interface name, policy, or image was changed for this diagnosis.

[The exact 4.4 upstream fix](https://github.com/sunplex07/android_kernel_sony_msm8998/commit/e052ad11f068d2d6f223921dd493258942d0ee8a)
was cherry-picked unchanged as e572bc741676e488e16b3bc909f95fc246ec688d,
retaining author, author date and full message, with a local GPG signature.
It registers syscall 436 for ARM64 and compat, rejects invalid flags/ranges,
bounds iteration to the fd table, supports CLOEXEC, and uses the existing
4.4 unshare_files helper. The separate camellia backport found during research
was not used: its close loop lacks the fd-table bound.

Rebuild kernel/boot.img on the server to apply this fix. Updating only boot.img
can test the hotspot fix with the already working installed ROM; keep matching
recovery/kernel images in the final package. A stable AP, client association,
DHCP, DNS and internet, plus repeated enable/disable, still need device testing.
There are also TC-BPF attachment EINVAL errors that require separate review;
this change is not a claim of fully functional tethering.

Primary callers reviewed:
[Axion bionic spawn.cpp](https://github.com/AxionAOSP/android_bionic/blob/lineage-23.2/libc/bionic/spawn.cpp),
[Axion netd TetherController.cpp](https://github.com/AxionAOSP/android_system_netd/blob/lineage-23.2/server/TetherController.cpp).

## cpuset mount and process profiles

Android requests cpuset,noprefix,cpuset_v2_mode. The running 4.4 kernel does not
recognize that option, so /dev/cpuset contains plain directories rather than
controller nodes. Adding SELinux permission for those wrongly labeled paths
would not fix the mount.

Two referenced commits were picked with 4.4 path/context adaptation:

- [51ef29944806](https://github.com/xiaomi-msm8998-devs/android_kernel_xiaomi_msm8998/commit/51ef2994480694c04e836e65a2fc9d88b6e56451)
  -> cf6ff7dd02896eff768678f8f7ebb516a8b675d6 adds the mount flag/parser/display.
- [706301189187](https://github.com/xiaomi-msm8998-devs/android_kernel_xiaomi_msm8998/commit/7063011891876d35bf898abf0a8b358123d25cee)
  -> 3551e1ff17bafbc846e6f75f76da1c1a18b2110d implements validation,
  empty-mask inheritance, attach, bind and hotplug behavior. It is not a no-op
  acceptance of the option. Both preserve upstream author/date/full message.

The kernel rebuild also contains these changes. Validate the mounted cpuset
controller and task-profile application after boot.

## Pinner cleanup

[7c5b1a27f805](https://github.com/crdroidandroid/android_device_xiaomi_sdm660-common/commit/7c5b1a27f8058ca526ad8cd7c488c79bc73c43b7)
-> common e1f6c62c52a92b1bb1efe0248e0616c70195f01b removes only the stale
/system/lib64/libRScpp.so pin entry, adapting the overlay path. Author/date/full
message and the other pin entries are retained. This overlay needs a ROM rebuild;
a kernel-only rebuild cannot apply it.

## Separate unresolved Wi-Fi firmware failure

On the earlier bad boot, ICNSS stayed at 0x180: no WLFW service 0x45, no firmware
ready event and no WLAN interfaces. The user's full power-off/cold boot restored
the QMI service, firmware and Wi-Fi. That intermittent warm-boot problem existed
on A15 and is still unresolved. The close_range fix does not address it.

Other open audit findings: missing AxDragonite external module with an all-built-in
kernel config, absent memory controller, fs-verity ENOTTY, GPU perfd DAC errors,
and tethering TC-BPF attachment failures. Raw logs remain local. See
postboot-audit.json for evidence hashes, checks and limits.
