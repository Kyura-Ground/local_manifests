# A16 recovery policy-loader failure

The user's kernel #4 recovery reached init SELinux setup, then aborted:

```
init: Opening SELinux policy from monolithic file /sepolicy
SELinux: Android master kernel running Android M policy in compatibility mode.
SELinux: avtab: invalid type or class
init: SELinux: Could not load policy: Invalid argument
init: InitFatalReboot: signal 6
```

This pstore was collected after the user restored a working LOS recovery. It
identifies kernel #4 (Oct 25 17:23:50 UTC), so the new kernel actually started;
the failure is later than the previous MADV_WIPEONFORK / arc4random abort.

The failed recovery's `/sepolicy` SHA256 is
`04523ca7a9f1ee82dcf43e00888bef466a1226573b9ead3bb379c12b9520636a`.
Read-only disassembly with Debian's prebuilt libsepol 3.11 succeeded and found
100 NLMSG extended-permission rules. No policy compilation was performed.

The old Android M compatibility heuristic treats extended-permission types
other than IOCTLFUNCTION (1) and IOCTLDRIVER (2) as the old M encoding. Modern
NLMSG uses type 3. The old-format path consumes one byte in place of the modern
specified + driver pair, desynchronizing the remaining avtab stream. The
misleading invalid type/class error follows this misclassification.

## Upstream fix

[34114e4c2536965966b1d7dd558d4bdeb009d831](https://github.com/xiaomi-msm8998-devs/android_kernel_xiaomi_msm8998/commit/34114e4c2536965966b1d7dd558d4bdeb009d831)
reverts the obsolete Android M compatibility logic. It was cherry-picked as
`51750c3aeac1dbd26941f44f12c6064041242cb7`, preserving upstream author, author date,
message and Change-Id, with a local GPG signature. The resulting avtab.c is
byte-identical to the reference commit's file. Only avtab.c changes.

The companion behavior from Linux
[900f83cf376bdaf798b6f5dcb2eae0c822e908b6](https://github.com/torvalds/linux/commit/900f83cf376bdaf798b6f5dcb2eae0c822e908b6),
ignoring unknown extended permissions rather than BUG(), already exists in the
ASUS CIP115 source and is retained. This fix restores decoding; it does not
implement modern NLMSG xperm enforcement. SELinux enforcement and ordinary
permission checks are retained. No permissive workaround or policy removal is
used. This A16 branch no longer supports the obsolete M xperm encoding.

`selinux-compat-audit.json` records twelve passing source/evidence checks.
These checks establish the failure mechanism and patch provenance, not a
successful policy load or boot with the next kernel image.

## Next device validation

Keep the working recovery active while rebuilding on the server:

```bash
repo sync -c -j2 kernel/asus/sdm660
git -C kernel/asus/sdm660 rev-parse HEAD
. build/envsetup.sh
axion X00TD userdebug va
m bootimage recoveryimage -j192
```

The expected kernel head is `51750c3aeac1dbd26941f44f12c6064041242cb7`.
No clean/full ROM rebuild is required to incorporate this kernel-only change.
Both images must use the corrected kernel. New images still require validation
and integration into the existing signed package before installation.

The current downloaded kernel #4 images and the previously repacked
`axion-2.9-NOESIS-20261025-UNOFFICIAL-VANILLA-X00TD-kernel-fix.zip` contain the
failing policy loader. Do not treat that ZIP as a working recovery/boot fix.
The original full ROM ZIP is unchanged. No additional flashing was performed
during this diagnosis. A recovery start, ADB, updater, ROM boot and hardware
tests remain pending after the next rebuild.
