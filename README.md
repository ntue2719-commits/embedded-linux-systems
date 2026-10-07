# Embedded Linux Systems

A collection of laboratory exercises and assignments for the **Embedded Linux Systems** course.

This repository focuses on practical Embedded Linux development using Linux Kernel, U-Boot, BusyBox, QEMU, kernel modules, device drivers, and embedded file systems.

## Development Environment

| Component | Configuration |
|---|---|
| Target Architecture | ARMv7-A |
| CPU | ARM Cortex-A9 |
| Emulator | QEMU `vexpress-a9` |
| Linux Kernel | 6.16 |
| Cross Compiler | `arm-linux-gnueabihf-` |
| Bootloader | U-Boot |
| Root Filesystem | BusyBox + initramfs |

## Repository Structure

```text
embedded-linux-systems/
├── common/         # Shared scripts and configurations
├── docs/           # Course-level documentation
├── labs/           # Laboratory exercises
├── assignments/    # Course assignments
└── resources/      # References and supporting materials
```

## Labs

| Lab | Topic | Status |
|---|---|---|
| LAB-01 | Linux Kernel Configuration and Boot | In Progress |
| LAB-02 | Device Driver and File System Management | Planned |

## Assignments

| Assignment | Topic | Status |
|---|---|---|
| Assignment-01 | Linux Device Driver Development | Planned |

## Learning Path

```text
LAB-01
Linux Kernel Configuration & Boot
        │
        ▼
LAB-02
Kernel Module, Character Driver,
procfs/sysfs, MTD and JFFS2
        │
        ▼
Assignment-01
ioctl Interface and Misc Device Driver
```

## LAB-01 Overview

LAB-01 focuses on building and booting a minimal Embedded Linux system using:

- Linux Kernel 6.16
- ARMv7 Cortex-A9
- QEMU `vexpress-a9`
- U-Boot
- BusyBox
- initramfs
- Device Tree

Boot flow:

```text
U-Boot
  │
  ▼
Linux Kernel
  │
  ▼
Device Tree
  │
  ▼
initramfs
  │
  ▼
BusyBox init
  │
  ▼
Shell
```

## LAB-02 Overview

LAB-02 extends LAB-01 and introduces:

- Linux Kernel Modules
- Character Device Drivers
- `file_operations`
- `copy_to_user()`
- `copy_from_user()`
- procfs
- sysfs
- MTD subsystem
- JFFS2
- BusyBox init integration

## Assignment-01 Overview

Assignment-01 extends the device-driver topics from LAB-02.

Main topics:

- `ioctl` interface
- Custom ioctl commands
- Driver statistics
- `miscdevice`
- Atomic operations
- procfs integration
- Driver architecture and analysis

## Basic Git Workflow

Update the local repository:

```bash
git pull origin main
```

After making changes:

```bash
git status
git add .
git commit -m "your commit message"
git push origin main
```

Example:

```bash
git add .
git commit -m "feat: add lab01 Linux 6.16 configuration"
git push origin main
```

## Commit Convention

```text
feat:     new feature or implementation
fix:      bug fix
docs:     documentation update
test:     test update
build:    build script or Makefile update
refactor: code restructuring
chore:    repository maintenance
```

Examples:

```text
feat: add lab02 character device driver
fix: correct QEMU boot arguments
docs: add lab01 boot sequence
test: add driver test script
```

## Notes

- Linux source trees and unnecessary generated build files should not be committed.
- Important configuration files such as `kernel-6.16.config` and `busybox.config` should be preserved.
- Each lab and assignment should include its own `README.md` with build, run, and test instructions.
- Generated binaries should preferably be reproducible from source.

## Academic Integrity

This repository is maintained for personal coursework, learning, and portfolio purposes.

Do not copy another student's source code or report. External references should be cited when appropriate.

Course and university academic-integrity requirements always take priority.

## Author

**Nguyen Tri Tue**  
FPT University  
Embedded Linux Systems