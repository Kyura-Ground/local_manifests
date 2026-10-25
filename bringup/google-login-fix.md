# Google login bitmap crash (2026-10-26)

Google Play services and GSF are installed. The login UI repeatedly crashes in
com.google.android.gms.ui / PreAddAccountActivity before normal sign-in can
continue. The stack is Bitmap.nativeWriteToParcel -> PromptInfo.writeToParcel
-> BiometricPrompt.authenticate, throwing:

```
java.lang.RuntimeException: Could not copy bitmap to parcel blob.
```

Rooted ADB and a bounded strace restricted to shared-memory operations confirm:

```
memfd_create("bitmap", MFD_CLOEXEC|MFD_ALLOW_SEALING) = 171
fcntl(171, F_ADD_SEALS, F_SEAL_SHRINK|F_SEAL_GROW|0x10) = -1 EINVAL
```

0x10 is F_SEAL_FUTURE_WRITE. The running ASUS 4.4 kernel has no implementation.
Axion's Bitmap.cpp uses it for immutable bitmaps when bitmap_use_memfd is enabled;
aflags reports that flag enabled by the default read-only configuration. An immediate-disable attempt was rejected,
and the flag remained default-enabled. No Google app data, accounts or credentials
were cleared, no images were flashed, and no device reboot was requested.

Two exact upstream patches were cherry-picked with local GPG signatures:

- [0f93721aa700](https://github.com/xiaomi-msm8998-devs/android_kernel_xiaomi_msm8998/commit/0f93721aa700175dfe3c6a5d5787a685cf23e71f)
  -> 4459685740657d48b63ebb60237ec8377f66169a adds the 4.4 seal implementation.
- [05d351102dbe](https://github.com/torvalds/linux/commit/05d351102dbe4e103d6bdac18b1122cd3cd04925)
  -> f58902fb4299738b5928d9b5254ef94b3b28ed98 fixes the original backport's
  MAP_PRIVATE COW behavior. It clears VM_MAYWRITE only on shared mappings;
  private mappings retain COW at fork.

Both patch IDs match upstream; author identities/dates, full messages,
Change-Id and Signed-off-by trailers are preserved. The original feature is
[Linux ab3948f58ff8](https://github.com/torvalds/linux/commit/ab3948f58ff841e51feb845720624665ef5b7ef3).
The 4.4 implementation also blocks write and hole-punch after sealing. It
preserves already existing writable mappings. HUGETLBFS is disabled in this
device configuration; these changes do not claim hugetlb sealing coverage.

Source head: f58902fb4299738b5928d9b5254ef94b3b28ed98. Previous hotspot/cpuset
head e572bc741676 is its ancestor. Rebuild bootimage on the server after syncing
the new kernel source; a build started from the older head lacks these two fixes.
No local compiler, ROM build or CI was invoked. Google login success, bitmap
parcelling and shared-memory semantics still require device validation with the
rebuilt kernel. Raw logs and syscall traces remain local; the audit records
hashes and bounded diagnostic facts without account credentials.

Primary framework source:
[Axion Bitmap.cpp](https://github.com/AxionAOSP/android_frameworks_base/blob/lineage-23.2/libs/hwui/jni/Bitmap.cpp).
