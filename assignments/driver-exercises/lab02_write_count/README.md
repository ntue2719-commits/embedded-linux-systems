# LAB 02 — Write Operation Counter

**Difficulty:** ★★  
**Target:** ARMv7 Cortex-A9, Linux kernel 6.1.90, QEMU `vexpress-a9`  
**Starting point:** Independent copy of the LAB-02 baseline character driver  
**Result:** **PASS — 6/6 functional checks reported in QEMU logs**

> **Naming:** This README documents exercise **Lab 02 of the new Lab 01–20 sequence** (`lab02_write_count`). It is distinct from the course's original **LAB-02 baseline** (`~/embedded_lab2/`).

## 1. Objective and requirements

Extend the existing `lab2_driver.c` to count **successful non-empty calls to `write()`** on `/dev/lab2`. Publish the statistic as a new line in the **existing** `/proc/lab2_info` report. Do not change the device name, major number, baseline sysfs attributes or overall read/write behavior.

| ID | Requirement | Expected behavior |
|---|---|---|
| FR-01 | Global counter | Add `write_count`, initially `0`. |
| FR-02 | Successful writes | Increment **once per successful `lab2_write()` returning a positive number of bytes**. |
| FR-03 | Zero-length writes | `write(fd, ..., 0)` returns `0`, with no counter increment. |
| FR-04 | Failures | Do not increment when `copy_from_user()` fails or the mutex acquisition is interrupted. |
| FR-05 | Concurrency | Update and observe `write_count` under the existing `lab2_mutex`. |
| FR-06 | procfs | Show `Write count   : <value>` in `/proc/lab2_info`. |
| FR-07 | Compatibility | Preserve the original driver interfaces and buffer behavior. |

**Counting contract:** A write longer than 1023 bytes is truncated by this baseline, but if 1023 bytes are copied successfully, the call is considered one successful write and increases `write_count` by one. Multiple `write()` system calls are counted separately. Opening the device without writing is not counted.

## 2. Baseline analysis and design reasoning

The baseline uses:

- `/dev/lab2` (character device, major **240**, minor **0**);
- `device_buffer[1024]` and `buffer_len` (text buffer, latest write replaces previous content);
- `lab2_mutex` to protect shared buffer state and `open_count`;
- `lab2_proc_show()` with `seq_file` to format `/proc/lab2_info`;
- read-only sysfs attributes under `/sys/class/lab2_class/lab2/`.

Original `lab2_write()` flow:

```text
write(fd, user_buf, len)
    -> if len == 0: return 0
    -> to_copy = min(len, 1023)
    -> mutex_lock_interruptible()
    -> copy_from_user(device_buffer, user_buf, to_copy)
         -> failure: unlock, return -EFAULT
    -> buffer_len = to_copy; append NUL terminator
    -> unlock
    -> return to_copy
```

**Design decision:** Put `write_count++` **after** `copy_from_user()` and buffer-length updates, but **before** releasing the mutex. This increments only after the data transfer and state update succeed, without an extra lock or a race between the buffer update and counter update.

The counter is reported inside `lab2_proc_show()` while that function already holds `lab2_mutex`. There is no need to add another `proc_create()` entry or a new sysfs file.

## 3. Implementation — exact code changes

Modify only `~/embedded_linux_driver_exercises/exercises/lab02_write_count/lab2_driver.c`.

**3.1 — Global variable:**

```c
static unsigned int write_count;
```

**3.2 — Successful write path in `lab2_write()`:**

```c
if (len == 0)
    return 0;

/* Existing code obtains lab2_mutex and checks copy_from_user(). */
if (copy_from_user(device_buffer, buf, to_copy)) {
    mutex_unlock(&lab2_mutex);
    return -EFAULT;
}

buffer_len = to_copy;
device_buffer[buffer_len] = '\0';

/* Count successful non-empty writes while holding the mutex. */
write_count++;

mutex_unlock(&lab2_mutex);
```

The excerpt above illustrates the placement; retain the existing `to_copy` calculation and mutex acquisition from the baseline. Do **not** replace the full function with this partial excerpt.

**3.3 — Procfs output (`lab2_proc_show()`):**

```c
seq_printf(m, "Data length   : %zu bytes\n", buffer_len);
seq_printf(m, "Open count    : %u\n", open_count);
seq_printf(m, "Write count   : %u\n", write_count);
```

No changes are needed to `lab2_proc_fops`, `lab2_init()`, or `lab2_exit()`.

## 4. Environment and filesystem paths

| Purpose | Path |
|---|---|
| Lab 02 source | `~/embedded_linux_driver_exercises/exercises/lab02_write_count/` |
| Shared initramfs staging | `~/embedded_linux_driver_exercises/rootfs/initramfs/` |
| Lab 02 initramfs archive | `~/embedded_linux_driver_exercises/output/initramfs_lab02.cpio.gz` |
| Kernel source/build tree | `~/embedded_lab1/kernel/linux-6.1.90/` |
| Kernel image | `~/embedded_lab1/output/zImage` |
| Device tree | `~/embedded_lab1/output/vexpress-v2p-ca9.dtb` |
| Optional test program | `test_zero_write.c` / `test_zero_write` |

Use `arm-linux-gnueabihf-` (ARM hard-float cross toolchain), **not** the Ubuntu host GCC for QEMU target binaries.

## 5. Build the kernel module (Ubuntu host)

```bash
cd ~/embedded_linux_driver_exercises/exercises/lab02_write_count
make clean
make
file lab2_driver.ko
modinfo lab2_driver.ko | grep vermagic
```

Check that `file` reports an **ARM 32-bit relocatable ELF** and `vermagic` targets Linux **6.1.90**. The module loaded and ran successfully in recorded QEMU output; a separate compile transcript was not provided.

### Prepare the module and initramfs

```bash
mkdir -p ~/embedded_linux_driver_exercises/rootfs/initramfs/lib/modules/6.1.90
cp ~/embedded_linux_driver_exercises/exercises/lab02_write_count/lab2_driver.ko \
   ~/embedded_linux_driver_exercises/rootfs/initramfs/lib/modules/6.1.90/
cd ~/embedded_linux_driver_exercises/rootfs/initramfs
find . -print0 | cpio --null -o --format=newc | gzip -9 \
  > ~/embedded_linux_driver_exercises/output/initramfs_lab02.cpio.gz
ls -lh ~/embedded_linux_driver_exercises/output/initramfs_lab02.cpio.gz
```

The staging rootfs is **shared** across exercises. Copying Lab 02's `.ko` replaces the staged copy from Lab 01, but it does **not** alter either exercise's source. Keep lab-specific initramfs archives separate.

## 6. Boot QEMU (Ubuntu host)

```bash
qemu-system-arm \
  -M vexpress-a9 -cpu cortex-a9 -m 512M -smp 2 -nographic \
  -kernel ~/embedded_lab1/output/zImage \
  -dtb ~/embedded_lab1/output/vexpress-v2p-ca9.dtb \
  -initrd ~/embedded_linux_driver_exercises/output/initramfs_lab02.cpio.gz \
  -append "console=ttyAMA0,115200 rdinit=/sbin/init"
```

Once at the BusyBox prompt (`/ #`):

```sh
uname -r
lsmod
# Only if not already loaded:
insmod /lib/modules/6.1.90/lab2_driver.ko
[ -e /dev/lab2 ] || mknod /dev/lab2 c 240 0
cat /proc/lab2_info
```

`lsmod` in the observed session showed `lab2_driver 16384 0`. The kernel message `loading out-of-tree module taints kernel` is expected when inserting a locally compiled out-of-tree module; by itself, it does not indicate a test failure.

## 7. Functional testing — observed QEMU transcript interpretation

### T01 — Initialization

```sh
cat /proc/lab2_info
```

Observed:

```text
Data length   : 0 bytes
Open count    : 0
Write count   : 0
```

**PASS:** The new counter begins at zero and is visible through procfs.

### T02 — First successful write

```sh
echo -n "HELLO" > /dev/lab2
cat /proc/lab2_info
```

Observed:

```text
lab2_driver: received 5 bytes
Data length   : 5 bytes
Open count    : 2
Write count   : 1
Last data     : [HELLO]
```

**PASS:** One successful write increments `write_count` by one. `Open count` is **2**, not 1, because the observed session had already executed `cat /dev/lab2` before the first write. The extra open did not increment `write_count`.

### T03 — Multiple successful writes

The observed shell input contained an **accidental concatenated command**:

```sh
echo -n "FPGA" > /dev/lab2echo -n "LINUX" > /dev/lab2
```

The driver reported **13 bytes** for that operation, so it must **not** be described as two separate intended writes. A later command was:

```sh
echo -n "LINUX" > /dev/lab2
cat /proc/lab2_info
```

Observed after the full sequence:

```text
Open count    : 4
Write count   : 3
Data length   : 5 bytes
Last data     : [LINUX]
```

**PASS:** Three successful nonempty write system calls were recorded in total: `HELLO` (5 bytes), the accidental 13-byte write, and `LINUX` (5 bytes). The buffer retained the latest write.

For a **clean, repeatable test** in a fresh module session, run each line separately:

```sh
echo -n "HELLO" > /dev/lab2
echo -n "FPGA" > /dev/lab2
echo -n "LINUX" > /dev/lab2
cat /proc/lab2_info
```

Expected `Write count   : 3` and `Last data     : [LINUX]`.

### T04 — Read does not increment the write counter

```sh
cat /dev/lab2
cat /proc/lab2_info
```

Observed device output `LINUX`, with `Write count   : 3` unchanged; `Open count` advanced to `5`. **PASS**.

### T05 — Zero-length write does not increment the counter

The shell expression `: > /dev/lab2` only opened and closed the device in the observed log. It did **not** prove that `write(fd, ..., 0)` had been called. A dedicated userspace test was therefore added.

**Test source: `test_zero_write.c`**

```c
#include <stdio.h>
#include <fcntl.h>
#include <unistd.h>

int main(void)
{
    int fd;
    ssize_t ret;

    fd = open("/dev/lab2", O_WRONLY);
    if (fd < 0) {
        perror("open");
        return 1;
    }

    ret = write(fd, "", 0);
    if (ret < 0) {
        perror("write");
        close(fd);
        return 1;
    }

    printf("write(fd, \"\", 0) returned: %zd\n", ret);
    close(fd);
    return 0;
}
```

**Compile on Ubuntu host:**

```bash
cd ~/embedded_linux_driver_exercises/exercises/lab02_write_count
arm-linux-gnueabihf-gcc -static -Wall -Wextra \
  -o test_zero_write test_zero_write.c
file test_zero_write
mkdir -p ~/embedded_linux_driver_exercises/rootfs/initramfs/usr/bin
cp test_zero_write ~/embedded_linux_driver_exercises/rootfs/initramfs/usr/bin/
cd ~/embedded_linux_driver_exercises/rootfs/initramfs
find . -print0 | cpio --null -o --format=newc | gzip -9 \
  > ~/embedded_linux_driver_exercises/output/initramfs_lab02.cpio.gz
```

Reboot QEMU using the command in §6, then:

```sh
# Insert the module only if not auto-loaded.
insmod /lib/modules/6.1.90/lab2_driver.ko
[ -e /dev/lab2 ] || mknod /dev/lab2 c 240 0
cat /proc/lab2_info
/usr/bin/test_zero_write
cat /proc/lab2_info
```

**Actually observed:**

```text
Before:
  Data length   : 0 bytes
  Open count    : 0
  Write count   : 0

lab2_driver: device opened (count=1)
write(fd, "", 0) returned: 0
lab2_driver: device closed

After:
  Data length   : 0 bytes
  Open count    : 1
  Write count   : 0
```

**PASS:** The zero-length syscall returned 0; the driver counted the successful `open()`, but **did not count a successful write**.

### T06 — Module reload resets statistics

```sh
rmmod lab2_driver
insmod /lib/modules/6.1.90/lab2_driver.ko
cat /proc/lab2_info
```

Observed `Data length: 0`, `Open count: 0`, and `Write count: 0`. **PASS**. This is expected module reinitialization, not a separate reset interface.

### Verification matrix

| Test | Evidence | Result |
|---|---|---|
| T01 | Initial `/proc/lab2_info` showed `Write count: 0` | PASS |
| T02 | `HELLO` successful → `Write count: 1` | PASS |
| T03 | Three successful writes → `Write count: 3` | PASS |
| T04 | Device read left counter at 3 | PASS |
| T05 | `write(fd, "", 0)` returned 0 and counter remained 0 | PASS |
| T06 | Unload/reload reset counter to 0 | PASS |

**Overall: PASS — 6/6 recorded checks.**

## 8. Results analysis and limitations

1. **Count syscalls, not bytes:** `write_count` tracks successful nonempty `write()` calls. A 5-byte write and a 13-byte write each add **one**.
2. **`open_count` differs from `write_count`:** Opening `/dev/lab2` for `cat`, `echo`, or a test binary changes `open_count`; a read-only open is not a successful write.
3. **Zero bytes are different from an empty shell command:** `: > /dev/lab2` is not a reliable zero-length write test; direct `write(fd, "", 0)` is.
4. **Procfs is a snapshot:** `lab2_proc_show()` reads statistics while holding the mutex; procfs itself does not invoke `lab2_write()`.
5. **Limitations:** The QEMU logs do **not** demonstrate injected `copy_from_user()` failures, simultaneous multi-process writes, counter overflow, or a >1023-byte write. These are specified or reasoned about from code but **not claimed as tested**.

## 9. Troubleshooting

| Symptom | Likely cause / action |
|---|---|
| `/proc/lab2_info` does not show `Write count` | Recheck `seq_printf()` in `lab2_proc_show()`; rebuild, replace module in rootfs, repack and reboot. |
| `Write count` stays 0 after normal write | Confirm `write_count++` follows successful `copy_from_user()` and remains in the locked region. |
| `Open count` larger than expected | `cat /dev/lab2` also opens the device; inspect complete command history. |
| `/dev/lab2` missing | Create node `mknod /dev/lab2 c 240 0` once after module load. |
| `insmod` says module already loaded | Check `lsmod`; do not insert twice. |
| `test_zero_write: not found` | Verify ARM binary with `file`, install it into initramfs and repack the correct Lab-02 archive. |
| Module version mismatch | Compare `uname -r` and `modinfo ... | grep vermagic`; target is Linux 6.1.90 ARMv7. |

## 10. Git submission policy

After successful QEMU verification, commit only the source code, Makefile, README and **optional** `test_zero_write.c` (valuable as reproducible test source). Do **not** commit `test_zero_write` executable, `.ko`, `.o`, `Module.symvers`, `modules.order`, or `.cpio.gz` archives.

Suggested repository destination:

```text
assignments/driver-exercises/lab02_write_count/
├── lab2_driver.c
├── Makefile
├── README.md
└── test_zero_write.c
```

Use independent conventional commits for implementation and documentation if desired, and verify `git status` before and after push.