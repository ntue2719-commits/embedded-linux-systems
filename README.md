# Embedded Linux Systems

A collection of laboratory exercises, assignments, and driver-development practice for the **Embedded Linux Systems** course.

This repository focuses on hands-on Embedded Linux development using the Linux kernel, U-Boot, BusyBox, QEMU, kernel modules, device drivers, and embedded file systems.

## Development Environment

| Component | Configuration |
|---|---|
| Host OS | Ubuntu 24.04.5 LTS (x86_64) |
| Target Architecture | ARMv7-A |
| Target CPU | ARM Cortex-A9 |
| Emulator | QEMU `vexpress-a9` (QEMU 8.2.2 on host) |
| Linux Kernel | **6.1.90** (verified in QEMU) |
| Cross Compiler | `arm-linux-gnueabihf-` (GCC 13.3.0 on host) |
| Bootloader | U-Boot (built in LAB-01) |
| Root Filesystem | BusyBox + initramfs |

> **Version note:** Earlier documentation mentioned Linux 6.16. The verified LAB-01/LAB-02 QEMU target currently runs **Linux 6.1.90**. The host kernel is separate from the ARM target kernel.

## Repository Structure

```text
embedded-linux-systems/
├── common/         # Shared scripts and configurations
├── docs/           # Course-level documentation
├── labs/           # Laboratory exercises
├── assignments/    # Course assignments
└── resources/      # References and supporting materials
```

The separate local workspace currently used for the 20 driver exercises is:

```text
~/embedded_linux_driver_exercises/
├── baseline/          # LAB-02 character-driver baseline
├── common/            # Makefile and local setup utilities
├── docs/              # Exercise notes and results
├── exercises/         # Independent Lab 21–40 directories
├── rootfs/            # Working QEMU initramfs tree
└── output/            # Generated initramfs images
```

**Repository note:** The exercise workspace above is currently local. This README documents its progress; it does not imply that the workspace has already been uploaded to GitHub. The final repository layout may be adjusted when the exercises are published.

## Labs

| Lab | Topic | Status |
|---|---|---|
| LAB-01 | Linux Kernel Configuration and Boot | QEMU boot verified |
| LAB-02 | Device Driver and File System Management | In progress: character driver baseline and Lab 21 verified |
| Lab 21–40 | Character Driver Extension Exercises | Lab 21 PASS; Labs 22–40 pending |

## Assignments

| Assignment | Topic | Status |
|---|---|---|
| Assignment-01 | Linux Device Driver Development | Planned |

## Learning Path

```text
LAB-01: Linux Kernel Configuration & Boot
                  |
                  v
LAB-02: Character Driver, procfs/sysfs,
        MTD/JFFS2, BusyBox init
                  |
                  v
Driver Exercises 21–40:
sysfs -> validation -> file offsets -> poll/wait queues
      -> ring buffer -> ioctl -> concurrency
                  |
                  v
Assignment-01: ioctl and misc device driver
```

## LAB-01 Overview

LAB-01 creates a minimal ARM Linux environment using:

- Linux Kernel **6.1.90**
- ARMv7 Cortex-A9 / QEMU `vexpress-a9`
- U-Boot
- BusyBox with initramfs
- `vexpress-v2p-ca9.dtb` Device Tree

Current boot test uses QEMU to load `zImage`, the Device Tree, and initramfs directly. U-Boot was built during LAB-01, but it is **not used in this direct-QEMU boot command**.

```text
QEMU
  |-- zImage (Linux 6.1.90)
  |-- Device Tree
  `-- initramfs
        |
        v
     BusyBox init
        |
        v
     /etc/inittab -> /etc/init.d/rcS -> shell
```

Verified LAB-01 artifacts:

```text
~/embedded_lab1/output/
├── zImage
├── vexpress-v2p-ca9.dtb
├── initramfs.cpio.gz
└── u-boot
```

## LAB-02 Overview

LAB-02 extends LAB-01 with:

- Loadable Linux kernel modules (`.ko`)
- Character device and `file_operations` (`open`, `read`, `write`, `release`)
- Kernel/userspace data transfer (`copy_to_user`, `copy_from_user`)
- Mutex protection for shared data
- procfs and sysfs
- MTD and JFFS2 (to be completed)
- BusyBox init integration (to be completed)

### Verified Character Driver Baseline

| Property | Current value |
|---|---|
| Module | `lab2_driver.ko` |
| Device node | `/dev/lab2` (created with `mknod` during test) |
| Major / Minor | `240 / 0` |
| Buffer size | `1024` bytes |
| procfs | `/proc/lab2_info` |
| sysfs | `/sys/class/lab2_class/lab2/` |
| Build target | Linux `6.1.90`, ARMv7 |

The module has been cross-compiled and successfully inserted into the QEMU ARM guest. Basic character-device read/write and procfs output have been verified.

### Driver Exercises: Lab 21–40

Each exercise has an independent source copy based on the LAB-02 baseline.

| Labs | Focus | Progress |
|---|---|---|
| 21–28 | sysfs attributes, counters, validation, open limits | **Lab 21 PASS**; 22–28 pending |
| 29–30 | Line-based reads and `O_APPEND` | Pending |
| 31–33 | `poll()`, wait queues, blocking/non-blocking reads | Pending |
| 34–36 | Ring buffer and `ioctl` | Pending |
| 37–40 | Producer/consumer, multiple readers, timeouts, advanced `ioctl` | Pending |

**Lab 21 – `device_name` sysfs attribute: PASS**

Verified in QEMU:

```text
$ uname -r
6.1.90

$ insmod /lib/modules/6.1.90/lab2_driver.ko
$ lsmod
lab2_driver 16384 0 - Live ... (O)

$ cat /sys/class/lab2_class/lab2/device_name
lab2

$ echo "test" > /sys/class/lab2_class/lab2/device_name
Permission denied

$ echo "Hello lab21" > /dev/lab2
$ cat /dev/lab2
Hello lab21
```

The guest also returned driver statistics from `/proc/lab2_info`, including `open_count = 2` after one write and one read of `/dev/lab2`. `open_count` counts successful opens cumulatively, not currently active users.

### Manual Exercise Workflow

The current learning workflow deliberately uses **manual commands**, not automated build/deploy/run scripts:

1. Edit the specific exercise's `lab2_driver.c`.
2. Run `make clean && make` in that exercise directory.
3. Check `file lab2_driver.ko` and `modinfo lab2_driver.ko`.
4. Copy the `.ko` into the working initramfs under `/lib/modules/6.1.90/`.
5. Repackage the initramfs with `cpio` and `gzip`.
6. Boot QEMU and run `insmod`, sysfs/procfs, and read/write tests manually.
7. Record PASS/FAIL and observations before proceeding.

> **Do not insert ARM `.ko` modules into the x86_64 Ubuntu host.** Load them only in the ARM QEMU guest.

Automation scripts and GitHub publication of the Lab 21–40 work are planned **after all 20 exercises have been completed and verified**.

## Assignment-01 Overview

Assignment-01 is planned to extend the device-driver topics from LAB-02:

- `ioctl` interface and custom commands
- Driver statistics
- `miscdevice`
- Atomic operations
- procfs integration
- Driver architecture and analysis

## Basic Git Workflow

When the exercises are ready for publication:

```bash
git status
git add .
git commit -m "docs: update embedded Linux lab progress"
git push origin main
```

Commit convention:

```text
feat:     new feature or implementation
fix:      bug fix
docs:     documentation update
test:     test update
build:    build script or Makefile update
refactor: code restructuring
chore:    repository maintenance
```

## Notes

- Do not commit full Linux source trees, temporary kernel build products, or unnecessary generated files.
- Preserve important kernel and BusyBox configurations.
- Keep a `README.md` in each completed lab/assignment documenting build, run, and test steps.
- Prefer reproducible build instructions over checked-in binaries.
- Keep baseline source unchanged; edit each exercise independently.

## Academic Integrity

This repository is maintained for personal coursework, learning, and portfolio purposes. Do not copy another student's code or report. Cite external references where appropriate and follow university course policies.

## Author

**Nguyen Tri Tue**  
FPT University  
Embedded Linux Systems