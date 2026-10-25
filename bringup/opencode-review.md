# Audit OpenCode dan integrasi AxionOS X00TD

Audit ini memeriksa source lokal dan commit publik setelah perbaikan OpenCode.
Pengguna melaporkan kompilasi berhasil sebelum audit ini. Laporan itu bukan hasil
pengujian saya atas image, boot, atau fitur perangkat. Perubahan tambahan di bawah
masih memerlukan build server. Tidak ada kompilasi lokal, CI, flash, atau ADB.

## Temuan yang diperbaiki

### Alokasi F2FS dengan GFP_NOFS

Commit OpenCode `39062a7758fa` memasukkan allocator global `kvmalloc_node` dan
menghapus allocator privat F2FS yang namanya bertabrakan. Ini menyelesaikan error
BTF, tetapi allocator awal dari Linux 4.12 memanggil `WARN_ON_ONCE` ketika flag
tidak mencakup `GFP_KERNEL`, lalu tetap dapat melakukan fallback ke vmalloc.
F2FS X00TD masih meneruskan `GFP_NOFS`, misalnya dari `fs/f2fs/xattr.c` melalui
`f2fs_kzalloc`/`f2fs_kmalloc` ketika alokasi pertama gagal. Sebagian alokasi
internal vmalloc memakai `GFP_KERNEL`, sehingga pembatasan reclaim filesystem
tidak terjamin. Allocator privat lama juga memiliki keterbatasan fallback ini;
memindahkannya ke helper global tidak otomatis menyelesaikan kontraknya.

Fix `83576aa45670` mengambil
[Linux ce91f6ee5b3b](https://github.com/torvalds/linux/commit/ce91f6ee5b3bbbad8caff61b1c46d845c8db19bf).
Flag yang tidak kompatibel langsung memakai `kmalloc_node`, tanpa warning dan
tanpa fallback vmalloc. `GFP_KERNEL` untuk BTF tetap mendapat fallback dan
`__GFP_ZERO` dipertahankan. Hanya hunk `mm/util.c` yang relevan diterapkan;
`lib/bucket_locks.c` tidak ada di tree ini. Tidak ada flag `GFP_NOFS` yang diubah
menjadi `GFP_KERNEL` pada caller filesystem.

### ID cgroup BPF salah pada backport sebelumnya

Ini berasal dari backport saya sebelumnya `abc1e65ac279`, yang mengikuti adaptasi
msm8998, bukan kesalahan baru OpenCode. `bpf_get_current_cgroup_id` mengembalikan
`cgrp->kn->id.id` (ID kernfs 64-bit), tetapi helper ancestor mengembalikan
`ancestor->id` (ID internal hierarchy/IDR). Pada level cgroup saat ini kedua API
harus menghasilkan ID yang sama. IDR tersebut tidak memenuhi kontrak ini.

Fix lokal `4e34303b063b` mengembalikan `ancestor->kn->id.id`. Referensinya adalah
[Linux 0f09abd105da](https://github.com/torvalds/linux/commit/0f09abd105da6c37713d2b253730a86cb45e127a),
yang memakai `cgroup_id(ancestor)`. Tidak ditemukan commit terpisah yang persis
memperbaiki substitusi ID pada backport legacy ini, sehingga koreksi satu baris
dicatat sebagai perubahan lokal, bukan diberi identitas penulis upstream.

### Prasyarat cgroup diterapkan sebagian

OpenCode `a4ea6002fc91` memang hanya mengambil `cgroup_ancestor` dari commit
berjudul penambahan helper skb, dan mencatat batas itu di jurnal. Ini cukup untuk
error kompilasi `helpers.c`, tetapi enum `skb_ancestor_cgroup_id` masih tidak
memiliki implementasi/prototype dispatch di `filter.c`.

Follow-up `59588a8e31a9` menyelesaikan bagian tersebut dari
[Linux 7723628101aa](https://github.com/torvalds/linux/commit/7723628101aaeb1d723786747529b4ea65c5b5c5):
helper skb, prototype dua argumen, dispatch TC, dan dokumentasi UAPI. Body helper
sama persis dengan Linux upstream, termasuk pemeriksaan socket/ancestor dan
ID kernfs. Enum yang sudah ada tidak diubah. Penulis, tanggal penulis, dan isi
pesan upstream dipertahankan. Kconfig mengevaluasi `SOCK_CGROUP_DATA=y`.

## Perbaikan OpenCode yang dipertahankan

| Commit | Hasil review |
| --- | --- |
| `a4ea6002fc91`, `44ece13f6ec3` | Parent walk memakai level cgroup yang sudah ada. Level terlalu dalam menghasilkan NULL; level negatif berakhir di parent NULL. Tidak menambahkan dereference setelah NULL. Akses mengikuti lifetime cgroup caller. Hunk skb yang semula tidak diambil sudah dilengkapi pada follow-up. |
| `636acce9937d` | Wrapper meneruskan size, node, GFP flags, PAGE_KERNEL, dan alamat caller ke allocator yang sudah ada. Deklarasi dan definisi cocok. X00TD ARM64 mengevaluasi MMU=y; ini bukan validasi kernel NOMMU. |
| `a2430a003408` | Perubahan kode mekanis dari SLAB_DESTROY_BY_RCU ke SLAB_TYPESAFE_BY_RCU. Bit tetap 0x00080000UL; RCU/slab behavior tidak diganti. Dokumentasi upstream menjelaskan nama lama. Sisa nama lama hanya komentar/dokumentasi, bukan referensi macro aktif. |
| `39062a7758fa` | Deklarasi/export kvmalloc_node, zeroing helpers, dan penghapusan definisi F2FS duplikat sesuai tujuan. BTF menggunakan GFP_KERNEL dan membebaskan dengan kvfree. Perlu follow-up GFP_NOFS di atas. |
| Axion common `e80a73d4f4af` | Legacy-um mendeklarasikan sysfs_battery_supply/devfreq/graphics/kgsl tanpa prefix vendor. M4 Lineage menambahkan prefix untuk jalur non-legacy. Menghapus aturan duplikat vendor mempertahankan aturan plain yang benar; vendor_sysfs_kgsl_gpuclk tetap ada karena tipe itu memang dideklarasikan legacy. Tidak ada permissive, allow-all, atau penghapusan neverallow. |
| local_manifests `1423340` | Override device/axion/common ke fork berisi fix. Cocok dengan nama project upstream. Source-lock sebelumnya belum mencatat project tambahan ini; laporan ini memperbaruinya. |
| build_script `e1e0a47` | Checkout Axion common lama ikut dibuang oleh workflow persiapan sync yang sudah ada, supaya perubahan remote diterapkan. Syntax Bash dan 10 skenario mock lulus setelah perubahan ini. Ini tidak menjalankan workflow nyata. |

## Kesesuaian README Axion

Referensi:
[README lineage-23.2](https://github.com/AxionAOSP/android/blob/lineage-23.2/README.mkdn),
Git blob `085c32809e4da6a42c25809f5d7a7ca25e83f6e4` saat diperiksa.

Device commit `2acde968f0da` menambahkan informasi X00TD 4 GB yang dikonfirmasi
pengguna: kamera belakang 13+5 MP, depan 8 MP, Snapdragon 636, maintainer ikwfahmi.
Variabel diletakkan sebelum inheritance common Axion. Ini metadata untuk varian
4 GB pengguna; build ini tidak mengklaim metadata kamera varian 6 GB.

| Bagian README | Implementasi / keputusan |
| --- | --- |
| Manifest/envsetup | Axion lineage-23.2 dengan git-lfs; workflow memilih X00TD userdebug va. |
| Device setup | TARGET_DISABLE_EPPE=true sebelum common_full_phone; tetap product lineage_X00TD. |
| About phone | AXION_CAMERA_REAR_INFO=13,5, FRONT=8, PROCESSOR=Snapdragon_636, MAINTAINER=ikwfahmi. |
| Kernel Manager | XML vendor untuk policy0/core0-3 dan policy4/core4-7. GPU KGSL devfreq; daftar frekuensi dibaca dari kernel dalam Hz. Tidak menyalin frekuensi Snapdragon 660/SoC lain. |
| Permissions | Delapan kontrol CPU/GPU mendapat owner system dan mode 0660 saat post-fs-data. Kebijakan CPU ada di Axion system_ext; kebijakan GPU ada di fork QCOM legacy yang ditinjau. Label/akses aktual tetap perlu runtime. |
| Power HAL | TARGET_DISABLES_LIBPERF=false mempertahankan layanan power QTI yang sudah dikonfigurasi. Custom hints opsional belum ditambahkan karena memerlukan tuning dan pengujian hardware. |
| HBM/bypass charging | Tetap false; node HBM dan kontrak bypass charging belum divalidasi. charging_enabled tidak diasumsikan identik dengan bypass. |
| Doze/debug/AxionFx | Opsional; tidak diaktifkan secara spekulatif. Tap-to-wake yang ada tidak otomatis berarti double-tap AOD pulse. Debug SELinux khusus Axion tidak dinyalakan permanen. |
| AxDragonite | Default upstream memang mengaktifkan external module. Sourcenya menyediakan guard versi 4.4; tidak dinonaktifkan hanya untuk melewati build. Dukungan runtime affinity dan load module belum diuji. |
| Signing | Tetap menggunakan keys.mk/identitas pengguna yang sudah ada; tidak menjalankan gk -s karena akan menghasilkan/menulis konfigurasi signing baru. |

Tidak ditemukan integrasi X00TD Axion yang persis untuk di-cherry-pick. Konfigurasi
perangkat ditulis berdasarkan README, contoh lavender, contoh
[mumba cbcb512dd8b2](https://github.com/AxionAOSP-devices/android_device_motorola_mumba/commit/cbcb512dd8b2a51a8b255884ac584cd8070e1a30),
parser XML Axion, CPU DTS dan atribut sysfs kernel X00TD sendiri.

## Validasi dan batasnya

- `opencode-review.json`: 21 pemeriksaan kontrak source/XML/permission lulus.
- `kernel-provenance-audit.json`: 1.533 commit upstream mempertahankan metadata;
  satu koreksi lokal dicatat terpisah; semua 1.534 commit kernel baru memiliki
  tanda tangan valid.
- Kconfig dievaluasi dalam Python, tanpa perintah eksternal, compiler, make,
  atau penulisan .config. MMU/CGROUPS/SOCK_CGROUP_DATA=y; konfigurasi BPF/JIT
  yang sebelumnya diperiksa tetap aktif.
- Perbandingan ulang 54 file BPF: 49 identik dengan referensi msm8998. Lima delta
  ditinjau: blank line JIT, allocator core yang sudah ada, ID ancestor yang
  dikoreksi, implementasi skb yang dilengkapi, dan dokumentasi UAPI terkait.
- AppArmor dan XFRM_USER_COMPAT masih memiliki allocator privat dengan nama
  kvmalloc; keduanya n dalam konfigurasi X00TD. Mengaktifkannya memerlukan
  integrasi lanjutan. WireGuard memakai nama __compat_* dengan macro, bukan
  definisi ulang allocator global.
- Diff baru lulus pemeriksaan whitespace; temuan format upstream lama tidak
  diubah massal. Tidak ada perubahan blob, enkripsi, signing key, atau checkout A15.

Kompilasi yang dilaporkan berhasil tidak membuktikan boot atau fungsi BPF benar.
Setelah mengambil commit baru, rebuild server diperlukan, lalu verifikasi boot,
VINTF, load AxDragonite, kontrol Kernel Manager dalam SELinux enforcing, kamera,
audio, modem, enkripsi, dan Wi-Fi pada cold boot maupun reboot. Property versi
BPF 5.4.299 tidak menjadi bukti semua API Linux 5.4 sudah tersedia.
