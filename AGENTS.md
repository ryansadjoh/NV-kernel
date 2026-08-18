# AGENTS.md — Automated Build Guide: Lavender 4.4 CAF Kernel (nv-kernel + SukiSU + SuFS)

Dokumen ini berisi instruksi eksekusi untuk Agen AI dan GitHub Actions Runner untuk mengompilasi kernel Linux 4.4 CAF Redmi Note 7 (`lavender`) dengan nama identitas **`nv-kernel`**, mendukung **SukiSU**, serta fitur **SuFS**.

---

## 1. Naming & Discovery Specifications

* **Kernel Name Branding:** `nv-kernel` (`LOCALVERSION="-nv-kernel"`)
* **Target Device:** Xiaomi Redmi Note 7 (`lavender`)
* **Kernel Base:** Linux 4.4 CAF (Code Aurora Forum)
* **Output Artifact:** `nv-kernel-Lavender-4.4-CAF-SukiSU-SuFS.zip`

### Repositori 4.4 CAF Rekomendasi
1. `paimon-project/android_kernel_xiaomi_lavender` (Branch: `4.4-caf` / `caf`)
2. `Yasir-siddiqui/android_kernel_xiaomi_lavender` (Branch: `caf-4.4`)
3. `qawdf/android_kernel_xiaomi_lavender` (Branch: `4.4`)

---

## 2. Configuration & Naming Injection

Agen AI wajib memasukkan konfigurasi penamaan `nv-kernel`, modul SukiSU, dan SuFS ke dalam `defconfig`:

```bash
cd kernel-source

# 1. Inject KernelSU / SukiSU Source Code
curl -LSs "[https://raw.githubusercontent.com/tiann/KernelSU/main/kernel/setup.sh](https://raw.githubusercontent.com/tiann/KernelSU/main/kernel/setup.sh)" | bash -

# 2. Patch Defconfig (Nama Kernel + SukiSU + SuFS)
DEFCONFIG_FILE=$(find arch/arm64/configs/ -name "*lavender*" -o -name "*perf_defconfig" | head -n 1)

echo "Patching Defconfig: ${DEFCONFIG_FILE}..."
cat <<EOT>> "${DEFCONFIG_FILE}"

# Kernel Branding Name
CONFIG_LOCALVERSION="-nv-kernel"

# SukiSU / KernelSU Flags
CONFIG_KSU=y

# SuFS / OverlayFS Support
CONFIG_OVERLAY_FS=y
CONFIG_SUFS=y
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
