# Axion X00TD: updater signal 6 and recovery init panic

Read-only recovery diagnostics identified the immediate error:

```
whole-file signature verified against RSA key 0
Installing update...
arc4random data MADV_WIPEONFORK failed
Error in /sideload/package.zip (killed by signal 6)
```

The previous boot's pstore independently recorded the same fatal with EINVAL,
followed by `Attempted to kill init! exitcode=0x00007f00`. First-stage init
reported recovery mode. The kernel build identity matches both boot.img and
recovery.img in the supplied Axion ZIP; both contain the same kernel SHA256:
`56f1ff96612729ae89f4b04617e1c1428cb3cbe66e8e6146129b56d273343eac`.
The updater executable hash in recovery matches the one extracted from the ZIP.

[Axion bionic arc4random.h](https://github.com/AxionAOSP/android_bionic/blob/lineage-23.2/libc/upstream-openbsd/android/include/arc4random.h),
inspected blob `1940398ba85606f19e022aa5844f51ef3679e33c`, makes WIPEONFORK
mandatory and leaves the former PID-based fork detector empty. This is a kernel
bringup prerequisite that was missed. The existing 4.4 source had no support.
Replacing only update-binary would leave recovery/Android startup affected.

Implemented upstream fixes on axion-23.2-4.4:

| Source | Local | Purpose |
| --- | --- | --- |
| [msm8998 9844c156b8a1](https://github.com/xiaomi-msm8998-devs/android_kernel_xiaomi_msm8998/commit/9844c156b8a1a46ca0d3c474fe1c815f4ef144c3) | `73f0cf362c07` | Real WIPEONFORK/KEEPONFORK UAPI, madvise, fork behavior and smaps wf flag. |
| [Linux 93949bb21b52](https://github.com/torvalds/linux/commit/93949bb21b52aae924fd26f9775c8655dd9e885e) | `630e76d696e7` | Do not prepare the child's anon_vma during fork; initialize lazily on fault. |

The source VMA bit 0x02000000 is still VM_ARCH_2/x86 MPX on this 4.4 tree.
The adapted backport uses the free bit 0x00800000, preserving that existing API.
This allocation was cross-checked against the
[4.4 LGE backport](https://github.com/TheDBP/android_kernel_lge_msm8996/commit/51c01a4c3469be8b468096d8551a595bb1a47238).
No false-success madvise stub or arc4random abort bypass was added. The child
does not inherit the parent's PRNG pages; the original upstream behavior and
the anon_vma correctness follow-up are both included.

Author, author date and message bodies were preserved, and both commits are
locally signed. `wipeonfork-audit.json` records 12 source-contract checks;
the existing source checks and full signature/provenance audit also pass.
No compilation or fork/madvise runtime test was performed locally. Tests on
the rebuilt kernel must cover child/grandchild zeroing, unchanged parent data,
KEEPONFORK, invalid shared/file mappings, partial ranges, and recovery/updater
startup. Source checks do not establish those runtime outcomes.

Build on the server:

```bash
repo sync -c -j2 kernel/asus/sdm660
. build/envsetup.sh
axion X00TD userdebug va
m bacon
```

This is an incremental rebuild; installclean is not required for these edits.
Use the resulting recovery to install the resulting ZIP. The old ZIP embeds
the old kernel and would overwrite a separately flashed new boot image.
For a kernel-only build, `m bootimage recoveryimage` is possible, but both
artifacts must be incorporated into the package/install sequence before use.

The original ZIP was not edited. ADB was used only to read logs, uname/properties
and hashes. No flash, sideload, reboot or device partition write was run.
SSH BatchMode access to the build server was unavailable, so no server build
was started by this session either.
