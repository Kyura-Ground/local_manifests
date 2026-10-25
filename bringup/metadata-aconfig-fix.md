# A16 boot-animation reboot: missing metadata for aconfig

Kernel #5 passed SELinux and networking BPF loading, then Android's
system_server repeatedly aborted with:

```
Missing permission definition for permission "android.permission.RANGING" associated with app op 151
```

Earlier log messages identify the cause: new aconfig storage cannot load
`android.permission.flags`. `/metadata` does not exist. The post-fs aconfig
initialization exits with status 6. After zygote dies four times before boot
completion, init restarts with `zygote-fatal`. The final init SIGABRT is a
consequence, not the original permission error.

## Upstream reference and adaptation

LuK1337's
[d7b10b07e338e0768fa5c52df27e0b253a018ac9](https://github.com/Tomoms/android_device_essential_mata/commit/d7b10b07e338e0768fa5c52df27e0b253a018ac9)
adds a faux metadata mount for aconfig. It was cherry-picked as
`20c2f74b5b55fe9d03d805190b7220aa27fa7d58`, mapping BoardConfig.mk to
BoardConfigCommon.mk and init.mata.rc to init.target.rc. The same four inserted
lines are retained: BOARD_USES_METADATA_PARTITION=true and an on-fs tmpfs mount
after mount_all. Author, author date, original subject/message and Change-Id
are unchanged. The adapted commit has a valid local GPG signature.

The board flag ensures a metadata mount point is generated in the system root.
The on-fs action mounts tmpfs before aconfig's post-fs initialization. Axion's
init builtin map executes `mount` in the init context, including when requested
by this vendor rc; no additional vendor_init SELinux allowance is needed.

## Device validation without rebuilding

The equivalent installed-image hotfix creates `/metadata` (0755 root:root,
metadata_file) and adds only the reviewed mount line to vendor init.target.rc.
The original rc was backed up, its mode/owner/context preserved, the exact new
contents read back, and both filesystems remounted read-only. Boot, recovery,
kernel, partition layout, userdata and encryption settings were not changed.
The device was rebooted once after these changes.

The user reports reaching setup/home. ADB independently observed:

```
sys.boot_completed=1
init.svc.zygote=running
init.svc.bootanim=stopped
uptime=47.83 seconds
```

This establishes initial Android boot past the RANGING crash. After USB was
reconnected and authorized, read-only shell captures confirmed the 10 MiB
metadata tmpfs, populated aconfig maps/boot directories, and registered RANGING
permission. Crash logcat was read successfully and empty. The captured logs
contain no repeat of the previous system_server fatal error. Protected metadata
root listing and dmesg were denied to shell; they were not treated as boot
failures or successful captures. Hardware and repeat-boot checks remain pending.
`metadata-fix-audit.json` records the checks.

The 10 MiB tmpfs is volatile and regenerated each boot. This legacy-device
workaround does not establish persistence for flag overrides or other services
using metadata. No existing storage partition is repurposed or formatted.

Reflashing an older ZIP removes this installed-image hotfix. Future packages
must include both the metadata root directory and the updated vendor rc, or
be rebuilt from the updated common device branch. A new kernel build is not
needed for this device-configuration change.
