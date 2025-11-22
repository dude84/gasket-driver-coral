# Coral Gasket Driver (modern-kernel fork)

This repository is a fork of Google’s original `google/gasket-driver` made compatible with **modern Linux kernels (6.8+, 6.13+, 6.17.x)** including **Proxmox VE 8/9** and **Debian 12/13**.

It contains minimal patches required for successful DKMS builds on newer kernels and provides instructions on building and installing a stable, persistent DKMS package that does not break during `apt upgrade`.

---

## What’s changed (kernel compatibility fixes)

### 1. Removed deprecated `no_llseek`
The kernel removed `no_llseek` in 6.6+.
This fork replaces it with:

`noop_llseek`

### 2. Fixed `MODULE_IMPORT_NS(DMA_BUF)`
Modern kernels require the argument to be a **string literal**, not a token.

This fork uses:

```
#if LINUX_VERSION_CODE < KERNEL_VERSION(6, 13, 0)
MODULE_IMPORT_NS(DMA_BUF);
#else
MODULE_IMPORT_NS("DMA_BUF");
#endif
```

This ensures compatibility with both older and newer kernels.

---

## Building a DKMS `.deb` package

### 1. Install dependencies

For Debian/Ubuntu:

```
apt update
apt install -y build-essential devscripts dh-dkms dkms linux-headers-$(uname -r)
```

For Proxmox:

```
apt install -y build-essential devscripts dh-dkms dkms pve-headers-$(uname -r)
```

### 2. Clone this repository

```
cd /root
git clone https://github.com/dude84/gasket-driver-coral.git
cd gasket-driver-coral
```

### 3. Build the DKMS package

```
debuild -us -uc -tc -b
```

This will generate a file above the repo directory:

`../gasket-dkms_1.0-<version>_all.deb`

### 4. Install the DKMS package

```
cd ..
dpkg -i gasket-dkms_1.0-*_all.deb
```

Check DKMS status:

```
dkms status
```

Expected:

`gasket/1.0, <kernel-version>, x86_64: installed`

---

## Preventing APT from overwriting your patched driver

Some distros ship their own `gasket-dkms` which can overwrite your patched package.

After installing your patched `.deb`, freeze it:

```
apt-mark hold gasket-dkms
```

Verify:

```
apt-mark showhold
```

To unfreeze later:

```
apt-mark unhold gasket-dkms
```

---

## Proxmox VE 9 / Debian 13 quick install (kernel 6.17.x)

1. Install headers:

```
apt install -y pve-headers-$(uname -r)
```

2. Build and install patched `.deb`.

3. Hold the package:

```
apt-mark hold gasket-dkms
```

4. Reboot and confirm modules loaded:

```
lsmod | grep -E 'gasket|apex'
```

---

## Verifying the TPU works

### 1. PCI detection

```
lspci -nn | grep -E '089a|Apex|TPU'
```

### 2. Driver binding

```
lspci -k -s 01:00.0
```

Expected:

`Kernel driver in use: apex`

### 3. Modules loaded

```
lsmod | grep -E 'gasket|apex'
```

### 4. Device nodes

```
ls -l /dev/apex_*
```

---

## Installing runtime (user space)

```
apt install libedgetpu1-std
```

Test TPU with Python:

```
pip3 install tflite-runtime
```

Example:

```
from tflite_runtime.interpreter import Interpreter, load_delegate
interpreter = Interpreter("model_edgetpu.tflite",
    experimental_delegates=[load_delegate("libedgetpu.so.1")])
print("TPU loaded OK")
```

---

## Notes

- Verified working on Debian 12/13, Proxmox VE 8/9, kernel 6.6–6.17.
- PRs welcome.
