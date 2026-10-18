# LAB-02 — Linux Character Device Driver

## Overview

This lab builds on LAB-01's minimal Embedded Linux system. It implements a loadable Linux character device driver and exposes device data and statistics through `/dev`, `/proc`, and `/sys`.

> **Current scope:** The character driver, procfs, and sysfs features have been built and tested on ARMv7 QEMU. The MTD/JFFS2 and BusyBox auto-load sections of the original LAB-02 handout are **not yet documented as completed**.

## Development Environment

| Component | Configuration |
|---|---|
| Host OS | Ubuntu 24.04.5 LTS (x86_64) |
| Target CPU / architecture | ARM Cortex-A9 / ARMv7-A |
| Emulator | QEMU `vexpress-a9` (QEMU 8.2.2) |
| Target Linux kernel | 6.1.90 |
| Cross compiler | `arm-linux-gnueabihf-gcc` 13.3.0 |
| Root filesystem | BusyBox + initramfs from LAB-01 |
| Build system | Linux Kbuild (external module) |

The host kernel and target kernel are different. Build the module for **ARMv7 / Linux 6.1.90** and load it **inside QEMU**, not on the x86_64 host.

## Repository Structure

```text
labs/lab02/
├── README.md
└── driver/
    ├── Makefile
    └── lab2_driver.c
```

The optional follow-up exercise is maintained separately at:

```text
assignments/driver-exercises/lab21_sysfs_device_name/
```

## Driver Features

- Character device: `/dev/lab2`
- Fixed major/minor: **240:0** (ensure the major number is available)
- Kernel buffer capacity: **1024 bytes**
- `open()`, `read()`, `write()`, and `release()` via `file_operations`
- Safe userspace transfers using `copy_to_user()` and `copy_from_user()`
- Mutex protection for shared buffer state
- procfs entry: `/proc/lab2_info`
- Read-only sysfs attributes: `buffer_len`, `open_count`, and `last_data`

**Note:** `open_count` is cumulative: it counts successful opens of `/dev/lab2` since module insertion, not the number of currently active users. A shell redirection and a subsequent `cat /dev/lab2` each increment the counter.

### Data path

```text
Userspace application
    | open / read / write / close
    v
/dev/lab2 (character device, 240:0)
    |
    v
VFS -> file_operations
       | read  -> copy_to_user()   -> user buffer
       | write -> copy_from_user() -> device_buffer
       v
Kernel buffer (1024 bytes)
    |
    +--> /proc/lab2_info         (statistics)
    +--> /sys/class/lab2_class/lab2/ (attributes)
```

## Prerequisites

LAB-01 artifacts must already exist:

```text
~/embedded_lab1/output/zImage
~/embedded_lab1/output/vexpress-v2p-ca9.dtb
~/embedded_lab1/rootfs/initramfs/
~/embedded_lab1/kernel/linux-6.1.90/
```

The Linux kernel must support modules, module unloading, procfs, and sysfs. The kernel source tree must be configured and prepared for building external modules.

Required host tools: `make`, `arm-linux-gnueabihf-gcc`, `qemu-system-arm`, `cpio`, `gzip`, `modinfo`, and `file`.

## 1. Build the ARM Kernel Module (Ubuntu Host)

From the repository root:

```bash
cd labs/lab02/driver
make clean
make
file lab2_driver.ko
modinfo lab2_driver.ko | grep vermagic
```

Expected architecture and kernel compatibility:

```text
ELF 32-bit LSB relocatable, ARM, EABI5
vermagic: 6.1.90 SMP mod_unload ARMv7 p2v8
```

The included Makefile references `$(HOME)/embedded_lab1/kernel/linux-6.1.90`. Adjust `KDIR` if your LAB-01 kernel source lives elsewhere.

## 2. Deploy to an initramfs (Ubuntu Host)

The following manual commands assume the repository is stored at `~/Github_embedded_system/embedded-linux-systems`. They create a **separate working rootfs** so the LAB-01 original is not modified.

```bash
cd ~/Github_embedded_system/embedded-linux-systems
mkdir -p /tmp/lab02-rootfs /tmp/lab02-output
sudo cp -a ~/embedded_lab1/rootfs/initramfs/. /tmp/lab02-rootfs/
sudo mkdir -p /tmp/lab02-rootfs/lib/modules/6.1.90
sudo cp labs/lab02/driver/lab2_driver.ko /tmp/lab02-rootfs/lib/modules/6.1.90/
```

Special device nodes in `/dev` require elevated permissions when copying or repackaging the rootfs.

```bash
cd /tmp/lab02-rootfs
sudo find . -print0 | sudo cpio --null -o -H newc | gzip -9 > /tmp/lab02-output/initramfs_lab02.cpio.gz
```

Confirm that the module is in the archive:

```bash
gzip -dc /tmp/lab02-output/initramfs_lab02.cpio.gz | cpio -it | grep lab2_driver.ko
```

## 3. Boot QEMU (Ubuntu Host)

```bash
qemu-system-arm \
  -M vexpress-a9 \
  -cpu cortex-a9 \
  -m 512M \
  -smp 2 \
  -nographic \
  -kernel ~/embedded_lab1/output/zImage \
  -dtb ~/embedded_lab1/output/vexpress-v2p-ca9.dtb \
  -initrd /tmp/lab02-output/initramfs_lab02.cpio.gz \
  -append 'console=ttyAMA0,115200 rdinit=/sbin/init'
```

At the BusyBox `askfirst` prompt, press **Enter** to open the shell (`/ #`).

## 4. Load and Test the Driver (Inside QEMU)

Run these commands **inside the ARM QEMU guest**, not on the Ubuntu host:

```sh
uname -r
insmod /lib/modules/6.1.90/lab2_driver.ko
lsmod
dmesg | grep lab2
```

If the device node is absent, create it manually:

```sh
[ -c /dev/lab2 ] || mknod /dev/lab2 c 240 0
ls -l /dev/lab2
```

Test character-device read/write:

```sh
echo 'Hello LAB02' > /dev/lab2
cat /dev/lab2
```

Check procfs and sysfs:

```sh
cat /proc/lab2_info
ls /sys/class/lab2_class/lab2/
cat /sys/class/lab2_class/lab2/buffer_len
cat /sys/class/lab2_class/lab2/open_count
cat /sys/class/lab2_class/lab2/last_data
```

Because `echo` appends `\n`, `echo 'Hello LAB02'` writes 12 bytes. Use `printf` to avoid the newline.

To unload the module after testing:

```sh
rmmod lab2_driver
dmesg | tail -n 15
```

## Test Status

| Item | Status | Evidence |
|---|---|---|
| Build kernel module for ARMv7 | PASS | `make`, `file`, `modinfo` |
| Boot Linux 6.1.90 on QEMU | PASS | `uname -a` in the guest |
| Insert module | PASS | `insmod`, `lsmod` |
| Character-device read/write | PASS | `echo > /dev/lab2`, `cat /dev/lab2` |
| procfs statistics | PASS | `cat /proc/lab2_info` |
| sysfs `device_name` (Lab 21 extension) | PASS | See separate Lab 21 source; read-only behavior verified |
| MTD / JFFS2 | Not yet verified | Planned LAB-02 section |
| BusyBox auto-load of driver | Not yet verified | Module currently inserted manually |

The Lab 21 extension is **not part of this baseline source**. It is stored under `assignments/driver-exercises/lab21_sysfs_device_name/`.

## Common Issues

| Symptom | Check / resolution |
|---|---|
| `insmod` reports invalid module format | Check `uname -r` in QEMU and `modinfo ... | grep vermagic` on the host. |
| `/dev/lab2` is missing | Create `mknod /dev/lab2 c 240 0` in QEMU. |
| Host `cp -a` cannot create files in `/dev` | Copy the initramfs with `sudo cp -a`. |
| `class_create` API error | This lab targets Linux **6.1.90**; APIs differ in later kernels. |
| Major 240 is unavailable | Check `/proc/devices` and select a free major; update the device node accordingly. |
| sysfs attributes are missing | Confirm `/sys` is mounted and inspect `dmesg` for initialization errors. |

## Next Steps

- Continue the independent Lab 22–40 driver extension exercises.
- Complete MTD/JFFS2 and BusyBox module auto-loading in the original LAB-02 handout.
- Add reproducible test logs and keep source, Makefiles, and documentation under version control; do not commit generated `.ko` files.