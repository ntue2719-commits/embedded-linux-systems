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
| Bootloader | U-Boot (built in LAB-01; U-Boot-mediated boot not verified) |
| Root Filesystem | BusyBox 1.35.0 + initramfs |

> **Version note:** Earlier documentation mentioned Linux 6.16. The verified LAB-01/LAB-02 QEMU target runs **Linux 6.1.90**. The host kernel is separate from the ARM target kernel.

## Repository Structure

```text
embedded-linux-systems/
├── common/                 # Shared scripts/configurations (if populated)
├── docs/                   # Course-level documentation (if populated)
├── labs/
│   ├── lab01/              # ARM Linux boot and configuration
│   └── lab02/              # Character driver baseline
├── assignments/
│   └── driver-exercises/
│       ├── lab21_sysfs_device_name/
│       ├── lab22_sysfs_buffer_capacity/
│       ├── lab23_open_failure_counter/
│       ├── lab24_last_read_size/
│       ├── lab25_ascii_validation/
│       ├── lab26_info_command/
│       ├── lab27_sysfs_is_open/
│       └── lab28_open_limit/
└── resources/              # Reference materials (if populated)
```

> **Repository note:** This tree reflects the intended organization. Some directories or lab files may exist only in the local workspace or may not yet have been pushed. Check `git status`, `git log`, and the GitHub repository for the exact published state.

### Local Driver Exercise Workspace

```text
~/embedded_linux_driver_exercises/
├── baseline/                # LAB-02 character-driver baseline
├── common/                  # Local build/setup utilities
├── docs/                    # Exercise notes and results
├── exercises/               # Independent Lab 21–40 directories
│   ├── lab26_info_command/
│   ├── lab27_sysfs_is_open/
│   └── lab28_open_limit/
├── rootfs/initramfs/        # Working QEMU root filesystem
└── output/                  # Generated initramfs images
```

The development workspace and Git repository are **separate**. Implement and test in `exercises/`; copy verified source, Makefile, README, and relevant test files into `assignments/driver-exercises/` for version control.

## Labs

| Lab | Topic | Status |
|---|---|---|
| LAB-01 | Linux Kernel Configuration and Boot | **QEMU direct boot verified** |
| LAB-02 | Device Driver and File System Management | **Character-driver baseline verified**; other course components remain |
| Lab 21–28 | Character Driver Extension Exercises | **8/8 functional exercise tests PASS** |
| Lab 29–40 | Advanced Character Driver Exercises | Pending |

## Assignments

| Assignment | Topic | Status |
|---|---|---|
| Assignment-01 | Linux Device Driver Development | Planned / framework not yet verified complete |

## Learning Path

```text
LAB-01: Linux Kernel Configuration & Boot
                         |
                         v
LAB-02: Character Driver, procfs/sysfs,
        MTD/JFFS2, BusyBox init
                         |
                         v
Driver Exercises 21–28 [FUNCTIONAL TESTS PASS]
  sysfs -> open counters -> validation -> command processing
        -> open-state reporting -> concurrent-open limits
                         |
                         v
Driver Exercises 29–40 [PENDING]
  line reads -> O_APPEND -> poll/wait queues -> ring buffer
        -> ioctl -> producer/consumer -> concurrency
                         |
                         v
Assignment-01: ioctl and misc device driver [PLANNED]
```

## LAB-01 Overview

LAB-01 creates a minimal ARM Linux environment using:

- Linux Kernel **6.1.90**
- ARMv7 Cortex-A9 / QEMU `vexpress-a9`
- U-Boot source/build
- BusyBox with initramfs
- `vexpress-v2p-ca9.dtb` Device Tree

The verified boot flow loads `zImage`, Device Tree, and initramfs **directly from QEMU**. U-Boot-mediated boot has not been verified.

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

Relevant LAB-01 paths:

```text
~/embedded_lab1/output/zImage
~/embedded_lab1/output/vexpress-v2p-ca9.dtb
~/embedded_lab1/rootfs/initramfs/
~/embedded_lab1/uboot/u-boot-2022.04/
```

## LAB-02 Overview

The verified character-driver baseline includes:

- Loadable Linux kernel module (`lab2_driver.ko`)
- `file_operations`: `open`, `read`, `write`, `release`
- `copy_to_user()` and `copy_from_user()`
- Mutex protection for shared driver data
- `/proc/lab2_info` via procfs
- Device attributes exposed through sysfs

Other LAB-02 coursework, including MTD/JFFS2 and additional BusyBox integration, is not marked complete here.

### Verified Character Driver Baseline

| Property | Configuration |
|---|---|
| Module | `lab2_driver.ko` |
| Device node | `/dev/lab2` (created using `mknod` in QEMU tests) |
| Major / Minor | `240 / 0` |
| Buffer capacity | `1024` bytes |
| procfs | `/proc/lab2_info` |
| sysfs | `/sys/class/lab2_class/lab2/` |
| Build target | ARMv7 / Linux `6.1.90` |

## Driver Exercises — Lab 21–40

Each exercise uses an **independent** copy of the LAB-02 baseline. Features are not automatically cumulative across exercises.

### Progress Overview

| Lab | Feature | Verified result | Status |
|---|---|---|---|
| 21 | Read-only sysfs `device_name` | Reports `lab2`; write rejected | **PASS** |
| 22 | Read-only sysfs `buffer_capacity` | Reports 1024 independently of data length | **PASS** |
| 23 | Exclusive open and rejected-open counter | Additional open rejected with `-EBUSY`; counters checked | **PASS** |
| 24 | Read-only sysfs `last_read_size` | Reports 9 bytes and partial read of 4 bytes; read-only | **PASS** |
| 25 | Printable ASCII validation | Accepted printable input; rejected `0x0A` / `0x01`; preserved prior data | **PASS — recorded cases** |
| 26 | Special `info` command | Generated statistics snapshot; `INFO` and `info\n` treated as normal data | **PASS — recorded cases** |
| 27 | Read-only sysfs `is_open` | Correct open/close state for two held file descriptors; write rejected | **PASS** |
| 28 | Maximum two active opens | Third open rejected; reopen succeeds after one release; 7/7 recorded tests | **PASS** |
| 29 | Line-by-line reading | Not started | Pending |
| 30 | `O_APPEND` behavior | Not started | Pending |
| 31–33 | `poll()` / wait queues / blocking and nonblocking reads | Not started | Pending |
| 34–36 | Circular buffer and `ioctl` functionality | Not started | Pending |
| 37–40 | Producer/consumer, multiple readers, timeouts, advanced `ioctl` | Not started | Pending |

> **Verification scope:** PASS means the reported functional test cases passed on QEMU. It does not imply exhaustive fault injection or multi-process concurrency stress testing. Git push status for every exercise has not been independently verified.

### Lab 21 — Sysfs `device_name`

- Implemented read-only `/sys/class/lab2_class/lab2/device_name`.
- Confirmed it prints `lab2`.
- Confirmed writes are denied and baseline read/write still works.

### Lab 22 — Sysfs `buffer_capacity`

- Added read-only `buffer_capacity` reporting `1024`.
- Verified the capacity remains constant while `buffer_len` changes.

### Lab 23 — Exclusive Open and Failure Counter

- Allowed one active open at a time.
- Rejected another open with `-EBUSY`.
- Verified successful-open, active-open, and failed-open counters.
- Confirmed reopening succeeds after releasing the first descriptor.

### Lab 24 — Sysfs `last_read_size`

- Added read-only `/sys/class/lab2_class/lab2/last_read_size`.
- Verified initial `0`, successful 9-byte read reporting `9`, and partial 4-byte read reporting `4`.
- Confirmed sysfs rejects writes with `Permission denied`.
- EOF/error-path behavior was not exhaustively verified.

### Lab 25 — Printable ASCII Validation

- Added validation for printable ASCII range `0x20`–`0x7E`.
- Accepted `hello, FPGA` (11 bytes).
- Rejected newline `0x0A` from ordinary `echo` and control byte `0x01` from `printf`.
- Confirmed rejected input does not overwrite the previous data (`hello, FPGA`).
- A stack-frame warning was seen in a temporary-buffer build; build succeeded. Heap-based temporary allocation was discussed as a possible improvement, not confirmed deployed.

### Lab 26 — Special `info` Command

- Recognized exact, case-sensitive four-byte command `info`.
- Generated a statistics response in the shared device buffer using formatted output.
- Verified normal write/read (`hello lab26`), command execution, and reading the response.
- Confirmed `INFO` and `info\n` are treated as normal data.
- Verified snapshot behavior: `Open count : 3` was generated during the write even though the subsequent read opened the device again.

### Lab 27 — Sysfs `is_open`

- Added `active_open_count`, updated during `open()` and `release()`.
- Added read-only `/sys/class/lab2_class/lab2/is_open`.
- Confirmed sequence `0 -> 1 -> 1 -> 1 -> 0` while opening FD 3, opening FD 4, closing FD 3, and closing FD 4.
- Confirmed userspace writes to sysfs fail with `Permission denied`.
- **7/7 recorded functional tests passed.**

### Lab 28 — Maximum Two Concurrent Opens

- Enforced a maximum of two active open file instances using a mutex-protected counter.
- Rejected the third open with `-EBUSY` (`Device or resource busy`).
- Confirmed a rejected open does **not** increment `open_count`.
- Confirmed the limit recovers after a descriptor is closed, allowing a subsequent open.
- Procfs final counters: `Open count = 3`, `Active open = 0`, `Max open = 2`.
- **7/7 recorded functional tests passed.**

Example QEMU transcript (Lab 28):

```text
/ # exec 3<> /dev/lab2
lab2_driver: device opened (total=1, active=1)
/ # exec 4<> /dev/lab2
lab2_driver: device opened (total=2, active=2)
/ # exec 5<> /dev/lab2
lab2_driver: device busy, maximum opens reached
-/bin/sh: can't create /dev/lab2: Device or resource busy
/ # exec 3>&-
lab2_driver: device closed (active=1)
/ # exec 5<> /dev/lab2
lab2_driver: device opened (total=3, active=2)
/ # exec 4>&-
lab2_driver: device closed (active=1)
/ # exec 5>&-
lab2_driver: device closed (active=0)
```

## Manual Exercise Workflow

The exercises are developed and tested manually to reinforce understanding of kernel and userspace mechanisms:

1. Edit only the target exercise's `lab2_driver.c` and other required files.
2. Run `make clean && make` in that exercise directory.
3. Check the `.ko` architecture and module metadata.
4. Copy the verified module into the working ARM initramfs.
5. Repack using `cpio` and `gzip`.
6. Boot QEMU and manually test device operations, procfs/sysfs, and kernel logs.
7. Record test evidence, then copy source, Makefile, README, and relevant tests to the Git repository.
8. Commit and push after verification; use separate documentation commits when appropriate.

> **Safety:** Do not insert ARM `.ko` modules into the x86_64 Ubuntu host. Load them in the ARM QEMU guest only.

### Example QEMU Boot (Lab 28)

```bash
qemu-system-arm \
  -M vexpress-a9 -cpu cortex-a9 -m 512M -smp 2 -nographic \
  -kernel ~/embedded_lab1/output/zImage \
  -dtb ~/embedded_lab1/output/vexpress-v2p-ca9.dtb \
  -initrd ~/embedded_linux_driver_exercises/output/initramfs_lab28.cpio.gz \
  -append "console=ttyAMA0,115200 rdinit=/sbin/init"
```

## Assignment-01 Overview

Assignment-01 is planned to extend LAB-02 driver concepts:

- `ioctl` interfaces and custom commands
- Driver statistics and procfs integration
- `miscdevice`
- Atomic operations and synchronization
- Driver design, verification, and analysis

Implementation and acceptance tests for Assignment-01 have **not** been confirmed complete.

## Git Workflow

The exercises use the local workspace for implementation and the Git clone for reviewed source and documentation.

```bash
cd ~/Github_embedded_system/embedded-linux-systems
git status
git add README.md
git commit -m "docs: update README with Lab 21-28 progress"
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

- Do not commit full kernel source trees or unnecessary `.ko`, `.o`, and generated initramfs files.
- Preserve important kernel/BusyBox configurations and reproducible instructions.
- Maintain a `README.md` for each completed exercise with observed test evidence.
- Keep the LAB-02 baseline unchanged; implement exercises independently.
- Do not mark GitHub publication as complete until `git push` and repository state have been checked.

## Academic Integrity

This repository is maintained for personal coursework, learning, and portfolio purposes. Do not copy another student's code or report. Cite external references where appropriate and follow university course policies.

## Author

**Nguyen Tri Tue**  
FPT University  
Embedded Linux Systems