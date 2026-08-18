# AGENTS.md — Automated Build Guide: Lavender 4.4 CAF Kernel (nv-kernel + KernelSU/SukiSU)

Dokumen ini berisi instruksi eksekusi untuk Agen AI dan GitHub Actions Runner untuk mengompilasi kernel Linux 4.4 CAF Redmi Note 7 (`lavender`) dengan nama identitas **`nv-kernel`** dan dukungan **KernelSU (SukiSU)**.

---

## 0. ATURAN WAJIB: Selalu Cek Last Action Terlebih Dahulu

**Sebelum memicu build baru atau melakukan perubahan apapun, agen WAJIB memeriksa status GitHub Actions terakhir:**

```bash
# 1. Lihat run terakhir
gh run list --repo ryansadjoh/NV-kernel --limit 3

# 2. Jika sedang berjalan (in_progress) → PANTAU, jangan trigger build baru:
gh run watch <RUN_ID> --repo ryansadjoh/NV-kernel --exit-status

# 3. Jika gagal (failure) → baca log kegagalan SEBELUM mengubah apapun:
gh run view <RUN_ID> --repo ryansadjoh/NV-kernel --log-failed

# 4. Jika sukses (success) → verifikasi artifact sebelum memicu build ulang
```

Catatan penting:
- Run yang macet di satu step >15 menit (mis. `Install Dependencies` >15m) → cancel dan trigger ulang (`gh run cancel <ID>`).
- **Jangan pernah trigger build paralel** — satu build kira-kira 18–25 menit, dan hasil artifact akan tertimpa.

---

## 1. Naming & Discovery Specifications

* **Kernel Name Branding (artifact/zip):** `nv-kernel`
* **Kernel Banner Spoof (uname -r / /proc/version):** gaya stock Samsung Galaxy S9/S9+ — `4.4.232-19761338 (release@21BBA028)` — identitas custom kernel & jejak GitHub Actions sengaja disembunyikan
* **Target Device:** Xiaomi Redmi Note 7 (`lavender`)
* **Kernel Base:** Linux 4.4 CAF (Code Aurora Forum)
* **Output Artifact:** `nv-kernel-Lavender-4.4-CAF-SukiSU-SuFS.zip`
* **Defconfig Target:** `lavender-perf_defconfig`

### Mekanisme spoof Samsung (WAJIB dipertahankan)
1. `CONFIG_LOCALVERSION="-19761338"` — versi 8-digit gaya firmware Samsung S9
2. `export KBUILD_BUILD_USER=release` + `KBUILD_BUILD_HOST=21BBA028` — builder gaya Samsung build farm (menghapus `runner@...`)
3. `sed -i 's/-$(date "+%m-%d")//' scripts/setlocalversion` — repo Yasir MEMODIFIKASI setlocalversion (baris 173) menambah suffix tanggal `-MM-DD`; sed ini menghapusnya. Tanpa ini versi bocor sebagai custom build (ada `-08-18`).
4. `touch .scmversion` — mencegah suffix git tambahan
5. Yang masih terlihat: compiler string `Proton clang 13.0.0` (opsional untuk disamarkan; hampir tidak pernah dicek app)

## 2. Source Paths (WAJIB — digunakan oleh workflow saat ini)

| Komponen | Path / URL | Branch |
|---|---|---|
| **Kernel Source (aktif)** | `https://github.com/Yasir-siddiqui/android_kernel_xiaomi_lavender.git` | `master-2` (default branch; auto-search pilih ini karena bintang terbanyak) |
| Kernel Source (fallback manual) | `https://github.com/paimon-project/android_kernel_xiaomi_lavender.git` | `4.4-caf` / `caf` |
| Kernel Source (fallback manual) | `https://github.com/qawdf/android_kernel_xiaomi_lavender.git` | `4.4` |
| **KernelSU 4.4 fork (WAJIB)** | `https://github.com/gmh5225/KernelSU-4.4.git` | `master` |
| Toolchain | `https://github.com/kdrag0n/proton-clang.git` | `master` (clang-13 + lld) |
| AnyKernel3 | `https://github.com/osm0sis/AnyKernel3.git` | `master` |
| Repo Workflow | `https://github.com/ryansadjoh/NV-kernel` | `master` |

PENTING — KernelSU:
- **JANGAN pakai `tiann/KernelSU` branch `main`** — hanya mendukung kernel 4.14+, setup.sh-nya TIDAK mengintegrasikan source di 4.4 dan `CONFIG_KSU=y` akan dibuang diam-diam.
- Gunakan fork `gmh5225/KernelSU-4.4` (adaptasi khusus kernel 4.4).
- Fork ini punya bug sintaks: `core_hook.c` baris 530 (`return -ENOSYS` kurang titik koma) — workflow sudah mem-patch-nya via `sed`. Jangan hapus baris patch itu.

---

## 3. Configuration & Injection (sebagaimana dijalankan workflow)

```bash
cd source

# 1. Integrasi KernelSU-4.4 (bukan tiann/KernelSU main!)
git clone --depth=1 https://github.com/gmh5225/KernelSU-4.4 KernelSU
ln -sf ../KernelSU/kernel drivers/kernelsu
printf '\nobj-$(CONFIG_KSU) += kernelsu/\n' >> drivers/Makefile
printf '\nsource "drivers/kernelsu/Kconfig"\n' >> drivers/Kconfig
sed -i 's/return -ENOSYS$/return -ENOSYS;/' KernelSU/kernel/core_hook.c

# 2. Patch Defconfig (Nama Kernel + KernelSU)
DEFCONFIG_FILE=arch/arm64/configs/lavender-perf_defconfig

cat <<EOT>> "${DEFCONFIG_FILE}"

# Kernel Branding Name
CONFIG_LOCALVERSION="-nv-kernel"

# SukiSU / KernelSU Flags
CONFIG_KSU=y

# OverlayFS Support (untuk module KernelSU)
CONFIG_OVERLAY_FS=y
CONFIG_TMPFS_XATTR=y
CONFIG_TMPFS_POSIX_ACL=y

# Namespace & KPROBES Options
CONFIG_NAMESPACES=y
CONFIG_USER_NS=y
CONFIG_KPROBES=y
CONFIG_HAVE_KPROBES=y
CONFIG_KPROBE_EVENTS=y
EOT
```

Catatan: `CONFIG_SUFS` TIDAK ADA di Kconfig kernel mana pun — jangan ditambahkan (hanya menimbulkan noise warning).

---

## 4. Verifikasi Hasil Build (WAJIB setelah sukses)

KernelSU benar-benar ter-compile jika log compile memuat baris seperti:
```
CC      drivers/kernelsu/core_hook.o
CC      drivers/kernelsu/ksu.o
CC      drivers/kernelsu/allowlist.o
```

Verifikasi biner (opsional, lokal): dekompresi `Image.gz-dtb` lalu cari string `KernelSU` / `allowlist` — harus ditemukan.

Packaging AnyKernel3 harus memakai `anykernel.sh` lavender (`BLOCK=/dev/block/bootdevice/by-name/boot`, `IS_SLOT_DEVICE=0`, `device.name1=lavender`) — workflow menimpa file default osm0sis dengan config lavender.
